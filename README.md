# Apple GPU Firmware ABI — Clean-Room Documentation

Clean-room reverse-engineered documentation of the Apple GPU firmware ABI for the **MacBook Neo**
(SoC `t8140`, GPU generation `G17P`).

The GPU is controlled by an on-die coprocessor running small real-time firmware. The host driver talks
to it over (a) a hardware **mailbox** carrying small messages multiplexed into logical **endpoints**, and
(b) **ring buffers in shared memory** ("channels") carrying command submissions and event/completion
notifications. This document set specifies both sides of that interface.

## Contents
| File | Covers |
|------|--------|
| [`00-overview.md`](00-overview.md) | Architecture, terminology, the running coverage index, confidence/gap notes. |
| [`01-transport-and-endpoints.md`](01-transport-and-endpoints.md) | Mailbox message format, doorbell/control registers, endpoint table, startup handshake, full endpoint catalog. |
| [`02-initdata-and-boot.md`](02-initdata-and-boot.md) | Firmware image/boot layout and the initialization/global-config structure tree handed to firmware. |
| [`03-channels-and-submission.md`](03-channels-and-submission.md) | Host→GPU ring channels, ring headers, per-workload command structs, command queue, doorbell encoding. |
| [`04-events-recovery-memory.md`](04-events-recovery-memory.md) | GPU→host event ring + event enum, hang/recovery, firmware-driven memory management. |
| [`05-interrupts-and-power.md`](05-interrupts-and-power.md) | Outbox interrupt raise/ack/clear, the interrupt→event-ring dispatch path, and the GPU power/idle/sleep/wake handshake. |

## Scope
This set describes both peers of the interface: the host-driver side and the coprocessor-firmware
side. Structures are given named fields with byte offsets, sizes, and types; endpoints are given
behavior-derived names; enums list their values; and the mailbox, doorbell, and ring encodings are
laid out. It is written as an interface description — a consumer should be able to implement or
emulate the interface from it.

## Confidence
Every field/endpoint is tagged **HIGH** (unambiguous and corroborated across both peers of the
interface) or **LOW** (inferred). Anything that could not be determined is called out as an explicit
**GAP** rather than guessed.

## Clean-room statement
All identifiers in this documentation — every struct, field, endpoint, and enum name — were chosen by the
authors based solely on observed behavior. No vendor-internal symbol names, source filenames, or verbatim
strings from the images appear in the deliverable. Public hardware identifiers (`t8140`, `G17P`) are used as
given. The output was verified by an independent clean-room audit pass.
