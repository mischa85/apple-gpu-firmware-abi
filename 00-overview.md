# Apple GPU Firmware ABI — Clean-Room Documentation

**Target:** MacBook Neo — SoC `t8140`, GPU generation "G17P".
**Sources analyzed:** the host GPU driver stack and the GPU coprocessor firmware image, both for
the same GPU (SoC t8140, generation G17P). All names in this document are **inferred from observed behavior**; no
vendor-internal symbol names are used (clean-room).

> Scope of this document set:
> - `00-overview.md` — architecture, the running index, terminology.
> - `01-transport-and-endpoints.md` — the coprocessor message transport (mailbox) and endpoint catalog.
> - `02-initdata-and-boot.md` — firmware boot and the initialization/configuration structure tree.
> - `03-channels-and-submission.md` — host→GPU command-submission ring channels and work structures.
> - `04-events-recovery-memory.md` — GPU→host event ring, hang/recovery, firmware-driven memory mgmt.
> - `05-interrupts-and-power.md` — the outbox interrupt path (raise/ack/clear), the interrupt→event-ring
>   dispatch join, and the GPU power/idle/sleep/wake handshake.

## 1. Hardware/software model

The GPU is driven by a dedicated **on-die coprocessor** (an application processor core reserved for GPU
control) that runs a small real-time firmware ("coprocessor firmware"). The main-CPU host driver does **not**
poke GPU execution registers directly for normal work; instead it:

1. Boots the coprocessor firmware (the firmware image is pre-placed in memory by the boot loader; the
   driver maps it and releases the coprocessor from reset).
2. Builds a large **initialization/configuration structure tree** in memory shared with the coprocessor,
   and hands the coprocessor the base address of that tree.
3. Communicates thereafter over two mechanisms working together:
   - a hardware **mailbox** ("doorbell") that carries small fixed-size messages, multiplexed into logical
     **endpoints**; and
   - **ring buffers in shared memory** ("channels") that carry the bulk data — command submissions from
     host to firmware, and events/completions from firmware to host. Mailbox messages are largely used as
     doorbells announcing "new entries are in ring X".

This is the same coprocessor-RTOS pattern Apple uses for several other on-die agents; the GPU is one
consumer of a generic transport (documented in `01`).

## 2. The two images

- **Host GPU driver stack**: a layered set of driver modules —
  a generic coprocessor/RTOS transport layer, a generic GPU family layer, a mobile-graphics family layer,
  and the **GPU-generation-specific driver** for G17P (which contains the concrete firmware ABI: the
  init-data builder, the channel/ring writers, and the firmware-event handlers).
- **Coprocessor firmware** (separate image, per GPU generation): an RTOS Mach-O that runs on the GPU
  coprocessor. It is delivered in a container holding several role/variant images. It is the "other end"
  of every structure documented here.

## 3. Terminology used in this document (clean-room names)

| This doc's term | What it denotes |
|---|---|
| coprocessor / firmware | the GPU control processor and the RTOS running on it |
| mailbox message | the fixed-width hardware doorbell message (see `01`) |
| endpoint | a logical channel of mailbox messages, addressed by a small integer id |
| init-data / global config | the configuration structure tree handed to firmware at boot (`02`) |
| channel / ring | a shared-memory ring buffer of fixed-size entries (`03`, `04`) |
| submission channel | host→firmware ring carrying rendering/compute/tiling work (`03`) |
| device-control ring | host→firmware ring carrying control commands (`03`) |
| event ring | firmware→host ring carrying completions/notifications (`04`) |
| stamp | a monotonic completion counter written to shared memory to signal progress |
| context | a GPU address-space / client submission context |

## 4. Running index (coverage tracker)

Legend: ⏳ in progress, ✅ documented, — n/a. This index is the auditable coverage list; it is
reconciled against the firmware-interface type inventory (110 firmware-interface identifiers).

### 4.1 Endpoints (transport) — see `01`
- Management endpoint: version-negotiate + endpoint-roll-call/map + ping/power-ack handshake ✅
- Crash-dump ✅, text-log ✅, structured firmware-log ✅, host-trace-bridge ✅, cpu-trace-stream ✅,
  io-reporting/telemetry ✅, entropy-request ✅, analytics-telemetry ✅, binary-log demux ✅
- GPU app endpoints (`gpu_app_endpoint_1`/`_2`: init-data handoff + submission/scheduler doorbells) ✅
- Secure endpoint proxy (exclave bridge) ✅

### 4.2 Structures — see `02`/`03`/`04`
Boot/init (`02`): firmware image layout & boot-config block; the global-config tree; the memory-mapping /
register-region array (24-byte entries, 44 regions, generic categories); DVFS/perf-state tables (primary
+ auxiliary + frequency arrays); the feedback power-controller shared-data block; firmware-role selection. ✅

Submission (`03`): channel/ring header; command-pointer ring entry; vertex/fragment/compute work structs;
command-queue struct; device-control-ring & data-master-ring entries; command-opcode enum; command
scheduler state; timestamp queue; secure-submission buffer info; the doorbell word encoding. ✅

Events/recovery/memory (`04`): firmware→host event-ring entry + event-type enum (15 values, `0x00`–`0x0E`);
channel-error/fault args; hardware-error/recovery record; stamp-update; shared-event/fence signal;
process-exit; UMA pool-grow request/response; page-management memory request; memory-descriptor entry;
plus a second (statistics/telemetry) event namespace (`0x10`–`0x1E`). Scheduler-status and firmware-initiated
recovery-info structs are not recovered in this pass (documented as gaps). ✅

### 4.3 Interrupts & power — see `05`
Interrupt delivery (`05`): the outbox-not-empty interrupt → vtable-installed receive pump → bulk outbox
getter (status consume/ack) → endpoint lookup/dispatch → GPU app-endpoint callback → gated work-loop →
event-ring drain (the join between `01` and `04`); the two-layer acknowledgment (hardware outbox ack vs
ring credit-return). The coprocessor-side offsets ARE recovered in `05` §8: the software-interrupt
set/clear/status offsets, the per-die idle bit, the deep-sleep-enable sysreg bit, the link-kick op-code,
and the power-on/wake MMIO sequence are HIGH. Only the *absolute boot-injected region bases* and the
*host-side outbox/inbox control offsets* remain GAP. ✅

Power/idle/wake (`05`): the control-endpoint power-state message class scheme; power-notification
send with defer/flush; coprocessor run-state values; the idle-off quiesce loop; the sleep-control /
power-interrupt accessor roles; and the **wake** handshake (inbox-doorbell kick raising the coprocessor
mailbox interrupt to un-park it from wait-for-interrupt idle), plus the deep-off doorbell/interrupt
regions. Power-ack word bitfields and the power-processor-doorbell encoding are gaps. ✅

## 5. Confidence & honesty notes
- Field confidence is tagged HIGH / LOW in each section.
- Consumer note (blocking): the numeric endpoint ids are not recovered in this pass; endpoints in `01` are
  identified by behavior/name only, and indexing the endpoint table by id needs the numeric id constants.
- Where an ABI item's body is not observable in this build, it is marked as an explicit GAP rather than
  guessed. These are noted in `02` (init-data handoff site),
  `04` (scheduler-status decode, firmware-recovery-info, GART-invalidate / context-base message encodings).
- Cross-checks that raise confidence: the host→GPU doorbell word encoding is independently derived in both
  `02` and `03` and agrees; the firmware→host event enum in `04` is taken from the actual dispatch switch
  (selector == discriminant); the coprocessor firmware image corroborates field meaning on the "other
  consumer" side.
