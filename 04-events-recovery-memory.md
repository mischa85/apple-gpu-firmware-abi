# Apple GPU Firmware ABI — Firmware→Host Events, GPU Recovery, Firmware-Driven Memory Management

Scope: the firmware→host direction. Documents (1) the event ring the firmware posts
completion/notification events into and its drain/dispatch loop, (2) the per-event payload
structs, (3) the GPU hang/recovery flow and its records, and (4) the firmware-driven memory
management (page-management / UMA pool growth requests, memory-descriptor entries) plus the
GART-invalidate and context-base-address programming.

All names below are my own, assigned from observed behavior.

Primary structures:
- Event-ring drain/dispatch loop: the top-level "peek one entry, switch on its type, handle
  it, advance read index" loop. Confirmed drain semantics by the per-case guard
  `if (entry.type != CASE_N) <fatal>`, i.e. the switch selector equals the discriminant word.
- Ring peek/validator: copies one `0x48`-byte entry, advances read index, checks a per-type
  "valid" bitmask. A second validator variant walks several sub-rings.
- Statistics event consumer: a *second, separate* event namespace, types `0x10`..`0x1e`.
- Hardware-error / recovery record construction: event case 4 in the drain loop, plus a
  host-side hardware-error signalling path.

---

## 1. The firmware→host event ring

### event_ring_reader  (validator/reader control block, size ≥ `0x30`)
Purpose: host-side cursor over the shared firmware→host event ring. One instance per ring
(there are several parallel rings — see §1.3).
Where used: the peek/validator reads all fields; the drain loop embeds one at
the device-state object `+0x148` (device-state words `0x148`..`0x14d`).

| offset | size | name              | type   | conf | meaning / notes |
|------|------|-------------------|--------|------|-----------------|
| `0x00` | 8    | ?                 | u64    | LOW  | present, not read in peek path |
| `0x08` | 8    | shadow_index_ptr  | u32*   | HIGH | pointer to the doorbell/shadow read-index cell; after advancing, `*shadow = read_index` |
| `0x10` | 8    | ring_base         | ptr    | HIGH | base of the entry array; `NULL` ⇒ fatal. Entry stride `0x48` |
| `0x18` | 4    | read_index        | u32    | HIGH | consumer cursor; wrapped mod `entry_count` after each entry |
| `0x1c` | 4    | write_index       | u32    | HIGH | producer cursor (written by firmware); loop empties while `read==write` (the u32 at the device-state object `+0xa5c` snapshot) |
| `0x20` | 8    | valid_type_mask   | u64    | HIGH | bitmask indexed by entry.type: `(mask >> (type & 0x3f)) & 1` must be set or fatal |
| `0x28` | 8    | entry_count       | u64    | HIGH | ring capacity (entries); read_index/write_index must be `< entry_count` |

Notes:
- The `+0x00` head u64 is present but uncharacterized (not read in the peek path); size is `≥ 0x30`. LOW.
- After copying an entry the reader issues a data-memory-barrier then publishes the new
  read_index to `*shadow_index_ptr` (release of the consumed slot back to firmware).
- The drain loop snapshots write_index into a scratch (the u32 at the device-state object `+0xa5c`) and loops until
  `read_index == snapshot`. On a fully-drained pass that also consumed any firmware-trace entries it
  runs the post-drain fixups in §3.

### 1.2 event_ring_entry  (size `0x48` = 72 bytes; 18 x u32)
Purpose: one firmware→host event. Fixed-size, tagged union.
Where used: copied whole (18 dwords) by the peek/validator; consumed field-by-field in the
drain loop. Field meaning of the union body depends on `type` (§2).

| offset | size | name        | type | conf | meaning / notes |
|------|------|-------------|------|------|-----------------|
| `0x00` | 4    | type        | u32  | HIGH | discriminant; switch selector; validated `entry.type == case` per branch |
| `0x04` | `0x44` | payload     | union| HIGH | 17 dwords of type-specific payload; layouts in §2. Copied verbatim |

Scratch decode of the payload as seen in the drain loop (byte offsets into the entry):
`w1`=`+0x04`, `w2`=`+0x08`, `w3`=`+0x0c`, `w4`=`+0x10`, `w5`=`+0x14` (split as u16 lo `+0x14` / u16 hi
`+0x16`), `w6`=`+0x18`, `w7`=`+0x1c`, `w8`=`+0x20`, `w9`=`+0x24`. Later dwords exist but are unused by
most handlers.

### 1.3 event_type enum (values are the drain-loop switch cases `0x00`..`0x0E`)
The valid range is enforced: `if (type > 0xE) skip`. Each numeric value below is
HIGH-confidence (the case guard `entry.type == N`). The *name* of each value is my own,
corroborated by the handler behavior described in §2.

| val  | my name                      | conf | how identified |
|------|----------------------|------|----------|
| `0x00` | EVT_FW_ALIVE (default)       | HIGH | default case; dispatches to vtable slot `+0x870` (the "firmware alive / init-ack" handler) |
| `0x01` | EVT_FW_TRACE                 | HIGH | writes up-to-96 trace-channel enable bits into the trace sink; payload = 4 x u32 bitmask + u16 count |
| `0x02` | EVT_RESERVED_2               | LOW  | falls through to skip (no handler) |
| `0x03` | EVT_RESERVED_3               | LOW  | falls through to skip |
| `0x04` | EVT_CHANNEL_ERROR / GPU_FAULT| HIGH | builds the hardware-error record and calls host hardware-error signal (§3) |
| `0x05` | EVT_RESERVED_5               | LOW  | skip |
| `0x06` | EVT_FAULT_INFO_QUEUE         | HIGH | pushes an `0x20`-byte record onto an 8-deep fault-info ring, then kicks a worker thread |
| `0x07` | EVT_PM_MEMORY_REQUEST        | HIGH | page-management memory request; emits distinct alloc-failure/client-kill trace markers and resolves a mem descriptor, §4.1 |
| `0x08` | EVT_PROCESS_EXIT_COMPLETE    | HIGH | notifies a process-exit sink object (at the device-state object word `0x1f6`) with a status word |
| `0x09` | EVT_UMA_MEMORY_REQUEST       | HIGH | pushes an `0x20`-byte UMA-grow request onto a 64-deep ring, kicks a worker thread, §4.2 |
| `0x0A` | EVT_UMA_GROW_COMPLETE        | HIGH | UMA async grow/alloc completion; optionally sets a "grow failed" flag and wakes the pool, §4.3 |
| `0x0B` | EVT_RESERVED_B               | LOW  | skip |
| `0x0C` | EVT_RT_COMPLETE / RELEASE     | HIGH | looks up an object by 64-bit token in a hash and releases it (completion of an async op), §2.7 |
| `0x0D` | EVT_SHARED_EVENT_SIGNAL       | HIGH | signals a shared/user event object identified by index+value (fence signal), §2.8 |
| `0x0E` | EVT_STAMP_UPDATE              | HIGH | delivers a single 64-bit stamp value to the stamp/timestamp sink |

Naming basis and confidence: the numeric event slots above are taken directly from the drain
loop's dispatch switch (the switch selector equals the entry's discriminant), which is why the
`0x00`–`0x0E` values are HIGH-confidence. Independently, a per-type payload validator describes
each type's payload shape; correlating those shapes against the handler behavior corroborates
the descriptive names above.

Two behaviors are described by the validator but have **no dedicated `0x00`–`0x0E` drain-loop case**
in this build, so their numeric slots are LOW-confidence:
- a GPU-restart notification — GPU restart is instead reached through the channel-error →
  hardware-error path (§3); and
- a metrology/aging result — these arrive via the statistics namespace (§5, type `0x1e`
  frequency/power-state records).

---

## 2. Per-event payload structs

Offsets below are byte offsets *into event_ring_entry* (i.e. relative to `+0x00` = type).

### 2.1 evt_fw_trace  (type `0x01`)
Purpose: firmware asks host to (re)program trace-channel enables; also stamps a timestamp.
Where used: case 1.

| offset | size | name           | type | conf | meaning / notes |
|------|------|----------------|------|------|-----------------|
| `0x00` | 4    | type (=1)      | u32  | HIGH | |
| `0x04` | 4    | mask_word0     | u32  | HIGH | trace channels 0..31; each set bit → enable channel(bit) |
| `0x08` | 4    | mask_word1     | u32  | HIGH | channels 32..63 (bit \| `0x20`) |
| `0x0c` | 4    | mask_word2     | u32  | HIGH | channels 64..95 (bit \| `0x40`) |
| `0x10` | 4    | mask_word3     | u32  | HIGH | channels 96..127 (bit \| `0x60`) |
| `0x14` | 2    | count          | u16  | HIGH | validated `< 0x18`; number of channels/entries |
| `0x16` | ...  | (unused)       |      | LOW  | |

Side effect: a monotonic timestamp is captured into a firmware-state field (`+0x1af68` of the
device state).

### 2.2 evt_channel_error  (type `0x04`) — GPU fault / channel error args
Purpose: firmware reports a GPU/channel fault; host records fault metadata and raises a
hardware-error to the scheduler (start of recovery).
Where used: case 4.

| offset | size | name              | type | conf | meaning / notes |
|------|------|-------------------|------|------|-----------------|
| `0x00` | 4    | type (=4)         | u32  | HIGH | |
| `0x04` | 4    | ?                 | u32  | LOW  | present, not directly consumed in the visible path |
| `0x08` | 4    | seqno / stamp     | u32  | HIGH | compared against a ring fill level (`w2`); accepted if `0xffffffff` or `< current && < 0x80000000` |
| `0x0c` | 4    | fault_class       | u32  | HIGH | `w3`; default `0x80` if `0xffffffff`; stored into hw-error record `+0x18dc4` |

The handler then reads a *fault-detail block* from device memory (the device-state object word `0x19a`, `+0x48bc`..`0x48d0`,
i.e. HW-latched fault registers) and populates the hardware-error record (§3). Confidence HIGH
on the record fields; the entry's own fields beyond seqno/fault_class are LOW.

### 2.3 evt_fault_info_queue  (type `0x06`)
Purpose: firmware posts a detailed fault-info record; host copies it into an 8-deep SPSC ring
and kicks a worker thread for deferred processing.
Where used: case 6. Destination ring at device-state `+0x17e50`, head
`+0x17f54`, tail `+0x17f50`, lock `+0x18760`; `0x20`-byte slots.

| offset | size | name        | type | conf | meaning / notes |
|------|------|-------------|------|------|-----------------|
| `0x00` | 4    | type (=6)   | u32  | HIGH | |
| `0x04` | 4    | seqno       | u32  | HIGH | `w1`; fill-level accept guard as in §2.2 |
| `0x08` | 4    | field_a     | u32  | HIGH | `w2`; validated `< 0x7f`; copied to slot+`0x08` |
| `0x0c` | 4    | field_b     | u32  | HIGH | `w3`; copied to slot+`0x0c` |
| `0x10` | 4    | field_c     | u32  | HIGH | `w4`; validated `< 0x40`; copied to slot+`0x10` |
| `0x14` | 8    | payload64   | u64  | HIGH | `w5..w6` (u16+u16+u32); copied to slot+`0x18` |

Stored record layout (`0x20` bytes) at ring: `{u64 0; u32 field_a; u32 field_b; u32 field_c; u32
pad; u64 payload64}`.

### 2.4 evt_pm_memory_request  (type `0x07`) — page-management memory request
See §4.1 (memory management). Payload fields documented there.

### 2.5 evt_process_exit_complete  (type `0x08`)
Purpose: firmware confirms a GPU-side process/context teardown finished.
Where used: case 8.

| offset | size | name           | type | conf | meaning / notes |
|------|------|----------------|------|------|-----------------|
| `0x00` | 4    | type (=8)      | u32  | HIGH | |
| `0x04` | 4    | status / ctx   | u32  | HIGH | `w1`; passed as the notify argument to the process-exit sink at the device-state object word `0x1f6` (vtable `+0x140`), with a leading tag byte `2` |

### 2.6 evt_uma_memory_request  (type `0x09`)
See §4.2. Payload fields documented there.

### 2.7 evt_rt_complete  (type `0x0C`) — async-op completion / object release
Purpose: firmware reports completion of an async operation keyed by a 64-bit token; host looks
it up and releases the tracking object.
Where used: case `0xC`. Lookup table/hash rooted at device-state `+0x1afe8`.

| offset | size | name        | type | conf | meaning / notes |
|------|------|-------------|------|------|-----------------|
| `0x00` | 4    | type (=`0xC`) | u32  | HIGH | |
| `0x04` | 4    | token_lo    | u32  | HIGH | `w1`; forms the 64-bit token key (w2<<32)\|w1 |
| `0x08` | 4    | token_hi    | u32  | HIGH | `w2`; matched against the tracking object word `0x11c` before release |

### 2.8 evt_shared_event_signal  (type `0x0D`) — shared/user fence signal
Purpose: firmware signals a host-visible shared event object (fence) to a target value.
Where used: case `0xD`. Event-object table at device-state `+0x11a30`, count `+0x11a28`.

| offset | size | name           | type | conf | meaning / notes |
|------|------|----------------|------|------|-----------------|
| `0x00` | 4    | type (=`0xD`)    | u32  | HIGH | |
| `0x04` | 4    | seqno          | u32  | HIGH | `w1`; fill-level accept guard as §2.2 |
| `0x08` | 4    | event_index    | u32  | HIGH | `w2`; validated `>= 0` and less than the count at device_state `+0x11a28`; indexes the event-object table |
| —    | 8    | signal_value   | u64  | LOW  | a 64-bit value is signalled to the event's signal method (vtable `+0x180`), carried by one of the word pairs (w3/w4 or w7/w8); exactly which is LOW/unresolved |

Note: the index in `w2` is HIGH and a 64-bit value is signalled to the object's vtable `+0x180`; which
word pair carries that value (w3/w4 or w7/w8) is LOW/unresolved. Confidence HIGH on the index and that
a 64-bit value is signalled, LOW on which two words form it.

### 2.9 evt_stamp_update  (type `0x0E`) — stamp/timestamp completion
Purpose: firmware publishes a completed command "stamp" (a monotonically increasing completion
counter); host advances all fences waiting on it.
Where used: case `0xE` → stamp sink at device `+0x150`.

| offset | size | name        | type | conf | meaning / notes |
|------|------|-------------|------|------|-----------------|
| `0x00` | 4    | type (=`0xE`) | u32  | HIGH | |
| `0x04` | 8    | stamp_value | u64  | HIGH | the 64-bit value (w2<<32)\|w1 = 64-bit stamp passed to the stamp sink |

### 2.10 evt_fw_alive  (type `0x00`, default)
Purpose: firmware "alive"/init-acknowledge; drain loop hands the whole entry to vtable slot
`+0x870` for one-time init bring-up handling. Payload not further decoded here (LOW).

---

## 3. GPU recovery / hang flow and records

### 3.1 Flow (from the channel-error event, case 4)
1. Firmware posts EVT_CHANNEL_ERROR (type `0x04`).
2. Host calls a HAL fault-latch reader (vtable slot `+0x1d0` on the device object,
   passing the device object and the entry pointer with bit 4 set); on failure it logs and faults.
3. Host reads the HW fault-detail block from `deviceMMIO + 0x48bc..0x48d0` and fills the
   **hardware_error_record** (§3.2).
4. Host calls the scheduler hardware-error signal — this begins the host-driven
   recovery/reset (quiesce, reset, replay).
5. On the *post-drain* pass, if any EVT_FW_TRACE was seen (a recovery marker), the loop runs
   the recovery-fixup block: flush stamp sink, check FW halted and, if halted, drives two
   power/rail gates (vtable slot `+0x1f0` on the objects at device_state `+0x618`/`+0x628`) and a reset (vtable slot `+0x198`
   on the object at device_state `+0xf638`). A watchdog timeout is (re)armed via device_state `+0x6c8` (1,000,000,000 ns).

### 3.2 hardware_error_record  (embedded in device state at `+0x18dc0`)
Purpose: latched description of the GPU fault that triggered recovery; consumed by the
scheduler hardware-error path.
Where used: written in case 4; base = `device_state + 0x18dc0`.

| offset (abs) | offset (rel) | size | name              | type | conf | meaning / notes |
|-----------|-------|------|-------------------|------|------|-----------------|
| `0x18dc0`   | `0x00` | 4    | fault_status      | u32  | HIGH | copied from MMIO `+0x48c8` |
| `0x18dc4`   | `0x04` | 4    | fault_class       | u32  | HIGH | from entry `w3` (default `0x80`) |
| `0x18dc8`   | `0x08` | 1    | fault_kind        | u8   | HIGH | derived from MMIO `+0x48cc` via a small lookup `(0x605040202 >> ...)`; nonzero also acts as a "recovery in progress" gate for events 6/9 |
| `0x18e98`   | `0xd8` | 8    | fault_address     | u64  | HIGH | copied from MMIO `+0x48d0` (faulting GPU VA) |
| `0x18ea0`   | `0xe0` | 1    | flags             | u8   | HIGH | bit3 set if MMIO `+0x48bc != 0` (fault-valid flag) |

**Note:** This is a sparse latch record embedded in device state; only the fields listed are characterized. `+0x09`..`+0xd7` and beyond `+0xe1` are unmapped, and no fixed total size is asserted.

### 3.3 Scheduler status / recovery-info structs — GAP
The named helpers for "decode & print scheduler status", "block for ring empty", "is FW awake",
"is the GPU booted", "process firmware-initiated recovery", and "process vertex/tiling-command
recovery" have no observable body in this build. Consequently the scheduler-status bitfield
struct and the recovery-info struct they would decode are **not recovered in this pass**.
Confidence: this is a firm negative — the byte-level layout of the scheduler-status word is a
documented GAP.

The observable recovery state instead lives in the fields touched by the fixup block:
- `device_state + 0x620` / `+0x630` : rail/power gate "already-done" latches (u8)
- `device_state + 0x6c0` : watchdog-armed flag (u8); `+0x6c8` : 8-byte deadline
- `device_state + 0x18dc8` : fault_kind doubling as "recovery active" gate (see §3.2)

---

## 4. Firmware-driven memory management

The firmware requests host memory in two channels: **page-management (PM) memory requests**
(type `0x07`) and **UMA pool grow requests** (type `0x09`), with a matching **grow-complete**
notification (type `0x0A`).

### 4.1 evt_pm_memory_request  (type `0x07`) — page-management memory request
Purpose: firmware needs the host to (de)allocate/kill pages for a client context; failure and
kill paths emit distinct short trace-code markers (allocation-failure vs client-kill).
Where used: case 7.

| offset | size | name          | type | conf | meaning / notes |
|------|------|---------------|------|------|-----------------|
| `0x00` | 4    | type (=7)     | u32  | HIGH | |
| `0x04` | 4    | op            | u32  | HIGH | `w1`; op selector: 0=alloc-fail(→status `0xd`), 1=client-kill(→status `0xe`), 2=?, 3=trace-only, 4=?(→status 8); validated `< 5` |
| `0x08` | 4    | pool_id       | u32  | HIGH | `w2`; validated `< 3`; also indexes a pool-name lookup table |
| `0x0c` | 4    | context_id    | u32  | HIGH | `w3`; used (with the device-state object word `0x54` and context_id) to resolve the owning context object |
| `0x10` | 4    | seqno         | u32  | HIGH | `w4`; fill-level accept guard as §2.2 |
| `0x14` | 4    | sub_id        | u32  | HIGH | `w5` (u16 lo/hi); used to match a per-context command entry (the driver object `+0x84`/`+0x88`) |

Behavior: resolves the context, walks its in-flight command ring to find the matching command
(by `sub_id` low byte + high 24 bits), records an error status on the command's completion
object (`+0xc8`), then posts a *host→firmware reply* structured as
`{u32 tag=9; u32 seqno; u64 ctx_ptr_field(+0x80); u64 index}` through a device queue
(`device_state + 0x3e0`, method `+0x1e8`). The reply tag `9` mirrors the firmware event id for
the completion handshake.

### 4.2 evt_uma_memory_request  (type `0x09`) — UMA pool grow request
Purpose: firmware asks host to grow a UMA (unified-memory) pool; host enqueues a grow request
for a worker thread and kicks it.
Where used: case 9. Destination ring at device-state `+0x17f58`, head
`+0x1875c`, tail `+0x18758`, lock `+0x18768`; 64 slots x `0x20` bytes.

| offset | size | name         | type | conf | meaning / notes |
|------|------|--------------|------|------|-----------------|
| `0x00` | 4    | type (=9)    | u32  | HIGH | |
| `0x04` | 4    | size_or_id   | u32  | HIGH | `w1`; validated `< 0x100`; stored to slot+`0x00` |
| `0x08` | 8    | gpu_va       | u64  | HIGH | `w2..w3` = the 64-bit value (w3<<32)\|w2; must be non-zero; stored to slot+`0x08` |
| `0x10` | 8    | value2       | u64  | HIGH | `w5..w6` (the 64-bit value (w6<<32)\|(u16 w5)); stored to slot+`0x10` |
| `0x18` | 4    | callback_tok | u32  | HIGH | `w7`; nonzero ⇒ *blocking* (spin until ring slot free); zero ⇒ drop if full. Stored to slot+`0x18` |
| ...  |      | seqno        |      | HIGH | a fill-level accept guard also applies (as §2.2) |

Stored grow-request slot (`0x20` bytes): `{u32 size_or_id; u32 pad; u64 gpu_va; u64 value2; u32
callback_tok; u32 pad}`. After enqueue, kicks the worker thread at device `+0x470` (method `+0x1f8`).

### 4.3 evt_uma_grow_complete  (type `0x0A`) — UMA async grow/alloc completion
Purpose: firmware confirms an async UMA grow finished (or failed); host clears a busy flag /
records failure and wakes the pool allocator.
Where used: case `0xA`.

| offset | size | name            | type | conf | meaning / notes |
|------|------|-----------------|------|------|-----------------|
| `0x00` | 4    | type (=`0xA`)     | u32  | HIGH | |
| `0x04` | 8    | token/handle    | u64  | HIGH | the 64-bit value (w2<<32)\|w1; must be non-zero to accept |
| `0x14` | 2    | failed_flag     | u16  | HIGH | `w5` low u16; if non-zero, sets a "grow failed" flag (writes 1 to the pool `+0xc`) at device_state `+0x119b8` |

Side effect: wakes the pool via a pool-wake helper on `device_state + 0xf0`.

### 4.4 mem_descriptor_entry  (size `0x18` = 24 bytes)
Purpose: a single GPU-VA↔host mapping descriptor inserted into the firmware boot/mapping table
(consumed when programming firmware address space); "new mapping" entry.
Where used: appended by the mapping-table insert primitive. Table base = the driver object word `0x353`, count
= the driver object word `0x354` (u16), capacity = the driver object `+0x1aa2` (u16); stride `0x18`.

| offset | size | name       | type | conf | meaning / notes |
|------|------|------------|------|------|-----------------|
| `0x00` | 8    | field0     | u64  | HIGH | first 8 bytes of the source descriptor, written to entry+`0x00` |
| `0x08` | 8    | field1     | u64  | HIGH | written to entry+`0x08` (source `{u32,u32}` pair) |
| `0x10` | 8    | field2     | u64  | HIGH | written to entry+`0x10` |

Exact semantics (which of {gpu_va, phys/host_addr, size|flags} each 8-byte word is) are not
disambiguated by the visible code — LOW on individual field meaning, HIGH on the 3x u64 layout
and that it is an append-into-fixed-capacity-table mapping record. After appending, a
"notify new mapping" callback runs.

### 4.5 GART invalidation and context base-address programming — GAP
The named helpers "request GART invalidate" and "submit context-ID base address" are, like the
recovery helpers in §3.3, not observable in this build. Their host→firmware message encodings
are therefore **not recovered here** (GAP). The only adjacent, observable programming is:
- the fault-detail latch read at `deviceMMIO + 0x48bc..0x48d0` during channel-error (§3.2), and
- the mapping-table append in §4.4 which is the host-side representation of context address
  space that firmware programming would consume.

---

## 5. Second event namespace — firmware statistics events (types `0x10`..`0x1E`)

Distinct from the `0x00`..`0x0E` command/notification ring, firmware also posts *statistics/telemetry*
records (valid range `0x10..0x1E`, guard `type - 0x10 <= 0xE`). These feed named
counters/state-machines rather than driving host actions. Documented at struct-granularity as
one tagged record (same `type` discriminant at `+0x00`; per-type dword payload). HIGH-confidence
identified sub-types (by their emitted ASCII trace tags):

| val  | my name              | conf | notes |
|------|----------------------|------|-------|
| `0x10` | STAT_GPU_UTIL        | HIGH | default case computes busy/idle deltas |
| `0x11` | STAT_RANGE_A         | LOW  | inserts value into range object `+0x90` |
| `0x12` | STAT_RANGE_B         | LOW  | inserts into `+0x88` |
| `0x13` | STAT_ACCUM_2X        | HIGH | 2 x u64 accumulate at `+0x40`/`+0x4c` from payload `w3..`/count `w6` |
| `0x14` | STAT_ACCUM_1X        | HIGH | adds `w1` to counter `+0x30` |
| `0x15` | STAT_MARK            | LOW  | keyed by magic `w1==0xa746 / 0x8d7f` |
| `0x16` | STAT_RESTART_REASON  | HIGH | records restart/driver-restart reason (matches a specific host userspace daemon process name) |
| `0x17` | STAT_UV_COUNT        | HIGH | |
| `0x18` | STAT_ZONE           | HIGH | |
| `0x19` | STAT_PM_LOOP        | HIGH | power-management loop state; validates many sub-fields (w7..w0x10) |
| `0x1a` | STAT_POWER_SEQ      | LOW  | |
| `0x1b` | STAT_RANGE_C        | LOW  | |
| `0x1c` | STAT_GPU_SW_STATE   | HIGH | |
| `0x1d` | STAT_NOOP           | LOW  | falls to fatal/skip |
| `0x1e` | STAT_FREQ_STATE     | HIGH | frequency/power state machine + P-state residency; this is the likely home of metrology/aging telemetry |

These record types are secondary to the requested scope; documented here so the two event
namespaces are not confused. Field-level layout of each statistics sub-type is not fully
expanded (LOW where marked).

---

## Summary of confidence & gaps
- HIGH: the event-ring reader control block (§1.1), the `0x48`-byte tagged entry (§1.2), the
  `0x00`..`0x0E` event-type enum values and their handler behavior (§1.3), and the payload structs
  for FW_TRACE, CHANNEL_ERROR, FAULT_INFO_QUEUE, PROCESS_EXIT, RT_COMPLETE, SHARED_EVENT_SIGNAL,
  STAMP_UPDATE, PM_MEMORY_REQUEST, UMA_MEMORY_REQUEST, UMA_GROW_COMPLETE (§2, §4).
- HIGH: the hardware_error_record built during recovery (§3.2) and the recovery flow (§3.1).
- GAP (firm negatives, not recovered in this build): the scheduler-status decode struct and
  firmware-initiated-recovery info struct (§3.3), and the GART-invalidate / context-base-address
  host→firmware message encodings (§4.5).
- LOW: individual field semantics of mem_descriptor_entry (§4.4), the exact word packing of the
  shared-event 64-bit signal value (§2.8), and the reserved event slots `0x02`/`0x03`/`0x05`/`0x0B`.
