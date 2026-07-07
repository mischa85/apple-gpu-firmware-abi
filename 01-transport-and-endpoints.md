# Coprocessor Transport Layer + Endpoint Framework (Clean-Room)

Scope: the message-passing substrate the GPU firmware ABI rides on — the hardware
mailbox layer, the 64-bit message format, the endpoint table, and the generic system
endpoints (management/handshake, logging, telemetry, entropy). All names below are my
own, derived from behavior; vendor identifiers are omitted.

Throughout, **host** = the application processor (the main CPU running the host driver), and
**coprocessor** = the GPU-side real-time microcontroller that the firmware runs on.

---

## 1. Mailbox message format, hardware registers, send/receive flow

### 1.1 The 64-bit message + endpoint id

Messages crossing the mailbox are **64 bits** (`u64`). The internal type is a 64-bit
value passed by value to every handler. Multiple log/error formats print the message as a
single `0x%016llx`:
- a management "unsupported message" diagnostic that prints the word as one 64-bit value
- a telemetry "invalid message" diagnostic printing the 64-bit word plus a command field
- a crash-dump trace-table header with direction / endpoint / timestamp / message columns,
  the timestamp and message columns being 64-bit.

**On the receive side the endpoint id travels alongside the message, not packed into the
64-bit payload.** The receive pump pulls mailbox entries in bulk; each entry is **16 bytes**:
an 8-byte message word plus a 32-bit endpoint id.

Per-entry decode:
- 64-bit message word at entry `+0x00`, saved to a 16-byte local `{msg, id}`.
- **endpoint id = u32 at entry `+0x08`**.
- `id` is then handed to the table lookup.

| field (16-byte inbound mailbox entry) | offset | size | conf | meaning / notes |
|---|---|---|---|---|
| message word (payload/opcode) | `0x00` | 8 | HIGH | printed as `0x%016llx` |
| endpoint id | `0x08` | 4 | HIGH | used as table index |
| (padding to 16) | `0x0c` | 4 | LOW | entry stride is 16 (`0x10` per iteration) |

The bulk fetch retrieves up to **8 entries per call** (capacity = 8; buffer is 128 bytes =
8×16). The pump loops draining the mailbox and re-fills while count==8. **HIGH.**

### 1.2 Message-word internal layout (as seen by handlers)

The management handler path reads the 32-bit low half of the message and splits it into
**two 16-bit subfields**:

- stores low 32 bits of the message word into the endpoint object.
- **bits [15:0] = field A** (a type/index), bounds-checked `> 0xc` → error.
- **bits [31:16] = field B**, checked `< 0xc` → error.
- Later re-read as two halfwords: bits[15:0] and bits[31:16].

So within a message word the low 32 bits carry two 16-bit fields (here used as a
protocol min/max version pair for the version-negotiate exchange). **HIGH** that the low
dword is two u16 halves; **LOW** on the universal meaning (it is command-specific — see
per-endpoint sub-commands).

The **reply/outgoing** message word is assembled as a packed 64-bit constant
(management ack path):
```
x9 = 0x000c | (0x000c<<16) | (0x0020<<48)        ; = 0x0020_0000_000c_000c
msg = x9 | (status << 32)                          ; status in bits [47:32]
```
So the outbound word here = `bits[15:0]=0x000c`, `bits[31:16]=0x000c`,
`bits[47:32]=status`, `bits[63:48]=0x0020`. The `0x0020` in the top 16 bits is a fixed
tag for this reply class (**LOW**: likely the message-type/endpoint selector encoded in
the high bits of the *wire* word, mirroring how outbound words carry their target — but
not independently confirmed). The `0x000c` twin fields echo the accepted version. Status
lives in bits[47:32]. **HIGH** on the bit positions of this specific reply; **LOW** on
generalization.

> Reconciliation: inbound, the mailbox layer delivers `{msg64, id32}` as a decoded pair.
> Outbound, the id/route appears folded back into the message word's high bits by the
> send path (the fixed `0x0020` high-half tag is the observable trace of that). The
> outbound packer was not fully recovered — **LOW**.

### 1.3 Hardware mailbox registers

The mailbox is the coprocessor's hardware mailbox block. The register set is named in a
diagnostic decoder that dumps mailbox state. My clean-room names:

| register (my name) | role | conf |
|---|---|---|
| `inbox_control_register` | host→coprocessor inbox status/level (ready/overflow/underflow) | HIGH |
| `inbox_doorbell_register` | write to push a message into the host→coprocessor inbox | HIGH |
| `outbox_control_register` | coprocessor→host outbox status/level | HIGH |
| `outbox_doorbell_register` | outbox-side set register | HIGH |
| `core_run_state_register` | core run/idle state | HIGH |
| per-mailbox ctrl (secondary block) | 0x%08x per-index ctrl regs | HIGH |

The numeric byte offsets of these registers are **not recovered** in this pass.
Offsets: **UNKNOWN / LOW**. (Concrete offsets left to a live-register or firmware-side
analyst.)

Control-register semantics that ARE observable (from the decoder's condition strings):
each control register encodes at least **not-ready**, **overflow**, and **underflow**
states, per mailbox index. **HIGH** these three status conditions exist; **LOW** on their
bit positions.

### 1.4 Send flow (host → coprocessor)

- Public send entry: takes (endpoint id, 64-bit message word, optional reply pointer,
  wait flag) on the 64-bit coprocessor object; a sibling non-64-bit variant exists. **HIGH.**
- The message word + target endpoint id are marshalled, the id/route folded into the
  wire word, and the word written to the inbox doorbell register, ringing the doorbell.
  The mailbox-layer wrapper takes an item pointer + item count, matching the bulk 16-byte
  item model. **HIGH** on the API shape; **LOW** on exact register writes.
- A spurious-doorbell guard and a single doorbell index (index 0) → **one doorbell**. **HIGH.**

### 1.5 Receive flow (coprocessor → host)

1. Outbox-not-empty interrupt → the outbox interrupt handler fires. **HIGH.**
2. Handler calls into the receive pump, which invokes the provider's bulk getter (vtable
   slot at object `+0xb0`, method offset `+0x918`) — the bulk mailbox reader that returns
   up to 8 `{message,id}` items. **HIGH.**
3. For each item: extract `id` (entry+8), look up the endpoint object via the id-indexed
   table lookup, then dispatch the 64-bit message to that endpoint. **HIGH.**
4. Unknown id → an "unknown endpoint" debug diagnostic, tail-called from the pump. **HIGH.**
5. After draining, an optional post-drain callback runs if object `+0x21f0` is set.
   **LOW** (purpose unconfirmed).

A per-message host↔coprocessor trace ring is maintained (a crash-dump decoder dumps it
with direction / endpoint / timestamp / message columns). Each logged entry = {timestamp u64,
message u64, direction tx/rx, endpoint}. **HIGH.**

---

## 2. Endpoint table / handle structure

Endpoints are stored in a **flat array indexed directly by numeric endpoint id**, hung
off the coprocessor object. From the lookup routine:

| coprocessor-object field | offset | meaning | conf |
|---|---|---|---|
| endpoint table base | `0x2208` | pointer to array of 8-byte endpoint-object pointers | HIGH |
| endpoint table count | `0x2210` | max valid id (u32); `id > count` → return NULL | HIGH |
| table lock | `0x2228` | acquired around lookup | HIGH |

Lookup: bounds-check `id`, then `entry = base + id*8`, `endpoint = *entry`.
**Stride = 8 bytes; id is the array index. HIGH.**

Registration is via an endpoint-registration routine; there are **16 distinct registration
call sites**, one per built-in/system endpoint, each a thin wrapper. The id argument is not
recoverable from the wrappers. **HIGH** there are ~16 registered system/built-in endpoints;
**LOW** on the exact id constants.

### Endpoint object (partial, from receive-dispatch and handlers)

| endpoint-object field | offset | meaning | conf |
|---|---|---|---|
| owning coprocessor / provider | `0x70` | back-pointer to processor object (status byte `+0x15d` read through it) | HIGH |
| last message word cache | `0xa0` | stores incoming msg for handler | HIGH |
| flags | `0x28` | bit0 gates fast-path vs virtual dispatch | HIGH |
| fast-path handler slot | `0xb0` | optional signed fn ptr | HIGH |
| filter/queue object | `0x98` | passed to filter test | LOW |
| low-32 of last message | `0x104` | u32; split into two u16 (see §1.2) | HIGH |
| aux flags byte | `0x120` | bit0 folded into reply status field | HIGH |

Dispatch: caches msg at `+0xa0`, runs a filter/enable check, then either a fast-path fn-ptr
at `+0xb0` or the endpoint's virtual message handler (signed vtable calls at method offsets
`+0x1e8`/`+0x188`/`+0x1c8`). **HIGH** on structure; per-endpoint handler bodies are not
recovered in this pass.

---

## 3. Endpoint catalog

> **Consumer note (blocking):** the numeric endpoint ids are NOT recovered in this pass.
> Endpoints below are identified by behavior/name only. To index the endpoint table by id,
> the ids must be obtained separately.

Numeric ids are **not** emitted as named constants, so most ids are **LOW/unknown**. Where a
message *sub-command* opcode is recoverable it comes from the handler-registration
installers as registration calls with literal opcodes (see §3.2). Sub-command opcodes are
the **message-type selector** each endpoint's message handler switches on (a byte/short of
the message word). All endpoint names below describe the endpoint by function.

### 3.1 Standard startup handshake — management / control endpoint

**Management / control endpoint** (my name). Direction: bidirectional. This is the
control channel that runs the boot handshake and power negotiation. Sub-handlers below.

Coprocessor status state machine (all HIGH):
`waiting-for-version → version-ok → running`, plus `unsupported-remote`, `powering-down`.
Transitions are driven by the handshake messages below.

Handshake / control messages (each handler takes the 64-bit message word):

| sub-handler (my name) | dir | purpose / encoding | conf |
|---|---|---|---|
| **version-negotiate** | coprocessor→host, host replies | coprocessor announces protocol version as a `[min,max]` pair in msg bits[15:0] and [31:16]; host checks against its own version and replies with accepted version (`0x000c`) + status. Version-mismatch → an incompatible-protocol-version diagnostic. Advances status waiting-for-version→version-ok. | HIGH |
| **endpoint-roll-call / endpoint-map** | coprocessor→host, host replies | coprocessor publishes which endpoints it hosts; host registers/enables its side. Requires status==version-ok. This is the endpoint-advertisement step; host then brings up per-endpoint objects. | HIGH |
| **ping / watchdog** | host→coprocessor ping, coprocessor→host ack | liveness ping carrying a sequence number + timestamp; mismatch → an invalid-ping-sequence diagnostic. Watchdog escalates via ping-retry-exhausted and no-response-timeout diagnostics. | HIGH |
| **power-ack / state change** | host→coprocessor request, coprocessor→host ack | power-state negotiation; unexpected value → an unsupported-power-state diagnostic. Requires powering-down status for the down path. Related: a power-notification-timeout gate. | HIGH |

Unhandled type on this endpoint → an "unsupported management message" diagnostic and an
"invalid management message" diagnostic. **HIGH.**

**Overall startup sequence (HIGH, from status machine + handler set):**
1. Coprocessor boots (firmware loaded by early boot; core start via the mailbox layer).
2. Coprocessor sends **version-negotiate** with its protocol `[min,max]`; host validates
   version, replies → status version-ok.
3. Coprocessor sends **endpoint-roll-call** (endpoint map); host registers each
   advertised endpoint into the table (§2) and enables its handlers → status running.
4. Steady state: periodic **ping/ping-ack** watchdog; **power-ack** on power transitions.

The management endpoint also owns a startup timeout, reported by a "not started in time"
diagnostic and a "failed to start" diagnostic. **HIGH.**

### 3.2 Logging / telemetry system endpoints

Sub-command opcodes below are literal immediates passed to per-endpoint handler
registration sites (a log-type variant registers by numeric log-type index). These opcodes
are the **message-type selector** of the inbound word for that endpoint. Confidence on
*opcode value* is HIGH (literal in code); on the *label* mapping it is as noted.

**Structured firmware-log endpoint** (my name). Direction: coprocessor→host (firmware
log stream) + host→coprocessor control. Sub-handlers (each takes the message word):
| sub-cmd (my name) | conf |
|---|---|
| new coalescer/buffer request | HIGH |
| source-info register | HIGH |
| source-info UUID chunk — carries a `shift` offset + 4 bytes of UUID per msg | HIGH |
| source-info role chunk — `curr_size`/`shift` into role buffer | HIGH |
| flush request | HIGH |
Invalid opcode → an invalid-command diagnostic.
Uses shared-memory "coalescer" ring buffers indexed by `id` (`coalescers[id]`). **HIGH.**
Per-source metadata struct with `uuid` and `role` fields. **HIGH** it exists; field
offsets LOW.

**Text log endpoint** (my name). Direction: coprocessor→host text log + host→coprocessor
control.
| sub-cmd | opcode | conf |
|---|---|---|
| request log buffer (variant a) | `0x9d` | HIGH (opcode literal) |
| request log buffer (variant b) | `0x94` | HIGH |
| request log buffer (variant c) | `0x8c` | HIGH |
| dimensions/config | — | HIGH |
| host-ready | `0x5e` | HIGH |
A max-entry-size config accessor exists. **HIGH** on opcodes; the three request-buffer
opcodes likely distinguish size classes (LOW).

**Host-trace-bridge endpoint** (my name). Direction: bidirectional (host driver trace
bridge). Installed sub-commands (opcode → role, all HIGH on opcode literal):
| opcode | role (my name) | conf |
|---|---|---|
| `0x5e` | send host-ready | HIGH |
| `0x6c` | send enabled-state | HIGH |
| `0x7d` | send flush-request | HIGH |
| `0xa8` | typefilter-changed notify | HIGH |
| `0x1b1` | flush | HIGH |
| `0x1f3` (499) | here-is-my-buffer | HIGH |
| `0x207` | here-is-my-typefilter | HIGH |
Also a buffer-request handler. Host-callback reasons: enabled / disabled / sync-flush /
typefilter-changed. **HIGH.** Note the two-byte opcodes (`0x1b1`, `0x1f3`, `0x207`) vs one-byte
(`0x5e`…`0xa8`) confirm the message-type selector is at least a **16-bit** field, consistent
with §1.2. **HIGH.**

**Binary-log demux / router** (my name; a sub-router shared by the structured-log /
text-log / cpu-trace endpoints). Selects a builtin handler by a **numeric log-type
selector value** (an internal builtin-handler table). Builtin selector values map to:
text-log, structured-firmware-log, cpu-trace. The per-entry dispatch takes a **u8 type
selector**. The installer values `0x2f`/`0x3e`/`0x3a`/`0x36`/`0x32` seen in the host-trace-bridge
installer are these builtin-entry registrations. **HIGH** on the mechanism; opcode→type
mapping LOW.

**CPU-trace-stream endpoint** (my name). Direction: bidirectional CPU-trace stream.
Sub-behaviors: set-endpoint-active gate, power-state callback gate, host-trace callback,
release-trace-chunk. Session config has a "host-trace-managed" flag and a trace
state enum = {unconfigured, configured, started, stopped}. **HIGH** (state enum +
callback), message opcodes LOW.

**IO-reporting / telemetry endpoint** (my name). Direction: bidirectional (perf/power
counters). Installed reporter-add opcodes (HIGH on literal): **`0x1b4`, `0x1b2`, `0x1ac`,
`0x1aa`** → the reporter-add handler. Also patchbay write-back opcodes **`0x26c`, `0x274`,
`0x273`, `0x271`, `0x26e`** → the patchbay write-back handler. Reporters keyed by id; reporter
type validated by an invalid-reporter-type diagnostic. A host-initiated update gate pushes
host→coprocessor updates. **HIGH** on opcodes; label mapping LOW.

**Crash-dump endpoint** (my name). Direction: coprocessor→host (coredump/panic delivery).
Handler-install opcodes (HIGH literals): **`0xbf`, `0xc5`, `0xac`, `0xb1`, `0xe2`, `0xde`**.
NMI-driven; a force-crash-dump entry, a check-for-crash gate that reads a
**`crashdump_header`** (my name), a local no-alloc copy buffer, then a rich coredump
decoder chain (register frames, task list, versions). The force-crash reason enum includes
fatal / non-fatal / force. **HIGH** on opcodes + coredump role; exact opcode→message
mapping LOW.

**Entropy-request endpoint** (my name). Direction: coprocessor→host request. Single
handler — firmware requests random bytes from the host RNG; host replies. **HIGH** (single
handler), opcode LOW.

**Analytics-telemetry endpoint** (my name). Direction: coprocessor→host telemetry. The
message handler extracts `cmd=0x%x` from the `0x%016llx` message → confirms low bits =
command. An event handler taking `unsigned int` consumes an analytics-event record from a
shared buffer with a `data_size` field, bounds-checked via a 3-operand add-overflow guard
(`offset + sizeof(event) + data_size`). Keys live in the vendor's reverse-DNS namespace.
**HIGH.**

### 3.3 Endpoint catalog summary

| endpoint (my name) | numeric id | direction | purpose | id conf |
|---|---|---|---|---|
| management / control | unknown (convention: low fixed id) | bidir | version-negotiate / roll-call / ping / power handshake | LOW |
| structured firmware-log | unknown | coproc→host + ctrl | structured firmware log stream + source metadata | LOW |
| text log | unknown | coproc→host + ctrl | plain text log | LOW |
| host-trace-bridge | unknown | bidir | host driver trace bridge | LOW |
| cpu-trace-stream | unknown | bidir | CPU trace chunks | LOW |
| io-reporting / telemetry | unknown | bidir | perf/power counters | LOW |
| crash-dump | unknown | coproc→host | coredump / panic delivery | LOW |
| entropy-request | unknown | coproc→host | RNG requests | LOW |
| analytics-telemetry | unknown | coproc→host | analytics events | LOW |
| `gpu_app_endpoint_1` | see §4 | bidir | GPU firmware ABI | LOW |
| `gpu_app_endpoint_2` | see §4 | bidir | GPU firmware ABI | LOW |
| secure endpoint proxy | numeric-indexed | bidir | proxies endpoints across a secure boundary (`%u`-indexed) | LOW |

The binary-log demux/router (§3.2) is a shared sub-handler selector, not a transport endpoint, and is excluded from this count.

~16 registration sites = ~10 named system endpoints + 2 GPU app endpoints + the secure proxy, leaving ~3 registration slots undocumented (reserved / management-internal) — a named gap. **LOW.**

---

## 4. Where the GPU app-specific endpoints plug in

Beyond the system endpoints, the GPU driver registers **application endpoints** on the
same transport:
- Two named app endpoints, **`gpu_app_endpoint_1`** and **`gpu_app_endpoint_2`**, inside
  the GPU coprocessor client driver. **HIGH** they exist as two distinct app endpoints.
- The GPU driver's **dedicated message worker thread** services these — the GPU endpoints
  run their message dispatch off their own thread loop, separate from the system endpoints.
  **HIGH.**
- The endpoints are created **per firmware-role** (`role`; a firmware-role enum in the
  sibling GPU driver). Command submission + scheduler work-queue gating is tied to them
  ("Scheduler work queues disabled but command submission is still enabled"). **HIGH** on
  association; payloads out of scope.

Registration path: these plug into the **same** endpoint-registration routine table (§2) —
the GPU driver is a client of the transport, so `gpu_app_endpoint_1`/`gpu_app_endpoint_2`
occupy entries in the flat endpoint array and receive dispatched 64-bit messages exactly
like the system endpoints. **HIGH** (architecture); the **numeric ids are not recoverable**
→ **LOW/unknown ids**.

Also present: a **secure endpoint proxy** variant of the whole stack (with a naming format
`%s.<endpoint>.%u`) that proxies endpoints across a secure boundary, and a message-based
power-state service. The `%s.<endpoint>.%u` naming implies endpoints are addressed by
**a numeric index `%u`** appended to a base name — consistent with the id-indexed table.
**HIGH** the numeric-index addressing model; payloads out of scope.

Payload semantics of the two GPU app endpoints (the actual GPU init-data, channel rings,
command submission, power/scheduler messages) are deliberately left to the GPU-ABI
analysts.

---

## Summary of gaps (for follow-up)

- Numeric endpoint ids: **unknown** (no id constants recovered). Needs live tracing or the
  outbound message packer.
- Mailbox register **byte offsets**: unknown. Register roles and semantics (control status
  bits: not-ready/overflow/underflow) known.
- Exact bit layout of the message-type selector beyond "low dword = two u16 halves, type
  selector is ≥16-bit": partial. Per-endpoint opcode *values* are HIGH; their internal
  bit placement within the word is LOW.
- The `0x0020` high-half tag on outbound management replies is unexplained (candidate:
  endpoint/route id folded into wire word) — LOW.
