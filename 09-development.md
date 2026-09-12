# 9. Development

Everything needed to build, flash, run, and observe the firmware on the first day.

---

## 9.1 Repositories

| Repo | Purpose |
|---|---|
| `3cat8-obc` | OBC flight firmware — the main work (branch `Yasser`, committed at `a7ad9e0b`) |
| `3cat8-operations-api` | Ground REST API (Python FastAPI), branch `main` |
| `3cat8-gs` | Ground tools — trace decoder, scheduler builder/bench scripts, file manager |
| `3cat8-obc-stm32` | Legacy bare-metal reference — **read only, never modify** |

The firmware source is under `espf/` (`app/` conops and init, `arch/` STM32 HAL and drivers, `config/` module and service configuration, `core/` libraries and services); code-generation and build scripts are under `build/` and `other/`.

## 9.2 Build

```bash
cd 3cat8-obc/build
python3 macaron.py -b noboot_debug -c -rl 2>&1 | tail -5
```

Always use `-c` (clean build) — incremental builds hit a known Ninja dependency cycle. `noboot_debug` is the development target. Module enables are set as CMake options in `build/CMakeLists.txt` (`ADD_OPTION(...)`); the ones that matter here: `CUBEADCS_GEN2_ENABLED`, `GNSS_ENABLED`, `MICROPYTHON_SERVICE_ENABLED`, `COMM_GW_ENABLED`, `ARDUCAM_ENABLED` are ON; `SLIP/LWIP/ETHERNET/TFTPD_SUPPORT`, `SDR_ENABLED`, `XBAND_FE_ENABLED`, and `DEBUG_UART4_TX_TEST_ENABLED` are OFF.

**Build-flag gotcha.** The build system propagates `ADD_OPTION` flags to the compiler (`-D<FLAG>=ON` reaches the code) and links the debug library correctly despite a CMake reserved-keyword pitfall that can otherwise drop it. **If a newly-added guarded feature appears compiled out despite its flag being ON, check these two mechanisms first** — a guarded feature silently compiling out is the failure mode they prevent.

**Memory budget** (noboot_debug, July 2026): FLASH ≈ 88.9%, RAM_D1 ≈ 88.5% — both near 89% with several drivers still incomplete, so monitor headroom as new drivers land.

The committed HEAD `a7ad9e0b` boots clean. A known boot regression exists in a debug-flag build variant, so any bench work that flashes a debug variant should confirm a clean boot trace first; the committed HEAD is the clean reference.

## 9.3 Flash

Press F5 in VS Code (Cortex-Debug), or:

```bash
STM32_Programmer_CLI -c port=SWD -w build/noboot_debug/3cat8-obc.bin 0x08000000
```

## 9.4 Trace

```bash
cd 3cat8-obc/other/scripts/trace_decode/
python3 trace_decode.py -p /dev/serial/by-id/usb-FTDI_TTL232R-3V3_FTHATVLV-if00-port0
```

Always use the `by-id` path — the `ttyUSBn` index shuffles with plug order. Traces appear on UART5 (H1:39/40) via the custom FTDI cable; trace level 0 = DEBUG (most verbose), which is where `eps_m` logs.

## 9.5 Operations API and GOSH

```bash
# Operations API (Swagger at http://127.0.0.1:8000/swagger/)
cd 3cat8-operations-api && source venv/bin/activate && uvicorn app.main:app --reload

# GOSH — the P60 console (CSP node 4, P2 connector)
minicom -D /dev/serial/by-id/usb-FTDI_TTL232RG-VREG3V3_FT5ZNAHP-if00-port0 -b 500000
```

GOSH settings: 500000 baud, 8N1, hardware **and** software flow control **off**. Useful checks: `rparam getall`, `rparam get vbat_v` (should read 12000–17000 mV), `cmp ident 4` (identifies the Dock).

## 9.6 Rules to work by

1. Always clean-build (`-c`).
2. Call `drv_iwdg_refresh()` in every new background-task loop, **and** register the task with `taskmon` (see [§8](08-system-findings.md)) — the direct refresh alone gives no supervision.
3. Never edit the generated regions of the auto-generated ConOps files — only the user-code regions.
4. The `pl_` prefix is for mission payloads only, not platform subsystems (power, attitude, TT&C).
5. For a new bus device, follow the verification method proven on the power subsystem: implement, review against the authoritative source before connecting hardware, map the live device empirically, cross-check against an independent reference ([§2](02-power-subsystem.md)).
6. Never command a P60 channel, or automate RITA power-on, before the channel checks in [§7](07-interfaces.md) are done.
