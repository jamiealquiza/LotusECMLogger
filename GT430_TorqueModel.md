# GT430 (C132E0278) Torque Model — Reverse-Engineering Notes

Notes for future tuning sessions on the Lotus T6e GT430 firmware (program `T6EVRGT430E01_BIN`, calibration `C132E0278`). The analysis disassembled the **USDM-patched GT430 program**; Donour's program differs from it only in the three patches listed below. Symbol names come from `LotusECMLogger/config/highSpeedLogger/database/C132E0278.csv` (Ghidra export). Items marked *(inferred)* were reasoned from names or data rather than traced instruction by instruction.

## Address conventions

| Item | Value |
|---|---|
| Calibration RAM base (`.cpt` offset 0) | `0x40008654` — `.cpt offset = address − 0x40008654` |
| Calibration size | 27,052 bytes (same as the calrom chunk in Lotus CRPs) |
| Small-data base `r13` | `0x40009430` (`lis r13,0x4000; ori r13,r13,0x9430` at flash `0x040524`) |
| Small-data base `r2` | `0x4000A8F8` (inside the calibration region) |
| Program flash base | `0x40000` (program file offset 0 = flash `0x40000`) |
| CPU / ISA | MPC5534, classic PowerPC Book E, big-endian (capstone `CS_ARCH_PPC`, `CS_MODE_32 \| CS_MODE_BIG_ENDIAN`) |
| Code reaches the calibration via | `lis rX,0x4001; addi rX,rX,-0x79ac` (= `0x40008654`), then `addi`/`lbz` with the `.cpt` offset |

Table lookup helpers: `0x061FC4` = 2D u8 lookup, `0x062624` = 2D u16 lookup, `0x061C94` = 1D lookup. Arguments: `r3`=nX, `r4`=nY, `r5`=X input, `r6`=Y input, `r7`=table, `r8`=X axis, `r9`=Y axis. 1D: `r3`=n, `r4`=input, `r5`=table, `r6`=axis.

## Pipeline overview

```
            ┌──────────────── ESTIMATE (0x0BBEB0) ─────────────────┐
MAF load ──►│ CAL_torque_load_to_torque(rpm, load) = gross torque  │
            │ × spark efficiency (retard from MBT)                 │
            │ × firing cylinders / 6       = obd_ii_engine_torque  │
            │ + friction (negative, 0x0BD268)  = net crank torque  │
            └──────────────────────────────────────────────────────┘

            ┌──────────────── LIMIT (0x0AB0A0 → 0x0BD570) ─────────┐
gear,rpm ──►│ torque_limit = min(RPM table, coolant table) × scale │
coolant  ──►│ if limit ≥ 510 Nm (USDM) / 911 Nm (Donour): no limit │
            │ gross allowed = (limit − friction + retard loss)     │
            │                 × 6 / (6 − failed cylinders)         │
            │ → CAL_torque_torque_to_load(rpm, gross) = load limit │
            │ → PI on (load limit − MAF load) + feed-forward       │
            │ → tps_target_from_torque_limit → clamps throttle     │
            └──────────────────────────────────────────────────────┘
```

Traction/ESP/TCU/DPM/cruise requests take a parallel path in `0x0BBEB0`/`0x0BD570`: arbitrated, converted with `CAL_torque_torque_to_load`, air-density corrected (×20000/density ratio), then `CAL_load_to_tps_mapping` → throttle, plus ignition retard (`0x04F590`) and cylinder cut (`0x04889C`) for fast cuts.

## Key functions

| Flash addr | Role |
|---|---|
| `0x0AB0A0` | Throttle limiting: looks up the RPM and coolant limit tables → `torque_limit`; calls `0x0BD570`; sets `tps_torque_limit_applied` |
| `0x0BD570` | Torque limiter: limit → gross torque → load limit → proportional-integral controller → throttle target. Also arbitrates external limits |
| `0x0BBEB0` | Torque estimation, external requests, spark-efficiency loss, `load_target_with_torque_limit` |
| `0x0BD268` | Friction sum: −(RPM×load table + coolant×load table + A/C) |
| `0x0CB334` | DPM traction control (uses `torque_net_engine_combustion`) |
| `0x04F590` | Ignition with torque retard |
| `0x04889C` | Cylinder cut (`torque_cut_level`) |

## Calibration tables (`.cpt` offsets)

| Table | Offset | Format / axes |
|---|---|---|
| `CAL_torque_limit_by_rpm` | `0x2966` | u8 [64] = 8 gear × 8 RPM. X gear `0x2956`, Y RPM `0x295E` (rpm = raw×31.25+500). **Same value in every gear in all known calibrations** |
| `CAL_torque_limit_by_coolant` | `0x16BE` | u8 [4]; X coolant `0x16BA` (°C = raw×5/8−40) |
| Scale of the two limit tables | — | **×2 Nm/count on stock/USDM; ×4 on Donour's program** (patched `slwi` at `0x0AB3EC` and `0x0AB424`) |
| `CAL_torque_load_to_torque` | `0x47A0` | u16 Nm [16×16]; X RPM u16 `0x4760`; Y load u16 mg/stroke `0x4780` |
| `CAL_torque_torque_to_load` | `0x3036` | u16 mg/stroke [16×16]; X RPM `0x2FF6`; Y torque Nm `0x3016` (top = 600 USDM, 690 Donour) |
| `CAL_torque_engine_friction_speed_component` | `0x1F3E` | u8 Nm [20×20]; X `engine_speed_8bit` axis `0x1F16`; Y `load_8bit` axis `0x1F2A`. Value is negated (friction is a loss) |
| `CAL_torque_engine_friction_temp_component` | `0x35C6` | u8 Nm [8×8]; X coolant `0x35B6`; Y load `0x35BE` |
| `CAL_torque_engine_ac_load_base` / `_scaler` | `0x0AFA` / `0x168A` | A/C load torque (×scaler/255) |
| `CAL_torque_spark_efficiency` | `0x3626` | u8 [16×16]; X RPM `0x3606`; Y retard from best timing `0x3616` |
| `CAL_torque_limit_proportional` / `_integral` | `0x07FC` / `0x2776` | Gains of the limiter's proportional-integral controller, by load error |
| `CAL_load_to_tps_mapping` | `0x1186` | u8 [20×20] throttle target by RPM × load |
| `CAL_revlimit_torque_limit_confirmation_threshold` | `0x0334` | u8 ×2 Nm |
| `CAL_torque_limit_cooldown_timer` / `_tps_rate_threshold` | `0x4674` / `0x4676` | |
| `CAL_inj_afr_base` (AFR target) | `0x1BCE` | u8 [20×20], AFR = raw/20+5; X RPM `0x1BA6`; Y load `0x1BBA` |
| `CAL_sensor_maf_scaling` | `0x22AE` | u16 [32] mg/s ×10; X voltage `0x226E` (×5/1023) |

## Hard-coded thresholds in the program

| Where | Value | Meaning |
|---|---|---|
| `0x0BD5A0` / `0x0BD59C` | `0x1FE` = **510 Nm** (USDM) → `0x38F` = 911 (Donour) | `torque_limit` ≥ this means **limiter disabled** (load limit = 0xFFFF). On USDM, a table value of 255 (510 Nm) switches the limiter off |
| `0x0BCCE4` | 911 Nm / −112 Nm | Clamp on the net torque estimate |
| several | `0x3FC` = 1020 Nm | Clamp on external and gross torque values |
| `0x0BC2C4` | `0x564` | Clamp on the load from `torque_to_load` (before density correction) |
| `0x0BC314` | `0x495` = 1173 | Load target saturates; scaled ×255/1173 into `load_target_with_torque_limit` |

## Useful RAM channels (for High-Speed Log)

| Symbol | Address | Notes |
|---|---|---|
| `torque_limit` | `0x4000249E` | Final limit in Nm (min of RPM and coolant tables) |
| `torque_limit_by_rpm` | `0x400024D2` | |
| `torque_limit_engine_speed` | `0x400024D4` | Actually the **coolant** limit result (catalog name is misleading) |
| `tps_torque_limit_applied` | `0x400024CC` | 1 = limiter is cutting throttle |
| `tps_target_from_torque_limit` | `0x4000156C` | |
| `torque_limit_net_friction_and_ign_retard` | `0x400026BA` | Gross torque allowed |
| `load_limit_from_torque_limit` | `0x40002696` | |
| `torque_limit_load_error` | `0x400026BE` | Load limit − MAF load |
| `obd_ii_engine_torque` | `0x40002650` | Gross × spark efficiency × cylinders |
| `torque_estimated_actual_load` | `0x40002652` | Gross from the load-to-torque table |
| `engine_friction_torque` | `0x40002654` | Negative |
| `torque_loss_from_ign_retard` | `0x40002698` | |
| `torque_net_engine_combustion` | `0x4000266C` | Used by DPM traction control |
| `load_measured_maf_filtered` | `0x4000187A` | What the limiter compares against |

## Tuning implications

- **Friction is added to the limit.** Gross torque allowed = limit + |friction|, so the friction table raises or lowers the airflow cap. At ~6000 rpm, full load: USDM 460+104 ≈ 564; Donour 520+52 ≈ 572; 2.6" pulley tune 504+104 ≈ **608** Nm gross. The pulley tune's "lower" limit allows the most air.
- **Friction also sets the reported torque.** The net estimate (`torque_net_engine_combustion`) feeds DPM traction control, IPS/traction power limits and misfire detection. Lowering friction raises the estimate.
- **The friction table stands for all parasitic loss, including the supercharger drive.** A smaller pulley raises real loss; the ECU only knows RPM × load, so pulley changes need friction recalibration *(inferred)*.
- **The limiter enforces the limit through MAF load,** so MAF scaling errors move the real torque ceiling: a MAF reading low lets real torque exceed the limit.
- **Ignition-retard torque loss is added back** when converting the limit, so knock retard doesn't cause extra throttle closing.
- **Never mix calibrations across programs:** Donour/pulley calibrations (4 Nm/count) on the USDM program halve every limit; USDM calibrations on Donour's program double them.

## Known files (SHA-256 prefix)

| File | SHA-256 | Contents |
|---|---|---|
| GT430 migration `C132E0278.CRP` | `ca01548b` | Program + calibration `20260102`. Forces the model identity on every start-up (validity check at `0x0C202C` overridden) |
| GT430 USDM patches `C132E0278.CRP` | `9e33bac9` | Migration + adjustable traction control (Race mode, cruise +/−, dash range field), P0463 range blanking removed, VIN 7th-character market fix, CAN byte fix, P0139 = 0 |
| Donour `C132E0278.CRP` | `e473d579` | USDM program + 3 patches (×4 limit scale, 911 ceiling) + performance calibration `20260307` |
| `C132E0278_TAB.20260502-93octane.cpt` (2.6" pulley) | `7a7d327c` | Donour calibration with limit 504 Nm, stock friction restored, slightly leaner top-load AFR. **Donour program only** |

CRP files hold a calrom chunk (ref `0x5` → flash `0x20000`, 27,052 bytes) and a program chunk (ref `0x4` → flash `0x40000`, 625,024 bytes). They are XTEA-CBC encrypted (T6 key in `CrpTools`), and EFI_PROT talks to the ECU at 500k on IDs `0x50`/`0x7A0` and `0x51`/`0x7A1`. `CrpCli`/`CrpTools` (net10.0) unpack them; a Python port of `CrpUnpacker.cs` also works.

## Other calibration facts

- **Unlock text:** `CAL_ecu_unlock_magic` at `.cpt` `0x69A6` = `WTF?` (enables RMA/live tuning/snapshots).
- **Fault-code enable bytes** (`u8_obd2level_t6`; **0 = disabled**): P0137 `0x4B70`, P0138 `0x4B71`, P0139 `0x4B72`, P0140 `0x4B73`, P0141 `0x4B74`, P0157–P0161 `0x4BF9`–`0x4BFD`, P0420 `0x4BB0`, P0430 `0x4BB5`, P0601 `0x4BE6`. On USDM, P0139–P0141 are already 0.
- **Coding block:** `COD_base` `0x40008060`; model field `0x40008080` (32 bytes, filled from `CAL_prog_version` at `.cpt` `0x59D4`); `coding_crc` `0x400080A0`. A mismatch there plausibly caused the P0601 seen after migration, cleared by Erase Model Info (target `C132E0278`, `coding_cmd` `0x40002770`).
- **Bank 1 downstream O2 voltage** `obd_ii_o2_sensor2_voltage` `0x4000184E` is sampled on a 0–5 V scale, so a linear wideband output can be logged through it with the High-Speed Log.

## Open questions

- What each adjustable traction-control level (1–6) maps to in the DPM slip tables.
- Whether the ECU interpolates linearly between `torque_limit_by_rpm` breakpoints (assumed).
- Exact market meanings of the VIN market codes in the USDM patch.
- The friction temperature and A/C components have not been compared across calibrations.
