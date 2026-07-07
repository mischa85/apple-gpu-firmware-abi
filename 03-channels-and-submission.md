# 03 — Host→Firmware Command Submission: Rings, Channels, Command Structs

Clean-room reconstruction of the host-driver → GPU-firmware command-submission path
for the t8140 / "G17P" GPU. All names below are my own.
Confidence: HIGH = unambiguous. LOW = inferred.

--------------------------------------------------------------------------------
## 0. Big picture (how a workload reaches the GPU)

There are two logically distinct ring systems, both living in memory shared with the
firmware coprocessor:

1. **Per-context work rings ("channels").** One set per GPU context, one channel per
   workload class. A channel is a ring of 8-byte *command pointers*; each pointer
   references a large per-workload command struct (the "work item") that the driver
   fills in in shared memory. Three workload classes:
   - **vertex/tiling** (I call it `WL_VERTEX`)
   - **fragment/render** (`WL_FRAGMENT`)
   - **compute** (`WL_COMPUTE`)

2. **Two global service rings** used to *notify/kick* the firmware and to issue
   device-wide control commands:
   - a **device-control ring** (ring index 0), entries `0x40` bytes
   - a **data-master ring** (ring index 1), entries `0x40` bytes

The actual "ring index for a given command opcode" is selected by a lookup table
(§4.3). A doorbell to the coprocessor is a single 64-bit mailbox word (§4.1).

--------------------------------------------------------------------------------
## 1. The channel / ring model

### 1.1 channel_ring_header  (the shared ring bookkeeping block)
Purpose: the shared-memory control block for one ring; holds the backing entry
array pointer, the ring capacity, and read/write/CFI indices that host and firmware
poll to flow-control the ring.
Where used: read by the ring accessors (write index, read index, CFI index) and by the
generic ring reader `peek_next_entry`. The accessor reads the pointer at ring_object +8
to get this header pointer, then indexes it.

| offset | size | name | type | conf | meaning |
|-----|------|------|------|------|---------|
| `0x00`| 4 | read_index | u32 | HIGH | consumer/read pointer. The u32 pointed to by the header pointer at the channel object +8. Also the "CFI/consumed" view used in `peek_next_entry` at header+`0x18` (see note). |
| `0x10`| 4 | cfi_index | u32 | HIGH | "consumed-for-interrupt" / secondary read cursor. The u32 at header `+0x10`. |
| `0x20`| 4 | write_index | u32 | HIGH | producer/write pointer. The u32 at header `+0x20`. All three indices are validated as `< 0x100` (max ring depth 256 entries). |

Note: Header spans at least `+0x24`; indices sit `0x10` apart (read `+0x00`, cfi `+0x10`, write `+0x20`). The interior ranges `+0x04`..`+0x0f` and `+0x14`..`+0x1f` are undocumented (padding / cache-line isolation, unconfirmed).

Note on the *validator's* view of a ring (a superset wrapper, used by
`peek_next_entry` and `validate_type`): that wrapper struct is `0x48` bytes per ring and
packs pointers + live indices:

### ring_validator_slot  (size `0x48`; per-ring, in the validator array)
Purpose: driver-side mirror used to walk a ring entry-by-entry and validate opcodes.
Where used: `peek_next_entry`; `validate_type` iterates the channel object + type*`0x48` +
offset (stride `0x48` confirmed there, indexing the channel object by the command-buffer
object times `0x48`).

| offset | size | name | type | conf | meaning |
|-----|------|------|------|------|---------|
| `0x08`| 8 | shadow_index_ptr | ptr | HIGH | pointer written back with the new read index: the u32 pointed to by the channel object +8 is set to the u32 at the channel object `+0x18`. |
| `0x10`| 8 | entry_array | ptr | HIGH | base of the entry array; entries are `0x48` bytes each: entry base + idx*`0x48`. If 0 → fatal "ring empty" path. |
| `0x18`| 4 | read_cursor | u32 | HIGH | current read position, advanced modulo capacity. |
| `0x1c`| 4 | write_cursor | u32 | HIGH | if `read==write` the ring is empty (early-out). |
| `0x20`| 8 | opcode_valid_mask | u64 | HIGH | bitmask keyed by entry opcode: the u64 at the channel object `+0x20`, shifted right by `(opcode & 0x3f)`, masked with 1, gates acceptance. |
| `0x28`| 8 | capacity | u64 | HIGH | ring depth; index compared `< capacity` and used as modulus. |

Note: Uncovered within the `0x48` slot: `+0x00`..`+0x07` (head u64, unread) and `+0x30`..`+0x47` (`0x18` bytes past the last documented field) — undocumented.

`peek_next_entry` copies **`0x48` bytes (18 × u32)** out of the entry into the caller's
buffer (the command-buffer object words 0..`0x11`) — i.e. the validator ring entry is
**`0x48` bytes**.

### 1.2 channel_object (the per-channel driver object; the channel object argument of every submit)
Purpose: driver-side channel that owns the 8-byte-pointer ring and the doorbell state.
Where used: first arg to the ring-write primitive and to all three per-class submit functions.

| offset | size | name | type | conf | meaning |
|-----|------|------|------|------|---------|
| `0x18`| 8 | current_slot | u32 | HIGH | this channel's ring slot id (0..7); ORed into the doorbell as `(slot&7)<<2` — see §4.1. |
| `0x54`| 4 | ptr_ring_capacity | u32 | HIGH | capacity of the 8-byte command-pointer ring; write index checked `< this`. |
| `0x60`| 8 | ptr_ring_state | ptr | HIGH | → sub-struct holding the pointer-ring counters (`+0x10` write idx, `+0x18` stride, `+0x40` published idx, `+0x60` modulus). |
| `0x68`| 8 | ptr_ring_base | ptr | HIGH | base of the 8-byte command-pointer array; entry written at `base + slot*8`. |
| `0xe0`| 8 | aux_ring_base | ptr | LOW | second (optional) pointer ring used when the aux-ring flag is nonzero in the ring-write primitive. |
| `0xe8`..`0xf0`| 4,4,4 | aux_ring modulus/idx | u32 | LOW | counters for the aux ring. |

--------------------------------------------------------------------------------
## 2. Command-pointer ring entry & the ring-write primitive

### 2.1 ring_command_pointer_entry  (size 8)
Purpose: one slot of a channel's command-pointer ring; a single 64-bit GPU/shared
pointer to the workload command struct that firmware should execute next.
Where used: written by the ring-write primitive.

Behaviour of the primitive:
- Spin until the pointer ring is not full: recompute `(write+1) mod cap` and compare
  to read index.
- Store the 64-bit pointer: the command-buffer object is written to the u64 at
  ptr_ring_base + slot*8.
- **a data-memory barrier (DMB ISH)** right after the store — ordering the payload
  before the index publish. HIGH.
- Optionally store a second value into the aux ring when the aux-ring flag is nonzero.
- Publish the new write index into ptr_ring_state `+0x40` (modulo `+0x60`). HIGH.

| offset | size | name | type | conf | meaning |
|-----|------|------|------|------|---------|
| `0x00`| 8 | work_item_gpu_ptr | u64 | HIGH | GPU/firmware pointer to the workload command struct. |

--------------------------------------------------------------------------------
## 3. Per-workload command structs (the "work items")

All three per-class submit functions share the same skeleton. The channel
object (§1.2) is the first argument; the command-buffer object is the per-context "command buffer" / descriptor object.
The context id is the u32 at the command-buffer object `+0x144` (WL_VERTEX uses `+0x1274`), threaded into
almost every struct as `context_id`. Each function allocates/obtains a large work-item
struct (the work item) and fills a fixed *header* that
firmware reads first, plus a class-specific body (the body is huge — hundreds of
fields copied 1:1 from the command-buffer object; I document the firmware-visible **header** exhaustively
and summarize the body).

### 3.0 Shared work-item header fields (offsets in the allocated struct)
Identical stores in all three functions.

| offset | size | name | type | conf | meaning |
|-----|------|------|------|------|---------|
| `0x00`| 4 | opcode | u32 | HIGH | workload opcode written first: `1` for vertex path, `2` fragment, `3` compute, `0` for the vertex/tiling "start" item. See §3.4 opcode table. |
| `0x04`| 8 | submit_seq | u64 | HIGH | monotonically increasing submission id, fetched-and-incremented from a global counter at the queue-root object `+0x5e8` and stored to the work item +4. |
| `0x0c`| 4 | context_id | u32 | HIGH | the u32 at the command-buffer object `+0x144` (vertex: `+0x1274`) copied in. Same value used as the doorbell/notify tag. |
| — | — | event/stamp block | mixed | HIGH | three "current value" fields are computed via a helper at vtable slot `+0x2d8` (returns a GPU stamp address for a fence object) and stored as u64s (frag: work item `+0x28`/`0x30`/`0x38`). These are the completion-stamp GPU addresses firmware writes on finish. |

### 3.1 Address-pair encoding (HIGH / LOW split for every GPU pointer)
Every GPU virtual address the driver hands the firmware is composed as
`base + fixed_offset + (per-slot delta)` and stored as a **single u64** (not
split into HIGH/LOW halves) — e.g. the value = tile_heap_base + `0x5a0` + slot_delta,
then stored at entry offset `0x850` as one 64-bit value. So in this ABI the pointers are
**64-bit little-endian words**, HIGH=upper 32, LOW=lower 32 of the same u64; I note this
once rather than per-field. The two bases come from the pointer at the channel object
word 2, `+0x18788` and `+0x18790` (two shared-VA regions). HIGH.

These `+0x18788`/`+0x18790` bases hang off the channel object's word-2 pointer (shared-VA region bases) and are distinct from the init-data config tree's `+0x18xxx` offsets in `02`.

### 3.2 fragment_work_item (opcode 2)
Purpose: a render/fragment workload; the largest struct (body extends past `+0x2200`).
Where used: pointer to it is enqueued via §2 and firmware kicked via `kick_fragment_channel`
(§4.1, doorbell type index **1**).

Header/near fields (offsets into the work item):

**Note:** the early header fields (opcode, submit_seq, context_id, stamp block) are byte-pinned (HIGH); the upper-region rows below are the word-indexed field order recovered from stubbed sites — treat them as *relative field order*, not exact byte offsets (LOW).

| offset | size | name | type | conf | meaning |
|-----|------|------|------|------|---------|
| `0x00`| 4 | opcode=2 | u32 | HIGH | writes 2 (else-branch) / writes 1 for the "prep" variant. |
| `0x04`| 8 | submit_seq | u64 | HIGH | counter. |
| `0x0c`| 4 | context_id | u32 | HIGH | |
| `0x8c0`..`0x8c8` (frag `+0x850`/`+0x852`)| 8,8 | tiler_va / render_va | u64,u64 | LOW | the two composed GPU VAs from §3.1 (work item `+0x850`, `+0x852`). |
| `0x854`| 4 | prim_kind | u32 | LOW | the first word of the descriptor. |
| `0x855`| 4 | tile_count | u32 | LOW | the u32 at the descriptor +4. Also used as the ring-slot for the pointer store (§2). |
| `0x857`| 4 | needs_flush | u32 | LOW | boolean: the command-buffer object word 9 is nonzero. |
| `0x858`| 8 | pipeline_tag | u64 | LOW | packed slot/gen value computed from `queue_slot<<0x28 | gen<<8 | idx` and mirrored to the command-buffer object word `0x2f`. |
| ~`+0x222d`| 1 | fault_flags | u8 | LOW | bit0 = "has scissor/depth-bounds"; bits2-3 = the value at the channel object word 2, `+0xf610` masked with 3 (a global fault/debug mode). |
| `0x220d`..`0x2225`| 4×8 | clear_rects[4] | u64 | LOW | four rect/bounds u64s copied from the command-buffer object words `0x14f`..`0x152` only when fault_flags bit0 set. |

Body (summarized, HIGH that it's a bulk copy): offsets `0x748`..`0x83f` are a verbatim
copy of ~90 consecutive u64s from the command-buffer object words `0x96`..`0x188` (register/state image)
— this is the fragment pipeline register state block. Individual semantics LOW.

### 3.3 compute_work_item (opcode 3)
Purpose: a compute workload; kicked via `kick_compute_channel` (doorbell type index **2**).

**Note:** the early header fields (opcode, submit_seq, context_id, stamp block) are byte-pinned (HIGH); the upper-region rows below are the word-indexed field order recovered from stubbed sites — treat them as *relative field order*, not exact byte offsets (LOW).

| offset | size | name | type | conf | meaning |
|-----|------|------|------|------|---------|
| `0x00`| 4 | opcode=3 | u32 | HIGH | writes 3. |
| `0x04`| 8 | submit_seq | u64 | HIGH | |
| `0x0c`| 4 | context_id | u32 | HIGH | copies the u32 at the command-buffer object `+0x144` into the work item `+0x0c`. |
| `0x3d0`| 8 | dispatch_va_a | u64 | LOW | composed VA `base+0x5a0+delta` (work item `+0x3d0`). |
| `0x3d2`| 8 | dispatch_va_b | u64 | LOW | second composed VA. |
| `0x3d4`| 4 | grid_kind | u32 | LOW | the first word of the descriptor. |
| `0x3d5`| 4 | grid_count | u32 | LOW | the descriptor word 1; also the ring slot. |
| `0x3d7`| 4 | needs_flush | u32 | LOW | the command-buffer object word 9 is nonzero. |
| `0x3d8`| 8 | pipeline_tag | u64 | LOW | packed slot/gen (same scheme as fragment) mirrored to the command-buffer object word `0x2f`. |
| `0x407`| 1 | fault_flags | u8 | LOW | same bit0/bits2-3 encoding. |
| `0x3ff`..`0x405`| 4×4 | clear_rects[4] | u64 | LOW | copied from the command-buffer object words `0xe4`..`0xe7`. |

Compute path also validates a *protection-options*-derived index: the argument comes
from the command-buffer object word `0xf4`, `+0x10` and must be `< 3`, corroborated by the
bounds-check diagnostics on the priority index and the event-flag index.

### 3.4 vertex_work_item (opcode 0/5)
Purpose: vertex/tiling workload; kicked via `kick_vertex_channel` (doorbell type index
**0**). Uses context id from the command-buffer object `+0x1274` and workload id from the command-buffer object word `0x18a`.

**Note:** the early header fields (opcode, submit_seq, context_id, stamp block) are byte-pinned (HIGH); the upper-region rows below are the word-indexed field order recovered from stubbed sites — treat them as *relative field order*, not exact byte offsets (LOW).

| offset | size | name | type | conf | meaning |
|-----|------|------|------|------|---------|
| `0x00`| 4 | opcode | u32 | HIGH | writes 0 for the tiling item; a separate "kick"/marker item uses opcode `5` (in the branch where the byte at the command-buffer object `+0x1264` equals 2). |
| `0x04`| 8 | submit_seq | u64 | HIGH | |
| `0x0c`| 4 | context_id | u32 | HIGH | copies the u32 at the command-buffer object `+0x1274` into the work item `+0x0c`. |
| `0x8a6`| 8 | tiler_va | u64 | LOW | composed VA. |
| `0x8ae`| 8 | render_target_va | u64 | LOW | second composed VA. |
| `0x8b6`| 4 | prim_kind | u32 | LOW | the int at the command-buffer object `+0xc4c`. |
| `0x8ba`| 4 | tile_count | u32 | LOW | the u32 at the command-buffer object `+0x18a` (workload id / ring slot). |
| `0x8c2`| 4 | needs_flush | u32 | LOW | the command-buffer object word 9 is nonzero. |
| `0x98e`| 1 | fault_flags | u8 | LOW | bit0/bits2-3 as above. |

--------------------------------------------------------------------------------
## 4. The doorbell (kick) & the two global service rings

### 4.1 Doorbell mailbox word encoding  (64-bit)
The three kick functions (`kick_vertex_channel`, `kick_fragment_channel`,
`kick_compute_channel`) all end by calling an indirect mailbox-send through vtable slot
`+0x8a8` (reached via the pointer at the root object `+0x33b`) with a single constructed
64-bit word:

```
doorbell = 0x8300000000000000
         | ((channel_slot & 7) << 2)     // low bits select the channel slot
         | workload_type_index           // 0=vertex, 1=fragment/render, 2=compute
```
Encoding:
- vertex kick: `... | 0x83000000000000` (type 0).
- fragment kick: `... | 0x83000000000001` (type 1).
- compute kick: `... | 0x83000000000002` (type 2).
The `(x & 7) << 2` slot field and the `0x83..` tag are HIGH.

Before kicking, each function does a timing-window bookkeeping read of a small
histogram at `+0x670`..`0x684` of the object at the channel root word `0x19b` (a "submissions per interval" counter,
guarded by a seqlock on `+0x680`). LOW meaning, HIGH that it's telemetry, not part of
the ABI.

### 4.2 Data-master notify message  (the "advance" doorbell)
The generic data-master submit core sends a *different* mailbox word for each completed
sub-entry:
```
notify = 0x85930c00 , arg0 = entry_value , arg1 = queue_tag , ...
```
i.e. `func(0x85930c00, entry, tag, 0, opcode, 0)`. The `0x8593xxxx` family is the
coprocessor-style endpoint tag; `0x85930cb8` is used by the vertex path completion notify.
HIGH that these are distinct mailbox opcodes; exact bitfields LOW.
Right before the ring-index publish it issues a data-memory barrier (DMB ISH) and then
bumps the ring's write index via a virtual call (vtable slot `+0x10` after reading vtable
slot `+0x38`).

### 4.3 Command-opcode → ring-index selector table
`get_ring_index_for_entry` (device-control variant) looks up the target ring from a u32
table indexed by the entry's opcode (the entry's word 0): value 0 → device-control ring,
1 → data-master ring.

Table dump (u32 LE), index = command opcode:
```
idx : ring     idx : ring     idx : ring     idx : ring
 0  : 0         16 : 1         32 : 0         48 : 1
 1  : 0         17 : 1         33 : 1         49 : 1
 2  : 0         18 : 0         34 : 1         50 : 1
 3  : 0         19 : 0         35 : 1         51 : 0
 4  : 0         20 : 0         36 : 0         52 : 0
 5  : 0         21 : 0         37 : 0         53 : 0
 6  : 0         22 : 1         38 : 0         54 : 0
 7  : 0         23 : 1         39 : 0         55 : 1
 8  : 0         24 : 0         40 : 1         56 : 1
 9  : 0         25 : 0         41 : 0         57 : 1
10  : 0         26 : 1         42 : 0         58 : 1
11  : 1         27 : 1         43 : 1         59 : 1
12  : 1         28 : 0         44 : 1
13  : 1         29 : 1         45 : 1
14  : 0         30 : 1         46 : 1
15  : 0         31 : 1         47 : 1
```
(HIGH: the table exists and selects the ring; the exact opcode→meaning naming is LOW
except where §4.5 recovers names.)

A parallel **byte** table (per opcode, dumped) is a "needs power-gate poke" flag consulted
right after enqueue (gates a call to vtable slot `+0xab0`); its values
are `{0,1,1,1,1,1,1,1,1,1,1,1,1,1,0,1,1,1,1,1,1,0,0,1,1,1,1,0,1,1,1,1,1,1,0,1,...}`.
The opcode-acceptance bitmask used alongside is the constant `0x3065be399f0c7ff`
(each set bit = an opcode legal to auto-power-gate). HIGH values, LOW naming.

### 4.4 device_control_ring_entry / data_master_ring_entry  (size `0x40`)
Purpose: a fixed 64-byte command written into one of the two global service rings.
Where used: `get_ring_index_for_entry` copies **`0x40` bytes** (8 × u64) from the
caller-supplied entry into the target ring slot (ring_base + write_idx*`0x40`, 8 u64
stores into the ring slot words 0..7), then publishes via vtable slot `+0x10` after a
data-memory barrier (DMB ISH).

| offset | size | name | type | conf | meaning |
|-----|------|------|------|------|---------|
| `0x00`| 4 | opcode | u32 | HIGH | first word; used to index both the ring-select table (§4.3) and the power-gate table. |
| `0x04`| 4 | count/len | u32 | LOW | the entry's word 1 — copied verbatim; role inferred. |
| `0x08`| `0x38` | payload | bytes | LOW | remaining 7 u64s copied 1:1. Layout is opcode-specific; not individually recovered. |

Ring depth for both service rings is **256** (indices validated `< 0x100`). Their
`channel_ring_header` (read/cfi/write at header `+0x00` / `+0x10` / `+0x20`) is exactly §1.1. HIGH.

### 4.5 Channel-command-type opcode enum (recovered values)
Opcodes observed as literals written into work-item/notify entries by the submit code,
plus one firmware-side name:

| opcode | my name | conf | notes |
|--------|---------|------|----------|
| `0x00` | CMD_VERTEX_START | HIGH | writes `0x00` to the vertex work item; and the "type 0" doorbell. |
| `0x01` | CMD_VERTEX_PREP / CMD_RENDER_A | HIGH | writes `0x01` (fragment prep); opcode `0x01` also validated with a `<0x18` sub-field in `validate_type`. |
| `0x02` | CMD_FRAGMENT | HIGH | writes `0x02` to the work item. |
| `0x03` | CMD_COMPUTE | HIGH | writes `0x03` to the work item. |
| `0x04` | CMD_FENCE_WAIT | HIGH | writes `0x04` when emitting a barrier/fence entry, followed by a stamp lookup; `validate_type` re-checks `entry==0x04` and reads a `0xffffffff` sentinel. |
| `0x05` | CMD_TIMESTAMP | HIGH | writes `0x05` to the marker item; corroborated by the timestamp-encoder name. |
| `0x0e` | CMD_INTR_NOTIFY | HIGH | writes `0x0e`, an interrupt/notify entry (writes context id, stamp, seq) in the NOP/prepare path and mirrored in all three submits. |
| `0x10` | CMD_ERROR_NOTIFY | HIGH | writes `0x10` with a `0xfe` sub-code; emitted on the error/timeout path. |
| `0x11` | CMD_KICK_HEADER | HIGH | writes `0x11` — the header of a fresh NOP/kick entry, written before the two composed VAs, in `emit_nop_kick_entry` and reused by the vertex NOP path. |
| — | CMD_PM_REBUILD | HIGH | name only; a firmware-side fatal-error log names a power-management "rebuild" command that is rejected when it arrives on the secure command-submission path. No literal identifier reproduced here. |

Note: this host→firmware command-opcode enum is a different namespace from the firmware→host event-type enum in `04` §1.3 — the numeric values `0x0e`/`0x10` mean different things in each.

--------------------------------------------------------------------------------
## 5. The command-queue struct (context / priority / protection / stamps)

Purpose: the per-context command queue object; carries the firmware context id,
scheduler priority, protection-options index, and completion-stamp counters. This is
the command-buffer object / "root" object threaded through submission.
Where used: fields read all over the submit functions; a priority mapping path; the
serialized type-encoding for the user-facing descriptor (decoded in §5.2).

| offset | size | name | type | conf | meaning |
|-----|------|------|------|------|---------|
| `0x144` | 4 | context_id | u32 | HIGH | copied into every work item as `context_id`. (Vertex queue uses `0x1274`.) |
| `0x450` | 4 | requested_priority | u32 | HIGH | driver priority level; compared to cached `+0x80c` in the priority-update path; out-of-range → an invalid-priority diagnostic. |
| `0x454` | 4 | realtime_qos | u32 | LOW | used only when priority==1 (real-time): the u32 at the root `+0x454`. |
| `0x80c` | 4 | active_priority | u32 | HIGH | last-applied priority cache. |
| `0x810` | 4 | firmware_priority | u32 | HIGH | mapped firmware priority; written from the priority table indexed by driver priority. |
| `0x814` | 4 | priority_param | u32 | HIGH | secondary priority value pushed to each work-queue's vtable slot `+0x1d0`. |
| `0x5e8` | 8 | submit_seq_counter | u64 | HIGH | global-per-queue-root monotonically increasing submission id (post-increment). |
| `0x5b0` | 8 | resource_pool | ptr | HIGH | → shared-memory allocator / stamp-address translator (vtable slot `+0x2d8` returns stamp GPU addrs). |

### 5.1 Priority mapping table (u32 LE)
`driver priority index → firmware priority`: `[0]=0, [1]=2, [2]=3, [3]=0, [4]=1`.
Only indices 0..5 are legal (`0x37 >> idx & 1` gate; note bit 2 is *clear* → index 2
handled specially). Three human-readable priority tiers are defined —
real-time, high, and default. HIGH.

### 5.2 User work-descriptor field layout
A serialized field-order descriptor for the *host-API* work descriptor lives in a
runtime type-encoding blob. Decoding it gives the field order below (this is the
host-side input the driver turns into the work items in §3 — I name it
`user_command_descriptor`). Types are from the encoding; names are mine.

| # | name | type | conf | meaning |
|---|------|------|------|---------|
| 0 | callback_table | ptr-to-ptr | LOW | function/pointer table. |
| 1 | arg_count | i32 | LOW | argument count. |
| 2 | gpu_address_base | u64 | LOW | base GPU address. |
| 3 | flags_a | u32 | LOW | flags word. |
| 4 | object_array | ptr array | LOW | array of referenced runtime objects. |
| 5,6 | dim_w, dim_h | i32,i32 | LOW | two ints (width/height, inferred). |
| 7 | memory_array | ptr array | HIGH | array of GPU-memory references. |
| 8,9 | mem_a, mem_b | i32,i32 | LOW | two ints associated with the memory array. |
| 10 | wait_fence_list | struct{head ptr, tail ptr} | HIGH | intrusive fence list (head + tail pointers). |
| 11 | fence_count | i32 | LOW | number of fences. |
| 12,13 | needs_prepare, is_protected | bool,bool | HIGH | two flags: pre-prepare required; protected/secure workload. |
| 14 | work_queue | ptr | HIGH | the scheduler work-queue this descriptor targets. |
| 15 | owning_task | ptr | HIGH | owning client task. |
| 16 | command_queue | ptr | HIGH | the per-context command-queue object (the §5 struct). |
| 17 | completion_fence | ptr | HIGH | block/completion fence object. |
| 18 | submit_flags | u64 | LOW | submission flag bits. |
| 19,20 | priority, qos | u32,u32 | HIGH | scheduler priority + QoS pair. |
| 21 | signal_event | struct{u32 value, u32[15] pad} | HIGH | event to signal on completion (1 value word + 15-word body). |
| 22 | wait_event | struct{u32 value, u32[15] pad} | HIGH | event to wait on before start (same shape). |
| 23,24 | signal_value, wait_value | u64,u64 | HIGH | 64-bit signal / wait target values. |
| 25 | scratch_ptr_array | ptr-to-ptr | LOW | void** scratch. |
| 26,27,28 | misc_a, misc_b, misc_c | u32,i32,i32 | LOW | three trailing ints. |
| 29 | completion_stamp | u64 | HIGH | trailing 64-bit completion stamp. |

Confidence: field presence/order is HIGH (from the encoding blob); individual names
are LOW where the encoding gives only a primitive type. Key HIGH takeaways: the
descriptor embeds a **command_queue pointer**, a **work_queue pointer**, a
**priority + qos pair**, two **event** blocks (each = 1 u32 value + 15×u32 body),
paired **signal/wait 64-bit values**, and a trailing 64-bit **completion stamp**.

--------------------------------------------------------------------------------
## 6. Ordering / correctness invariants observed (HIGH)
- Payload store → a data-memory barrier (DMB ISH) → index publish, in both the
  channel pointer-ring primitive and the service-ring core. The consumer
  (`peek_next_entry`) also DMBs before writing back its read index.
- All ring indices are kept modulo capacity via the recurring
  `x - (x/cap)*cap` pattern (division-based modulo), never a mask — so capacities need
  not be powers of two.
- Rings full-check spins/early-returns rather than overwriting.

--------------------------------------------------------------------------------
## 7. Gaps / LOW areas
- The bulk pipeline-state bodies of the three work items (hundreds of u64s copied from
  the queue object) are documented as "register/state image" blocks; individual field
  semantics are LOW and would need the firmware-side consumer to resolve.
- The exact bitfields of the `0x8593xxxx` data-master/notify mailbox opcodes are LOW
  (only the constants and their DMB/publish surroundings are HIGH).
- The full command-opcode enum beyond the 10 values in §4.5 is not named; the
  ring-select and power-gate tables (§4.3) prove ~58 opcodes exist but only
  those with observed literal writes or firmware names are named.
- `validate_type` is a very large per-ring validator (7 rings × the §1.1 header, at the
  base `+0x600`/`0x630`/.../`0x6f0`, plus two sub-rings at `+0x710`/`+0x740` and a tail ring) —
  confirms **7 primary rings + extras** in the validator's world, but its per-ring
  semantics beyond the read/write/capacity triples are LOW.
