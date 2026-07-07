# Apple GPU Firmware ABI — Interrupt Delivery, Acknowledgment, and GPU Power / Idle / Wake

Scope: the two directions the earlier documents left open.
1. **How a firmware→host notification actually arrives** — the outbox interrupt, the
   receive pump it drives, the acknowledgment/clear sequence, and the exact call path
   from that interrupt down to the event-ring drain documented in `04-*`.
2. **GPU power / idle / sleep / wake** — the power-state message protocol on the control
   endpoint, the idle-off (sleep) quiesce, and the **wake** handshake that pulls the
   coprocessor out of its "waiting for an interrupt" idle.

Target: t8140 / G17P. All names below are my own, chosen from observed behavior.
Confidence tags: **HIGH** (unambiguous / cross-consumer), **LOW** (inferred), **GAP**
(living in the firmware image / guarded region and not recoverable from the host
driver).

---

## 0. One-paragraph model

A coprocessor→host **outbox-not-empty interrupt** is delivered to a host work-loop as the
action of an interrupt event source. That action is the **mailbox receive pump**, a
*virtual* method installed in the transport-receiver object's vtable. The pump repeatedly
calls the transport provider's **bulk outbox getter** (provider at receiver `+0xb0`, vtable
slot `+0x918`) to drain up to 8 inbound entries at a time — each entry 16 bytes `{u64
message, u32 endpoint_id}` — and for each entry does an **endpoint-table lookup** and hands
it to the **endpoint dispatch**. For the two GPU application endpoints the dispatch reaches a
per-endpoint **message-received callback** which virtual-calls a handler on the GPU master
object (slot `+0x1e8`); via a gated work-loop hop that lands on the **event-ring drain**
(documented in `04-*`). Power is negotiated separately on the **control endpoint** with
64-bit messages whose high 16-bit field selects a message class; when idle the coprocessor
firmware descends a two-level low-power ladder and **parks in a wait-for-interrupt idle**,
and the host wakes it by **ringing the inbox doorbell** (the "kick").

---

## 1. Interrupt delivery (top-half) and fan-out

### 1.1 The receive pump is a vtable-installed interrupt action — HIGH

The receive pump has **no direct-call references**; its only inbound references are
**DATA / vtable-slot** references. Each is a slot inside a C++ vtable; all three share the
pump and an identical sibling method, differing only in a class-discriminator boolean getter
(returns 0 vs returns 1) — i.e. **one base transport-receiver class plus two variants**. The
pump is therefore installed as the receiver's **interrupt action / outbox handler override**
and is invoked purely through a signed vtable pointer by the interrupt event source.
Registration is **data-driven** (vtable + runtime event-source install), which is why no
static call edge exists. **HIGH.**

### 1.2 Transport-receiver object (partial) — HIGH

| offset | size | name | type | conf | meaning / notes |
|-----|------|------|------|------|-----------------|
| `0x00`   | 8 | vtable                    | ptr | HIGH | contains the pump slot |
| `0xb0`   | 8 | transport_provider        | ptr | HIGH | bulk-getter target: vtable slot `+0x918` of the object at receiver `+0xb0` |
| `0xda`   | 1 | post_dispatch_flag        | u8  | HIGH | set to 1 when `+0x2249` bit0 set |
| `0x21f0` | 8 | tail_action_obj           | ptr | HIGH | if nonzero, pump calls a tail action once after draining |
| `0x2208` | 8 | endpoint_table_base       | ptr | HIGH | array of endpoint pointers, stride 8, indexed by endpoint_id |
| `0x2210` | 4 | endpoint_count            | u32 | HIGH | bounds check in the lookup |
| `0x2228` | 8 | endpoint_table_lock       | ptr | HIGH | held around table access |
| `0x2249` | 1 | dispatch_postproc_enable  | u8 (bit0) | HIGH | gates the `+0xda` post-step |

(The endpoint-table triple `+0x2208/+0x2210/+0x2228` is the same structure documented in
`01-*` §2; repeated here only as the interrupt path's fan-out point.)

### 1.3 Bulk outbox getter (the drain + status consume) — HIGH role / GAP offsets

The pump calls vtable slot `+0x918` of the provider with `(&buf128, &count)`, `count` preset to 8,
`buf128` a 128-byte / 8-entry buffer (each entry 16 bytes `{u64 message, u32 endpoint_id}`),
looping while a full batch of 8 is returned. This slot is the transport's **bulk outbox
getter**: reading the outbox FIFO here is where the not-empty / overflow / underflow status
is consumed and the not-empty condition is re-armed/cleared. The concrete implementation
lives in the transport/provider layer and is **not recovered in this pass** (see §2).
Register offsets = **GAP**; the *slot* (`provider +0x918`) and the *role* are **HIGH**.

### 1.4 Endpoint-table lookup and dispatch — HIGH

- Endpoint-table lookup `(receiver, endpoint_id)`: take lock at receiver `+0x2228`, bounds-check
  endpoint_id less than receiver `+0x2210`, index the array at receiver `+0x2208` with stride 8, return the
  endpoint pointer (or 0). Straight id→object resolver. **HIGH.**
- The dispatch `(endpoint, &msg)`: caches the message word at
  endpoint `+0xa0`; checks an enable bit at endpoint `+0x28` and a gate byte at `+0x15d` of the object at endpoint `+0x70`;
  then either a **fast path** raw fn-ptr at endpoint `+0xb0`
  called as the fast-path function with the object at endpoint `+0x50` and the message, or a **slow path** through the endpoint vtable
  (`+0x1f0` deliver, `+0x188` get worker/target, `+0x1c8` post-to-worker). **HIGH.**
- The same dispatch has two non-interrupt callers that fabricate a message word and inject
  it synchronously (e.g. a self-posted `0x0600000000000000` event) — proof the dispatch is
  reused for host-originated events, not only interrupts. **HIGH (context).**

### 1.5 Hardware IRQ source — GAP

The concrete IRQ line / event-source binding lives in the transport layer and is not
resolved here. From the recoverable role set it is an **outbox-not-empty class interrupt**
delivered to the GPU work-loop (consistent with the event-source model, §1.1). The exact
interrupt index / device-tree binding = **GAP**.

---

## 2. Interrupt status / ACK / CLEAR

### 2.1 Register roles (HIGH) — offsets (GAP)

The transport exposes a mailbox control/status register pair per direction plus a
doorbell/set register and a run/idle-status register. The following **roles** are
unambiguous from the mailbox-state decoder's condition strings and the register-label
strings:

| register (my name) | role | conf |
|---|---|---|
| `inbox_control_register`  | host→coprocessor inbox level/status | HIGH |
| `outbox_control_register` | coprocessor→host outbox level/status | HIGH |
| `inbox_doorbell_set_register` | write to push/ring the host→coprocessor inbox (the "kick") | HIGH |
| `outbox_set_register` | coprocessor-side set register | HIGH |
| `run_idle_status_register` | core run/idle indicator | HIGH |

Each control register encodes at least **not-ready**, **overflow**, and **underflow**
per-direction status (distinct outbox and inbox status strings exist). **HIGH** these three
status conditions exist per direction; their bit positions and the register byte offsets are
**GAP** (§2.3).

### 2.2 Two distinct acknowledgments — HIGH

There are **two** acknowledgments in the RX path, at different layers:

1. **Transport/outbox ACK** — performed *inside* the bulk outbox getter
   (`provider +0x918`, §1.3): reading/advancing the outbox is what clears the not-empty
   condition and re-arms the interrupt. Body = GAP; slot + role = HIGH.
2. **Event-ring credit-return ACK** — after consuming each event-ring entry, the peek/
   validator issues a data-memory-barrier and then **writes the new consumer index back
   through the shared index pointer** (ring-internals `+0x08`), returning the slot/credit to
   the firmware. This is the firmware→host ring's own flow-control ACK, distinct from the
   hardware outbox ACK above. **HIGH.** (See §3.5 and `04-*` §1.1.)

### 2.3 Why the offsets are a GAP

No recoverable host-driver code resolves the transport status registers directly. The
register-touching routines are effectively absent here — matching the `01-*` finding. The
numeric offsets for inbox/outbox control, doorbell-set, and run/idle-status are therefore a
firm **GAP**; only roles and reaching-slots are HIGH.

---

## 3. RX → event-ring-drain call path

This section closes the join between `01-*` (transport) and `04-*` (event ring): it shows
how a received message on a GPU application endpoint drives the drain.

### 3.1 GPU application-endpoint callbacks — HIGH

The two app-endpoint callbacks (near identical, differing only in a debug/label constant —
matching the two app endpoints from `02-*` §1.3):
- read a target object at receiver `+0x148`;
- decode the incoming message (using a type token) and extract a field at the decoded message `+0x88`;
- **virtual-call vtable slot `+0x1e8` of the target** with that field — the entry into GPU-master
  firmware-event handling. **HIGH.**

### 3.2 Work-loop hop and the drain as a vtable method — HIGH / GAP

The dispatch slow path (`+0x188` get worker/target, `+0x1c8` post) and the app-callback's
`+0x1e8` virtual marshal the work from interrupt/work-loop context onto the GPU's own
**gated work-loop** (a gated work-loop for the GPU endpoints is confirmed by the diagnostic
strings), rather than running the drain inline in the outbox handler. **HIGH (a gated hop
exists).**

The **event-ring drain** is itself a **vtable method** — its address occupies slots in the
GPU-master vtable region and it has no direct-call references, so it is reached through that
master vtable off the `+0x1e8` / work-loop chain. **HIGH** that it is vtable-invoked; the
exact slot index → exact caller is not numerically pinned (**GAP**).

### 3.3 Drain shape (corroborates `04-*`) — HIGH

The drain calls a per-ring drainer with `ring_sel = 0` then `1` — **two event rings are
drained per invocation**. Each ring's control block sits at `self + ring_sel*0x240 + 0x800`:

| offset | size | name | type | conf | meaning / notes |
|-----|------|------|------|------|-----------------|
| `0x00` | 8 | ring_obj                 | ptr | HIGH | argument to peek/validator |
| `0x08` | 8 | consumer_idx_shared_ptr  | ptr | HIGH | producer/consumer snapshots read from here |
| `0x10` | 8 | backing_liveness         | ptr | HIGH | 0 ⇒ fatal |
| `0x18` | 4 | producer_snapshot        | u32 | HIGH | the value at `+0x08` |
| `0x1c` | 4 | consumer_snapshot        | u32 | HIGH | the value at `+0x08` + `0x20` |
| `0x28` | 8 | capacity                 | u64 | HIGH | bounds/overflow check |

The loop consumes while producer != consumer via the peek/validator (§3.5), switching over
event types 0..`0xe` (the enum documented in `04-*` §1.3). On drain-complete with work seen,
it re-arms downstream objects (the objects at receiver word `0x53`, `+0x618`/`+0x628`, via vtable slot `+0x1f0`; and the object at receiver `+0x5b0`
via vtable slot `+0x498`) and re-arms a 1e9-ns watchdog. **HIGH.**

### 3.4 Ring peek / validator + credit ACK — HIGH

Ring internals (ringblk, with a `0x48`-byte output buffer):

| offset | size | name | type | conf | meaning / notes |
|-----|------|------|------|------|-----------------|
| `0x08` | 8 | consumer_idx_shared_ptr | ptr | HIGH | consumer idx written back here after consume (the ring ACK) |
| `0x10` | 8 | entries_base            | ptr | HIGH | entry = base + idx*`0x48` |
| `0x18` | 4 | consumer_idx            | u32 | HIGH | advanced mod capacity |
| `0x1c` | 4 | producer_idx            | u32 | HIGH | compared to consumer to detect empty |
| `0x20` | 8 | valid_type_bitmap       | u64 | HIGH | `(bitmap >> (entry_type & 0x3f)) & 1` must be set else fatal |
| `0x28` | 8 | capacity                | u64 | HIGH | modulo for index advance |

Entry stride **`0x48`** (18×u32), word0 = event type. After copying an entry it advances the
consumer index modulo capacity, issues a data-memory-barrier, and writes the new consumer
index back through `+0x08` (the credit return / ring ACK of §2.2). **HIGH.** (This matches
`04-*` §1.1 field-for-field; the two documents were derived independently.)

---

## 4. Power-state message protocol (control endpoint)

The control endpoint (documented in `01-*` §3.1) carries power negotiation. Each message is
one 64-bit word; the **high 16-bit field (bits[63:48]) is a message-class tag**, reconciling
the three observed classes:

| class value (bits[63:48]) | direction | meaning | conf |
|---|---|---|---|
| `0x0020` | host↔coprocessor | version / hello reply | HIGH |
| `0x0080` | coprocessor→host, host replies | endpoint roll-call | HIGH |
| `0x00a0` | host→coprocessor | **power-state notification** | HIGH |

(The `0x0020` version reply `0x0020_0000_000c_000c` is the same high-field tag noted in
`01-*` §1.2.)

### 4.1 Host→coprocessor power notification (class `0x00a0`) — HIGH

Sender `(this, power_state)`:
```
word = (power_state & 0xffffffff) | (0x00a0 << 48)
```
- **Deferral**: if the coprocessor run-state (the processor object `+0x158`, §5) is `< 0x06` (not yet
  running), the message is **not sent**; instead a pending request is latched at
  the device object `+0x124` (`= 2` if `power_state == 0`, else `1`). **HIGH.**
- Otherwise it is sent (endpoint vtable slot `+0x1e8`, sendMessage with the device object, the word, 0, 1) and on
  success the device object `+0x124` is cleared to 0. **HIGH.**
- `power_state`: nonzero = power-up / keep-up request, 0 = power-down / allow-down request
  (mirrors the 1/2 deferred encoding). **LOW** on the exact numeric meaning of each value.

### 4.2 Deferred-request flush at roll-call completion — HIGH

The roll-call handler (class `0x0080`) runs only when the processor object `+0x158` equals `0x05`, builds a reply
`word = (msg & 0x8003f00000000) | result32 | (0x0080<<48)`, and sends it. **When roll-call
message bit51 is set and the send succeeds** it transitions the run-state to running (sets
run-state `0x06`) and then **flushes any deferred power request**: if the device object `+0x124` is nonzero it
re-invokes the power sender with the device object `+0x124` equal to 1. This is the exact point where a power
request that arrived before the coprocessor was up is finally delivered. **HIGH.**

### 4.3 Coprocessor→host power-ack — HIGH role / GAP fields

A power-ack handler exists (identified by behavior): the power-down ack is
guarded by run-state == powering-down, and an unsupported requested state logs an
"unsupported power state" diagnostic. The handler **code location is not positively pinned**
(found by string only), so the ack word's bitfield layout is a **GAP**; the role is HIGH.

---

## 5. Coprocessor run-state machine (the processor object `+0x158`)

The host mirrors the coprocessor status in the processor object `+0x158`; a setter `(processor, N)` also
publishes the value to a telemetry reporter. Observed numeric values (from the setter and the
comparisons in the senders):

| value | meaning | conf | notes |
|-------|---------|------|-------|
| `0x04`   | prior / init state          | LOW  | guard `==0x04` in the control dispatcher |
| `0x05`   | version-ok / roll-call pending | HIGH | `==0x05` gate in the roll-call handler; set by the dispatcher |
| `0x06`   | **running / powered**        | HIGH | set after roll-call bit51; power sends require `>= 0x06` |
| `0x8000` | unsupported-remote / bad version | HIGH | the `…ffff0bad` reject path |
| `0x8001` | terminal / gone              | LOW  | early-return guard in the roll-call handler |

This refines the named status machine in `01-*` §3.1 (waiting-for-version → version-ok →
running, + unsupported-remote, + powering-down) with the numeric encodings.

---

## 6. Sleep / idle-off path (host quiesces the GPU) — HIGH flow / GAP offsets

Idle-off is driven by delay tunables (parsed from device configuration; defaults
corroborated in `02-*` §2.2):

| field | default | role | conf |
|-------|---------|------|------|
| device object `+0x217c` | 2    | GPU idle-off delay (ms) | HIGH |
| device object `+0x2188` | `0x28` | power-sequencer idle-off delay (ms) | HIGH |
| device object `+0x2194` | 5    | firmware early-wake timeout (ms) | HIGH |

A probabilistic "smart idle-off" predictor (additional tunables) decides when to allow
power-off. **LOW** detail.

The driver's power-state change entry (system sleep/wake interest hook → a gated
power-change worker) sends the class-`0x00a0` notification (§4.1) and then runs the
**watchdog quiesce/wait loop**: while the device object `+0x100` equals 2 it sleeps for the processor object `+0x128` ns,
polls the rings drained (mailbox vtable slot `+0x888` then `+0x210`), and bounds the wait with a
"no response to power notification in N s" timeout. **HIGH.**

Below the message layer, a GPU **power-gate / sleep-control register unit** exposes these
roles (each reached through a vtable accessor; the concrete MMIO offsets are **GAP**, the
roles are HIGH):

| role (my name) | conf |
|----------------|------|
| write `sleep_control_register` (run/sleep control) | HIGH |
| read `sleep_control_register` | HIGH |
| read `power_irq_status_register` (wake/power interrupt latch) | HIGH |
| write `power_irq_clear_register` (ack the wake/power IRQ) | HIGH |
| reset per-cluster sleep-state bits | HIGH |
| wake-confirmation predicate ("did it wake?") | HIGH |
| predicate: powered-off, waiting on host | HIGH |
| predicate: powered-off, **waiting for a doorbell kick** | HIGH |

---

## 7. Wake path — the key deliverable

### 7.1 Firmware idle end-state — HIGH

When idle the coprocessor firmware descends a two-level low-power ladder (a shallow "nap"
and a deep "sleep", with direct nap→sleep transitions) and **parks in a wait-for-interrupt
idle** (a retention variant preserves SRAM/state for the shallow case). While in the deepest
off state it reports the "powered-off, waiting for a doorbell kick" predicate. This is
exactly the "asleep, waiting for an interrupt" condition an emulator/hypervisor observes.
**HIGH.**

### 7.2 How the host wakes it (the mechanism to reproduce) — HIGH

**Primary wake = ring the inbox doorbell ("kick").** The kick routine:
- Precheck: the device object (receiver `+0x298`) `+0x111ea` must not equal 2 (else fatal-state guard),
  and vtable slot `+0xae8` of the receiver must report readiness bit0 set (else "not ready" fatal).
- Build the doorbell word:
```
word = ((selector == 0) ? 0x0083 : 0x0087) << 48   // bits[63:48] class/type
     | ((channel & 7) << 2)                         // bits[4:2]
     | sub_type                                     // bits[1:0]  (0/1/2)
second word = 0
```
- Emit through mbox_ep_a (the receiver `+0x19d8`) via vtable slot `+0x8a8`, passing the receiver `+0x19d8`, the word, 0, and the slot pointer.

The `+0x8a8` send writes the word to the **inbox doorbell-set register** (§2.1), raising the
coprocessor's mailbox interrupt. Ringing this against a coprocessor in the
"waiting-for-kick" state (§7.1) is the power kick that wakes it. The host then confirms the
wake via the wake-confirmation predicate bounded by the early-wake timeout (the device object `+0x2194`,
default 5 ms); failure paths log a "wake took too long" diagnostic and run a wake-event
validation routine. **HIGH.** (This doorbell encoding is the same one documented in `03-*`
§3.3 for command submission — i.e. a submission kick and a wake kick are the same primitive.)

### 7.3 Dedicated power/interrupt MMIO regions — HIGH (exist) / GAP (encoding)

Beyond the normal mailbox doorbell, the firmware is granted distinct MMIO **regions**
(entries in the region-mapping array of `02-*` §2.7 / §4). Their existence and role are HIGH
(named by the firmware-side region-type table); their concrete write encodings/offsets are
**GAP** (they live in the firmware image / guarded region, not written from recoverable host
code):
- a **power-management-processor doorbell** region — a doorbell separate from the mailbox
  inbox, for the deep power-managed case;
- **per-die idle-status** regions (two) — the idle/run indicators per die;
- a **GPU software-interrupt** region and a **banked interrupt-controller** region — the
  ~2 interrupt-controller regions of `02-*` §4;
- a **power-transition register block** — backs the §6 sleep-control / power-IRQ accessors.

### 7.4 Emulator / hypervisor reproduction note

To resolve "GPU asleep waiting for an interrupt" while the firmware is parked in its
wait-for-interrupt idle (§7.1):
1. **Common case** — model the **inbox doorbell** (the `+0x8a8` send, i.e. a write to the
   inbox doorbell-set register) as **raising the coprocessor's mailbox interrupt**. That
   single event un-parks the firmware.
2. **Deep-off case** — additionally model the **power-management-processor doorbell** region
   and the **GPU software-interrupt** region (§7.3) as interrupt sources, followed by the
   coprocessor clearing its `power_irq_status` (via the clear role).
The exact power-management-processor doorbell offset/encoding is a **GAP** here — it is the
one piece that must be read from the firmware image / guarded region rather than the
host driver image.

---

## 8. Coprocessor-firmware-side confirmation (from the GPU firmware image)

The sections above are all from the host driver. The coprocessor's own firmware image
was analyzed separately to recover the coprocessor side of the wake. The firmware, unlike
the host driver, touches hardware directly, so several concrete register offsets are
recoverable here.

### 8.1 The wait-for-interrupt idle ladder — HIGH
Two idle primitives:
- **Light park**: `isb ; dsb sy ; wfi ; b .-4` — spins in `wfi` forever; this is the
  terminal "parked, waiting for a kick" state. Any unmasked interrupt resumes past the `wfi`.
- **Context-retaining deep idle**: it saves the callee-saved GPRs and FP/SIMD (d8–d15, FPCR)
  to a per-core save area (pointer in impl-defined system register `S3_1_c15_c0_1`), gates
  the deep path to one core identity (`MIDR_EL1 == 0x610f00e0`), then **sets bit 41
  (`0x2000000000`) of impl-defined system register `S3_0_c15_c10_0`** (the deep-sleep
  enable), executes `dsb sy ; wfi`, and restores the original sysreg value immediately after
  — i.e. the deep-sleep bit is armed for exactly one `wfi`. It returns a boolean "did we
  actually deep-sleep". The context save/restore is the tell that **this deep mode loses
  GPR/FP state across the sleep**. Resume source = any unmasked interrupt (§8.2/§8.3). This
  is the concrete "asleep waiting for an interrupt" state.

### 8.2 Coprocessor-side interrupt take/ack — HIGH
The IRQ vector body (with a fast/NMI variant) calls the interrupt-controller object's
"take/acknowledge pending" op (vtable slot **`+0x18`**), then dispatches through a router,
which reads the pending source/event word (vtable slot **`+0x20`**) and masks/routes it (vtable
slot **`+0x88`**). So the host's inbox-doorbell arrives as an ordinary controller source that
un-parks the `wfi`. The event/acknowledge register is the controller base plus the controller field `+0xe0`
(reading it acks); status bit 17 (`0x20000`) means "no valid event / spurious". The inbound
"ring doorbell / kick" toward the peer is a coprocessor-link device operation, op-code **`0x0d`**;
the link register block lays its inbox / outbox / doorbell pages **16 KiB apart** (sub-regions
at base `+0x0000` / `+0x4000` / `+0x8000`). The literal doorbell store offset within its page is
emitted from a boot-filled dispatch slot → **GAP**.

### 8.3 Software-interrupt set / clear / status registers — HIGH (offsets), GAP (absolute base)
The banked interrupt controller uses these register offsets (`n` = a per-instance die/word index):

| role | offset | conf |
|------|--------|------|
| software-interrupt **SET** (raise) | `+0x100` (controller field `+0xd0`) | HIGH |
| software-interrupt **CLEAR** | `+0x108` (controller field `+0xd4`) | HIGH |
| status / IACK-lo | `n*8 + 0x110` (`+0x114` alt bank) (controller field `+0xd8`) | HIGH |
| pending word | `n*0x40 + 0x800` (controller field `+0xdc`) | HIGH |
| event / IACK read (ack) | `n*0x40 + 0x810` (`+0x830` alt) (controller field `+0xe0`) | HIGH |

(Older non-banked variant: status/IACK at `n*4 + 0x120`; control/config bank at
`0x100`/`0x104`/`0x108`.) The absolute controller base is boot-injected → GAP.

### 8.4 Power-on / wake completion sequence — HIGH (offsets), GAP (absolute bases)
The GPU power-state transition handler drives the transitions; state codes are
**`0x08` = deep-off/sleep, `0x10` = nap, `0x21` = on** (corroborated by the illegal-transition
asserts). Entering the "on" state from deep-off does raw absolute-offset MMIO against several
**boot-injected base globals** (zero in the static image):

(The base globals are distinct boot-injected MMIO windows, labelled A–D here.)

| step | base global | offset | encoding | conf |
|------|-------------|--------|----------|------|
| enable A | base A | `0x104` | `\|= 1` | HIGH |
| enable B | base A | `0x108` | `\|= 1` | HIGH |
| mode | base B | `0x1010` | `= 7` | HIGH |
| enable C | base C | `0x2128` | `\|= 1` | HIGH |
| enable D | base C | `0x2024` | `= 1` | HIGH |
| power-control | a region global (span `0x160000`) | `0x340120` | RMW low bits from a boot nibble | HIGH |
| power-on flag | base D | `0x10` | `= 1` | HIGH |
| **wake counter** | base D | `0x1c` | `+= 1` (fenced) | HIGH |

The sleep/power-down path polls a **per-die GPU-idle status bit (bit 0)** of a boot-injected
status word before gating power. Separate wake/sleep bookkeeping counters exist, paired with
the §8.1 deep-idle context save.

### 8.5 What is still GAP after the firmware pass
The **absolute physical bases** of every region above are boot-injected pointer globals (zero
in the static image) — their offsets/encodings are recovered but the base addresses come from
the device tree / boot data on the host side. In particular the **power-management-processor
doorbell's absolute base and its store offset within the page** remain GAP: the firmware
consumes that stimulus only as a normal controller event (§8.2/§8.3), and the enum→base
mapping is applied host-side, not by a hard-coded store in the firmware.

### 8.6 Net effect on the wake mechanism
Combining §7 (host) and §8 (coprocessor): the host rings the inbox doorbell (§7.2); on the
coprocessor that raises a controller interrupt which resumes the core from the §8.1 `wfi`; the
firmware acks via the event/IACK register (§8.2/§8.3) and runs the §8.4 power-on sequence,
bumping its wake counter. An emulator/HV that delivers the mailbox interrupt on the doorbell
write reproduces the common-case wake end-to-end; only the deep-off power-processor-doorbell
absolute base (§8.5) must be supplied from host boot data.

## 9. Confidence summary and remaining gaps

**HIGH**
- The RX pump is a vtable-installed interrupt action (DATA/vtable refs only); endpoint
  lookup; dispatch; GPU app-endpoint callbacks → GPU-master vtable `+0x1e8`; two-ring drain;
  ring peek/validator + consumer-index credit ACK.
- The outbox drain + status ACK is reached through the provider (receiver `+0xb0`) vtable slot
  `+0x918`; register roles (inbox/outbox control with not-ready/overflow/underflow,
  doorbell-set, run/idle-status).
- Power protocol: high-field message-class tag (`0x00a0` power / `0x0080` roll-call /
  `0x0020` version); power-notify send with defer/flush logic (the device object `+0x124`); run-state
  values 5/6/`0x8000`; the quiesce/wait loop.
- Wake: doorbell word encoding + emit slot (kick → `mbox_ep_a` `+0x8a8`);
  the firmware idle end-state; existence and roles of the power-processor doorbell /
  per-die idle-status / software-interrupt regions and the sleep-control / power-IRQ
  accessors.

**LOW**
- Exact numeric `power_state` values in the `0x00a0` message; roll-call bit layout beyond
  bit51; run-state values `0x04` / `0x8001`.

**HIGH (coprocessor firmware side, §8)**
- The wait-for-interrupt idle ladder (light park + context-retaining deep idle arming bit 41
  of `S3_0_c15_c10_0`, losing GPR/FP state across sleep); the coprocessor-side interrupt
  take/ack (controller vtable slots `+0x18`/`+0x20`/`+0x88`, event/IACK at the base plus the controller field `+0xe0`, spurious =
  status bit 17); the software-interrupt set (`+0x100`) / clear (`+0x108`) / status
  (controller field `+0xd8`) registers and per-die pending offsets; the power-on/wake completion MMIO
  sequence and its state codes (`0x08` deep-off / `0x10` nap / `0x21` on); the per-die idle bit 0.

**LOW**
- Exact numeric `power_state` values in the `0x00a0` message; roll-call bit layout beyond
  bit51; run-state values `0x04` / `0x8001`.

**GAP** (resident in guarded region or boot-injected — not in either static image)
- Host-driver-side concrete MMIO offsets for inbox/outbox control, doorbell-set, and
  run/idle-status, and the hardware IRQ index / device-tree binding of the outbox interrupt.
  (The coprocessor-side register offsets ARE recovered — §8 — but the **absolute physical
  bases** are boot-injected globals, so they come from the device tree / boot data.)
- The power-ack word bitfields (handler pinned by string only, §4.3).
- The power-management-processor doorbell's absolute base and in-page store offset (§8.5):
  the firmware consumes the stimulus only as a normal controller event and the enum→base map
  is applied host-side.
- Exact vtable slot indices behind `+0x1e8` / `+0x1c8` and the drain's own master-vtable
  slot (all invoked via signed vtable pointers).
