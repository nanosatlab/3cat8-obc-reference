# 0. Glossary

Terms, acronyms, and project-specific names used across this documentation. If a term in any section is unclear, look here first. Facts reflect the committed source at `a7ad9e0b`.

---

**aocs_cntrl** — The SDK attitude-control service. On 3Cat-8 it is a proxy that would command an external CubeADCS unit; with no attitude module active, it runs no control loop.

**ArduCam OV5642** — A camera module with a connector on the OBC. Enabled for deployment capture via `ARDUCAM_ENABLED=ON` (`build/CMakeLists.txt`); the service is `espf/core/services/arducam/` and its FP server is registered in the ESPS stack config (`espf/config/ESPLATFORM_NETWORK_STACK/inc/ESSA_StackConfigApps.h`). Not physically connected.

**beacons service** — The SDK service that periodically transmits a telemetry frame (sourced from DataCache) to the UHF radio for downlink. On 3Cat-8 it runs but its telemetry presets are unconfigured (§8).

**CAN (Controller Area Network)** — The differential serial bus (CANH H1:3 / CANL H1:1) carrying CSP to the P60, CubeADCS, S/X-band, and the 3Cat-Gea hub. Needs 2×120 Ω termination (60 Ω).

**COBS (Consistent Overhead Byte Stuffing)** — The byte-framing used in the scheduler's `.slog` output file; decoded on the ground to read what the scheduler ran.

**COL-01** — The project label for the UART4 single-peripheral contention: one MCU pin (PH13) drives both the RITA RS-485 output and the GNSS-R/RO TTL line, so the two cannot run at once. Resolution is an open hardware-assignment decision (§7).

**comm_gw (communication gateway)** — The SDK's reusable blocking request/response primitive (`comm_gw_send()`) for a reliable call over the bus. `ttc`'s reliable UHF dispatch (§4) is a bounded-retry wrapper around it.

**ConOps (Concept of Operations)** — The operating-mode state machine: `SAFE_NO_CTRL`, `SAFE_CTRL`, `IDLE`, `MISSION`. Implemented in an auto-generated SDK file; only the user-code regions may be edited.

**CP (Command Protocol) / FP (Function Protocol)** — Two EnduroSat command protocols. FP is the OBC's internal function-call dispatch (used by the scheduler, the Operations API, and inter-component calls); CP is a separate telecommand protocol. A scheduler slot can dispatch either, a CSP command, or a MicroPython script.

**CSP (CubeSat Space Protocol)** — The network-layer protocol for the CAN bus (think IP for CubeSats), implemented by `libcsp`. Each device has a node address; the OBC is node 1, the P60 Dock node 4. `eps_m` uses CSP **port 7** for its rparam reads.

**CubeADCS Gen2** — An external CubeSpace attitude computer. Its driver is compiled in (`CUBEADCS_GEN2_ENABLED=ON`), but the module is **inactive** in the activation blob, so no attitude task runs in the current build.

**CuPID / FZPA / SP+Monopoles** — The three deployables fired by `pl_deployments`: the CuPID PocketQube deployer, the Fresnel Zone Plate Antenna (for the GNSS radio-occultation experiment), and the solar-panel + monopole set.

**DataCache** — A passive shared-memory store. Drivers write telemetry into it; other components read. It never polls. Status codes: `INIT` (never written), `OK` (fresh), `TOUT` (stale). Never assume an entry is valid until its status reads `OK`.

**DE / nRE** — RS-485 transceiver direction signals. DE (driver enable, active-high, MCU pin PF13) enables the transmitter; nRE (active-low, PF14) the receiver.

**eps_ctrl** — The SDK service that owns the battery-voltage FDIR fault and EPS channel-control thresholds. Distinct from `eps_m`; the low-battery fault is owned here.

**eps_m** — The OBC driver for the GomSpace P60 power subsystem over CSP/CAN, using rparam. Always-on background service, polls every 10 s (§2).

**ESPS (EnduroSat Protocol Stack)** — EnduroSat's serial bus/protocol stack. The **ESPS MAC bus** (USART1, on UART1-RS485) is the multi-drop bus carrying the OBC (address 0x33), the UHF radio (0x11), EPS I, S-band, and the solar panels.

**es-sys-config** — EnduroSat ground tool (over ST-Link) for reading/writing the module-activation blob at flash `0x81C0000`.

**es-tftp** — EnduroSat's file-transfer protocol over CSP, intended (deferred) for S/X-band bulk science-data downlink. Not the standard IP TFTP daemon (which is not compiled in this build).

**FDIR (Fault Detection, Isolation, Recovery)** — The SDK fault service. A major-level fault auto-transitions the satellite to SAFE. Each fault is owned by exactly one agent.

**FIDL / FDEPL (Franca interface / deployment files)** — The interface-definition files the SDK's code generator consumes to produce command and DataCache code. To add a DataCache entry you edit the FIDL and regenerate; never hand-edit the generated files.

**FreeRTOS** — The real-time OS the SDK runs on (task scheduler, queues, mutexes, timing).

**GNSS-R/RO** — GNSS Reflectometry / Radio Occultation: the science payload (`pl_gnss_ro`, §6). Uses reflected/occulted GNSS signals for Earth observation.

**GOSH (GomSpace Operations Shell)** — A serial console to the P60 (500000 baud, 8N1, flow control off). Used as the independent reference to verify EPS telemetry (§2).

**H1 / H2** — The two 40-pin OBC header connectors; pins are referenced as H1:N / H2:N.

**hs_comms** — The MISSION-mode payload handler for X/S-band high-speed downlink (§4.4). Distinct from `ttc`.

**IWDG (Independent Watchdog)** — The STM32 hardware watchdog (30 s) that resets the chip if not refreshed. Refreshed by `taskmon` (§1).

**libcsp** — The open-source CSP network/routing/transport library used by all CSP/CAN communication on the OBC.

**LoRa** — One of the two TT&C radios (with UHF). Uses SPI5 (PF6–PF9), which conflicts with UART7 (VER-01). Not yet integrated.

**macaron.py** — The SDK build script (wraps CMake + Ninja). Always build clean (`-c`).

**MicroPython** — A microcontroller Python, enabled on the OBC for scheduler scripting. The on-board compiler is disabled; scripts are pre-compiled to `.mpy` on the ground (`mpy-cross`).

**module-activation blob (0x81C0000)** — A flash region, separate from the firmware, that sets which subsystem modules (EPS type, ADCS type) are active at boot. Persists across reflashes; written via `es-sys-config` or a full firmware-update bundle. No on-orbit single-bit activation command exists.

**NVM (Non-Volatile Memory)** — Persistent storage. Holds the ground-adjustable battery-safe threshold and the beacon telemetry presets.

**OBC (On-Board Computer)** — The EnduroSat OBC Type I (STM32H743, FreeRTOS, SDK 7.1.0) — the central computer of 3Cat-8.

**Operations API** — A ground-side REST API (FastAPI) exposing OBC commands/telemetry over HTTP (Swagger at `/swagger/`). See §10.

**OSS (Onboard Scheduler Service)** — The SDK scheduler that dispatches time-tagged slots from a `.sch` file at one-second precision (§3).

**P60 (GomSpace NanoPower P60)** — The power subsystem: a PDU and ACU aggregated by a Dock (CSP node 4). The EPS on 3Cat-8.

**payload_ctrl (payload controller)** — The SDK framework that starts/stops mission-payload handlers (by ground command or ConOps). The `pl_` prefix marks a payload-controller handler.

**pl_deployments / pl_payloads / pl_gnss_ro** — The payload-controller handlers for, respectively, the deployables (§5.2), the RITA science hub (§5.1), and the GNSS-R/RO payload (§6).

**PVT (Position, Velocity, Time)** — The navigation solution produced by the GNSS payload's B16 receiver.

**rparam (remote parameter protocol)** — The GomSpace protocol for reading/writing named parameters on its devices over CSP. Big-endian addressing; the reply echoes the requested 2-byte address. `eps_m` uses it for P60 housekeeping.

**RITA / 3Cat-Gea / MB-RITA** — The polarimetric-imaging payload. **RITA** is the science instrument; **3Cat-Gea** (a.k.a. **MB-RITA**) is its hub board. RITA is autonomous — it runs the experiment internally; the OBC powers, confirms, starts, and retrieves (§5).

**RS-485** — A differential serial standard. On 3Cat-8 it carries the RITA interface (UART4, H1:17/18), transceiver DE-gated on PF13.

**SAFE mode** — The emergency operating mode entered when battery voltage falls below the safe threshold. Payload handlers are emergency-stopped; background services keep running.

**.sch / .slog** — The scheduler's binary input (time-tagged slots) and output log files.

**sch_builder** — The ground-side tool that generates byte-compatible `.sch` files for the OBC scheduler (§3, §10).

**SDK** — Here, the EnduroSat SDK 7.1.0: the vendor foundation providing ConOps, the payload controller, the background-service model, DataCache, FDIR, and the FP/CSP command infrastructure.

**SLIP / lwIP** — A serial-line IP stack. **Disabled** in this build (`SLIP_SUPPORT_ENABLED=OFF`), so UART8 — the peripheral it would use — is free/unused.

**ST-Link / SWD** — The STM32 debug/programming interface (over USB) used to flash firmware and the activation blob.

**sys_instancer** — The SDK mechanism that instantiates and spawns background-service modules (e.g. `eps_m`) at startup, gated by the module-activation blob.

**taskmon** — The SDK task supervisor: refreshes the IWDG unconditionally at real-time priority, and software-resets a monitored task that stops checking in. The project's own services are not registered with it (§8).

**ttc** — The always-on TT&C background service (UHF, and LoRa when added). Runs in every mode, including SAFE. No `pl_` prefix — it is a platform service, not a payload handler (§4).

**TT&C (Telemetry, Tracking and Command)** — The radio subsystem for ground communication; on 3Cat-8, UHF and LoRa via `ttc`.

**UART4 / UART5 / UART7 / UART8** — STM32 serial peripherals. UART4 = the RITA/GNSS contention (COL-01); UART5 = bench trace output; UART7 = a GNSS relocation candidate (conflicts with SPI5/LoRa, VER-01); UART8 = free/unused (the SLIP peripheral, disabled).

**UHF (Ultra-High Frequency)** — One of the two TT&C radios. It sits on the **ESPS MAC bus** (USART1, ESPS address 0x11) — not a dedicated UART. Downlink is via the beacons service and uplink via the standard MAC dispatch, so no UHF-specific OBC firmware is required (§4).

**VER-01** — The project label for the UART7/SPI5 mutual-exclusion question: they share MCU pins and SPI5 carries LoRa, so moving GNSS-R/RO to UART7 would break LoRa. An open architectural decision (§7).

**Y-momentum** — A momentum-bias attitude state the ConOps sets on MISSION entry (`on_entry_Mission`), giving the satellite gyroscopic stiffness about one axis during mission operations.
