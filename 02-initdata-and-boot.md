# GPU Firmware Boot & Init-Data / Global-Config Tree

Scope: GPU firmware image boot layout, the boot-config block the driver fills in,
and the large initialization/global-configuration structure tree the driver builds
in shared memory and hands to the firmware at startup. Plus the endpoint over which
the init-data base address is delivered.

Confidence is HIGH when the meaning is unambiguous from a matching human-readable
configuration key or a firmware-side field group; LOW otherwise. All struct/field/endpoint
names below are neutral names chosen for this document; vendor identifiers and verbatim
image/configuration strings are deliberately not reproduced.

---

## 1. Firmware image / boot layout

### 1.1 The firmware descriptor block (passed to the boot handshake)

The host driver receives a firmware descriptor (an array of 8-byte words) and copies the first
`0x9a` bytes into the device object `+0x88` (also duplicated verbatim in every boot-config accessor).

`fw_descriptor` (source layout, indices are into the passed array):

| offset | size | name                  | type | conf | meaning |
|------|------|-----------------------|------|------|---------|
| `0x00` | `0x20` | image_id / uuid_hi    | u8[] | LOW  | copied to the driver object `+0x88`..`+0xa0` |
| `0x20` | 4    | fw_kind               | u32  | HIGH | the u32 at the descriptor `+0x24`; switch values 1/3/4 select handshake path |
| `0x24` | 4    | (sub-kind / index)    | u32  | LOW  | the word at descriptor index 4 (byte `+0x20`), compared to `0x1e`/`0x1f` |
| ...  | ...  | payload words         | u64  | LOW  | copied to the driver object `+0xa8`..`+0x110` |
| `0x90` | 2    | trailer               | u16  | LOW  | copied to the driver object `+0x118` |

After the copy, the object holds two identity values that steer ALL boot-layout
decisions:

| offset (driver object) | name        | conf | meaning |
|---------|-------------|------|---------|
| `0xa8`   | gpu_gen     | HIGH | compared to `0x1f` and `0x1e` throughout the boot accessors |
| `0xac`   | gpu_variant | HIGH | compared to 1 / 3 / 4, selects sub-layout |

`gpu_gen`=`0x1f` and `gpu_gen`=`0x1e` are two GPU generations; `gpu_variant` (1/3/4) selects a
die/config sub-variant. This build targets a single machine, so at runtime one branch is taken.

### 1.2 Boot-layout accessors (offsets into the loaded firmware image)

These accessors all take `(this, role, out_err, fw_descriptor, flags)` and return an offset
or size into the firmware image, keyed by (`gpu_gen`,`gpu_variant`) and a `role`
argument (0 or 1 = two firmware roles, e.g. primary vs. auxiliary die).

| accessor | returns | value(s) | conf |
|----------|---------|----------|------|
| text section base offset accessor | text section base offset | role1→`0x178000`, role0→`0x180000` | HIGH |
| text section size accessor | text section size | computed from a 10-entry (base,size) table | HIGH |
| resume-flag offset accessor | FW resume-flag offset | role1→`0x178000`, role0→`0x180000` | HIGH |
| runtime image size accessor | runtime image size | role1→`0x178000`, role0→`0x180000` | HIGH |
| boot-config offset accessor | boot-config offset | returns `0x0` for the valid (gen,variant) pairs | HIGH |

Notes:
- The two numbers `0x178000` (gen `0x1f`) and `0x180000` (gen `0x1e`) are the runtime image
  size / the offset at which the runtime portion begins. Text base = runtime image
  size for these parts, i.e. the fixed header/loader precedes the runtime image and the
  runtime text begins at `0x178000`/`0x180000`.
- The text-section SIZE accessor reads a per-(gen,variant) table of ten `(base,size)`
  4-byte pairs and returns `(last.base − first.base) + last.size` — i.e. it sums a
  10-segment section map. HIGH that it is a 10-entry section table; LOW on the individual
  segment meanings.
- boot-config offset returns 0 for this machine: the boot-config block lives at the very
  start of the writable firmware region (offset 0), consistent with the driver writing the
  init-data pointer near the image base.

### 1.3 Boot sequence

1. Read a mode/resume flag. Bit0 selects cold-boot vs. resume path.
2. Look up the firmware image resource by name (a firmware-image name string and a
   host-driver module tag string).
3. Obtain the firmware image base and store it at the driver object `+0x30`. This base string is later used
   to compose endpoint names.
4. Cold-boot path: hand the coprocessor its firmware and start it (coprocessor start),
   passing two memory descriptors — these are the shared mailbox/ring buffers.
5. Endpoint registration (guarded by the driver object `+0x31` "already-registered" flag): compose two
   channel names from the base string (the driver object `+0x30`), each formed by appending a one-word
   suffix string:
   - suffix string → **app_endpoint_1**
   - suffix string → **app_endpoint_2**
   Register four handlers, stored at:
   - the driver object `+0x2a`, `+0x2b`  ← app_endpoint_1 (two handlers: a data handler and a second callback)
   - the driver object `+0x2c`, `+0x2d`  ← app_endpoint_2 (same two callbacks)
   Set the driver object `+0x31` = 1.
6. If FW was not loaded by the bootloader, a fatal log fires.

So there are **two app-level mailbox endpoints** past the built-in coprocessor endpoints,
which this document calls **app_endpoint_1** and **app_endpoint_2**.

---

## 2. The init-data / global-config struct tree

The config tree is assembled in two layers:

- **accelerator hardware-state object** (called here `hw_ctx`), pointed to by the
  builder's device object `+0x298`. It carries a large embedded config image at offsets
  `+0xf2a0 … +0xf5xx` (power/performance control) and `+0x19780` (a per-perf-state frequency
  table). Its vtable slot `+0x2d8` is a configuration-key reader used
  to fill config fields from device-configuration keys.
- **The shared init-data region** which contains the perf-state tables, the register-region
  mapping array, and pointers to the sub-structs.

### 2.1 config builder entry

The top-level config populator calls the power/perf-state seeder first, then reads
~200 device-configuration keys and writes them into `hw_ctx` and into sub-structs. Every
write is sourced via the configuration-key reader (vtable slot `+0x2d8`) → if present, store
the value at a fixed offset. Because each field is paired with a human-readable
configuration key, these are HIGH confidence.

The state-header fields written up front (in the seeder):

| offset (device object) | name                    | conf | meaning |
|------------|-------------------------|------|---------|
| `0xf34`     | max_perf_state_index    | HIGH | hw_ctx word `0x32ef`, minus 1 |
| `0xf38`     | dvfs_state_0            | HIGH | perf-state lookup(0), vtable slot `+0x11b8` |
| `0xf3c`     | dvfs_state_1            | HIGH | perf-state lookup(1) (if enabled bit at hw_ctx `+0x509`) |
| `0xf40`     | dvfs_state_2            | HIGH | perf-state lookup(2) (if enabled bit at hw_ctx `+0x50a`) |
| `0xf30`     | sched_quantum_default   | HIGH | default `0x28`, overridable via a scheduler-state key |

### 2.2 Power / performance controller config (embedded in `hw_ctx` at `+0xf2a0`…`+0xf5xx`)

This is the power and performance feedback-controller config (referred to here as the
`power_controller`). Firmware-side field groups confirm this field group. All HIGH unless
noted (each has a matching configuration key).

Power-limit PI controller:

| offset (hw_ctx) | name                       |
|--------------|----------------------------|
| `0xf2a0`      | device_max_power           |
| `0xf2a8`      | pwr_filter_time_constant   |
| `0xf2cc`      | pwr_integral_gain          |
| `0xf2d0`      | pwr_integral_min_clamp     |
| `0xf2d8`      | pwr_proportional_gain      |
| `0xf2d4`      | pwr_ctrl_target (copied from `+0xf2a0`) |
| `0xf2dc`      | pwr_ctrl_gain (neg): `-max_pstate*100 / device_max_power` |
| `0xf2e0`/e4/e8| pwr_ctrl_min/quantum (seeded from sched_quantum & max_pstate*100) |

Performance (utilization) PI controller, active when a valid base perf-state key is
present; if the base perf-state is invalid a fatal log fires:

| offset (hw_ctx) | name                          |
|--------------|-------------------------------|
| `0xf354`/48/4c/50 | perf base/target pstate scaling (derived from base-pstate key) |
| `0xf308`      | perf_target_utilization (minus deadzone `+0xf30c` key) |
| `0xf320`      | perf_filter_time_constant     |
| `0xf324`      | perf_filter_time_constant2    |
| `0xf328`      | perf_filter_drop_threshold    |
| `0xf330`      | perf_integral_gain            |
| `0xf334`      | perf_integral_gain2           |
| `0xf338`      | perf_integral_min_clamp       |
| `0xf340`      | perf_proportional_gain        |
| `0xf344`      | perf_proportional_gain2       |
| `0xf4d5`..f4f1| fast-adapt timers/steps (4 timers + 4 steps) |
| device object `+0xf88` | perf_limiter_filter_tc (default 5) |

Idle / wake / boost / fast-die / power-zone / self-throttle group. Representative subset
(all HIGH, one field per key), described by function rather than key literal:

| offset (device object or hw_ctx) | field (function) |
|----------------------|------------------|
| device object `+0x217c` | GPU idle-off delay, default 2 |
| device object `+0x2188` | idle-off delay for the power-sequencer block, default `0x28` |
| device object `+0x2194` | firmware early-wake timeout, default 5 |
| hw_ctx `+0xf310`   | perf-boost minimum utilization |
| hw_ctx `+0xf314`   | perf-boost step |
| hw_ctx `+0x19780` region | perf-reset iteration count |
| hw_ctx `+0xeec4`/1dd8/1ddb/eedc | fast-die-0 release-temp / prop-target-delta / proportional-gain / integral-gain |
| hw_ctx `+0xf12c`/f158/f14c | package-power filter-time-constant / Kp / Ki |
| hw_ctx `+0xf1e4`..f22c | self-throttle controller (12 keys) |
| hw_ctx `+0xf384`..f434 | fast-analog voltage/margin controller (enabled if hw_ctx word `0xda` bit0x22) |
| device object `+0x20e8`..`0x2138` | utilization-metering periods, ~24 keys |

Power-zone controller: 5 zones, each 3 keys (target / target-offset / filter-tc), written
into the fast-analog block at hw_ctx `+0xf384` onward.

The metering-period block (device object `+0x20e8`…`+0x2138`) is then *copied out* into another struct at
the device object `+0x390` starting `+0x04` — that is the firmware-facing utilization-metering config
sub-struct (HIGH: contiguous 11-word copy).

### 2.3 power_controller shared-data fields

The following accessor functions fetch device-configuration keys for the `power_controller`
shared-data block handed to firmware. The vendor key names are not reproduced; each field is
described by its function:

- power-readback client function (see §3, filtered GPU power readback)
- dynamic_split_ratio
- standby_count
- standby_duration
- power_sample_period
- perf_ctrl_target
- deadline_control_effort
- game-mode set
- budget-set client function

Confidence HIGH that these are the `power_controller` shared-data fields; LOW on exact struct
offsets (they are set through client-function indirection, not direct stores in the builder).

### 2.4 Sub-struct fillers (fixed-layout config blocks)

The builder calls three vtable methods on `hw_ctx` that fill fixed-layout config structs
that are then embedded in the init-data:

**SoC-thermal config** (via vtable slot `+0xd00`, dest = `soc_thermal_cfg`):

| offset (dest) | size | name              | conf | meaning |
|------------|------|-------------------|------|---------|
| `0x00`      | 8    | enabled_die_mask  | HIGH | `0xaa0` if gen==`0x1f` else `0x82a` |
| `0x08`      | 4    | temp_threshold_c  | HIGH | const `0x7d` (125) |
| `0x0c`..    | var  | per-die entry array (zeroed then indexed by set bits of the die mask) | HIGH | stride 4 per die index |

**Fast-die controller config** (via vtable slot `+0xd08`, dest = `fast_die_cfg`):

| offset (dest) | size | name              | conf | meaning |
|------------|------|-------------------|------|---------|
| `0x00`      | 8    | enabled_die_mask  | HIGH | same mask as above |
| `0x08`      | 4    | float_param (`0x42dc0000` = 110.0f) | HIGH | |
| `0x28`..    | var  | per-die entry array (zeroed, indexed by die-mask set bits) | HIGH | stride 4 |

**Power-estimation config** (via vtable slot `+0xd10`): reads GPU hardware-ID fields (config-region
fields at `+0x118/+0x120/+0x128`) and fills three float tables (at the destination
`+0xe50`/`+0xe90`/`+0xed0`/`+0xf10`): analog offset = `(field&0xfff+1)*0.2`, plant coeff =
`(field&0xfff)+1`, etc. HIGH that these are per-perf-state power-model coefficients; LOW on
exact array element mapping.

### 2.5 Perf-state (DVFS) tables and register-region mapping array

Built inside the shared allocator, which reads the perf-state config keys: number of
perf-states, table count, main perf-states, secondary/on-chip-memory perf-states,
compute-subsystem perf-states, fabric perf-states. A caller reads a per-pstate frequency
table at hw_ctx `+0x19780`, indexed by pstate*4, value in Hz (÷1000000 → MHz).

Firmware-side field-group names confirm the per-pstate entry fields: perf-state low,
perf-state high, perf-state select, graphics-core perf-state, power-boost controller. The
firmware bounds the count by its max perf-state count (16).

### 2.6 Perf-state (DVFS) table layout (detailed)

Four perf-state tables are parsed into the shared config block (base = config `+0x18db0`,
called `pstate_block` here):

| table (neutral name) | dest |
|----------------------|------|
| primary_pstates      | pstate_block (config `+0x18db0`) |
| onchip_mem_pstates   | pstate_block, parallel arrays |
| compute_pstates      | config `+0x1a808` |
| fabric_pstates       | config `+0x1a950` |

Count fields (HIGH):

| offset (config) | name              | conf | meaning |
|--------------|-------------------|------|---------|
| `0x18de0`     | num_perf_states+1 | HIGH | number-of-perf-states key, stored at pstate_block `+0x30` |
| `0x18f34`     | perf_table_count  | HIGH | table-count key, mirrored to shared header |

Validation: `perf_table_count < 3` and equals the value at config `+0x4ec`; `num_perf_states ≤ 0x10`
(16) — matches the firmware's max perf-state count.

**primary_pstate_entry** — stride **`0x30` (48 bytes)**, 12 × u32 per entry, copied from
the source blob (tight unrolled ×6 loop):

| offset | size | name        | type | conf | meaning |
|------|------|-------------|------|------|---------|
| `0x00`..`0x2c` | 4 each | field_0..field_11 | u32 | LOW | 12 per-state parameters (freq/volt/dvfs/margins). A forward-fill pass propagates the previous non-zero value into any zero slot. Individual field meanings not separable. |

For each primary/on-chip-memory entry the frequency is ALSO written to a flat parallel array:

| offset (config) | name             | stride | type | conf | meaning |
|--------------|------------------|--------|------|------|---------|
| `0x18f38`     | pstate_freq_hz[] | 4      | u32 (Hz) | HIGH | config `+0x18f38`, indexed by pstate*4 |
| `0x19780`     | pstate_freq_hz_table2[] | 4 | u32 (Hz) | HIGH | caller reads `/1000000`→MHz; sits `0x848` past `+0x18f38` (`0x848` = the shared perf-state payload block size) |

**aux_pstate_entry** (compute / fabric) — source entry = 16 bytes = two u64:

| offset | size | name        | type | conf | meaning |
|-----|------|-------------|------|------|---------|
| `0x0` | 8 | frequency | u64→u32 | HIGH | stored as `freq/1000` = **kHz** (note: different unit than the Hz arrays above) |
| `0x8` | 8 | state_value | u64→u32 | HIGH | raw voltage / dvfs index, stored verbatim |

Aux blob header: hdr word 0 low 32 bits = group count (`<3`, equals the value at config `+0x4ec`),
hdr word 1 = entries per group. The on-chip-memory table reserves **`0x40` (64 bytes) per
(table,pstate)** slot (bounds = config `+0x19338`, plus table_count*4, plus num_pstates*`0x40`).

The register-region mapping array entry format and the base-address handoff are in §2.7
and §3.

### 2.7 Register-region mapping array

This is the array whose entries correspond 1:1 to the hardware-register-region enum (44
entries; category breakdown in §4).

Array header (on the firmware object, the device object here):

| offset (device object) | size | name           | conf | meaning |
|------------|------|----------------|------|---------|
| `0x1a98`    | 8    | region_array_ptr | HIGH | base, indexed as base plus count*`0x18` |
| `0x1aa0`    | 2    | region_count   | HIGH | u16, incremented per insert |
| `0x1aa2`    | 2    | region_capacity| HIGH | u16, bounds-checked (overflow → panic) |

**region_mapping_entry** — stride **`0x18` (24 bytes)**:

| offset | size | name          | type | conf | meaning |
|------|------|---------------|------|------|---------|
| `0x00`| 8    | device_va     | u64  | HIGH | GPU/device virtual base = translate(input phys base) via mapper accessor vtable slot `+0x2d8` |
| `0x08`| 4    | size_pages    | u32  | HIGH | byte length ≫ 12 (# of 4 KB pages) |
| `0x0c`| 4    | reserved0     | u32  | HIGH | written 0 |
| `0x10`| 2    | core_mask     | u16  | HIGH | `1 << (K − core_index)`; core_index from vtable slot `+0x358` — selects GPU cluster/die |
| `0x12`| 2    | flags         | u16  | HIGH | cache/prot mode {1 or 3} OR (extra ? 4 : 0); mode=3 default, =1 outside a special high-address window (gated by the device object byte `+0x2a9`) |
| `0x14`| 4    | reserved1     | u32  | HIGH | written 0 |

The physical MMIO bases fed into this array are programmed into the accelerator object by
a dedicated MMIO-base programmer (e.g. `+0x286→0x220104000` sz `0x400000`,
`+0x745→0x40165c000`, `+0x853→0x300280000` sz `0x2000`, `+0x5b0→0x3003d0000` sz `0x1000`,
`+0x637→0x3003c0000`). Each region enum member (§4) is one such (base,size) pair mapped into
a 24-byte entry. HIGH on entry format; LOW on the exact enum-index → base mapping (the code
programs bases by field offset, not by iterating the named enum).

---

## 3. Init-data base-address handoff to firmware

### 3.1 Where the config struct lives

The global-config / init-data struct is embedded **inside the accelerator device object**,
reached everywhere as the value at fw_obj `+0x298`. All §2 config fields are written at offsets within
this object (e.g. `+0xecc0`, `+0xf2a0…`, `+0x18db0` perf-state block, `+0x19780` freq table,
`+0x1a98` region-mapping array). The device virtual address of this struct is what firmware
needs. Firmware refuses to run if the host-mapped path is disabled — the firmware log at that
site formats a 64-bit init-data-address log field, confirming the firmware-side field is a 64-bit
address.

### 3.2 The mailbox / doorbell channels

Two ring/mailbox endpoint objects hang off the firmware object:

| offset (fw_obj) | name        | conf | meaning |
|--------------|-------------|------|---------|
| `0x19d8`      | mbox_ep_a   | HIGH | used by the doorbell send (vtable slot `+0x8a8`) |
| `0x1a10`      | mbox_ep_b   | HIGH | sibling; both polled for readiness (vtable slot `+0x888`) |

These correspond to the two app endpoints registered by name in the boot sequence:
app_endpoint_1 and app_endpoint_2, whose handler pairs are stored at fw_obj words `0x2a`..`0x2d`.

### 3.3 Mailbox message encoding (doorbell / kick)

The 64-bit message word:

```
 bits [63:48] = msg_type   : 0x83 (normal) or 0x87 (when selector arg != 0)
 bits  [4:2]  = channel    : (arg & 7) << 2   (0..7)
 bits  [1:0]  = sub_type   : 0 / 1 / 2  (OR'd in at specific call sites)
```

Second mailbox word = 0. Sent through `mbox_ep_a` (fw_obj `+0x19d8`) via vtable slot `+0x8a8`
(authenticated indirect branch). Confidence HIGH on the bitfield layout and that this
doorbell rides mbox_ep_a.

### 3.4 GAP — exact publish site of the init_data device address (LOW)

The doorbell encoding above is confirmed, but the precise site that publishes the 64-bit
init_data device address to firmware was NOT positively identified. Two hypotheses, in order
of likelihood:

1. (Most likely) The init_data device address is written into a fixed slot of the shared
   boot/config region (the struct is mapped; its device VA is placed where firmware reads it
   at the handshake), and firmware is then told to consume it via the `0x83/0x87` doorbell on
   mbox_ep_a. This matches the firmware init-data-address log field being a read of a struct
   field rather than a decoded mailbox word.
2. The address is packed into a mailbox word — NOT observed; the per-context base-address
   submit routine is inlined/stubbed with no body, and no site was found ORing the init_data
   pointer into a message word.

To close: inspect the setter of fw_obj `+0x19d8`/`+0x1a10` and the config-builder's final stores
into the device object `+0x380`/`+0x390` and the device-mapping primitive for the slot that
receives the struct's device VA.

---

## 4. Hardware-register-region enum (region-mapping array contents)

The register-mapping array (§2.7) has **44 entries**, one per hardware MMIO register block
the firmware is granted access to. The individual vendor block names are not reproduced; the
approximate functional category breakdown (from the firmware-side region-type table) is:

| category | approx count | role |
|----------|--------------|------|
| clock / DVFS control | ~8 | fabric and cluster clock generators and DVFS control MMIO banks |
| graphics-cluster (multi-cluster) | ~3 | per-cluster config/clock MMIO banks |
| power-sequencer block | ~10 | the power/config sequencer MMIO banks (config, scratch, power-transition, etc.) |
| power-manager | ~9 | power-manager MMIO banks, per-die misc, scratch, idle-agent MMIO |
| thermal / metrology / telemetry | ~7 | temperature sensors, throttle-stat counters, telemetry dashboard spaces |
| interrupt controller | ~2 | banked interrupt-controller MMIO plus software-interrupt path |
| doorbell | ~1 | power-management-processor doorbell |
| memory / cache | ~4 | cache MMIO banks, memory-XU, on-die interconnect MMIO |

Confidence HIGH on the total count (44) and the 24-byte entry format (§2.7); the exact
per-member array ordering and the mapping from category to array index were not determined
(LOW).
