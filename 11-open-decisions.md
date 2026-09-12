# 11. Open Decisions & Roadmap

The single list of what is unresolved and what to do next — the successor's starting point. Each detailed elsewhere; this gathers them.

> **Snapshot caveat.** This reflects the committed state at `a7ad9e0b` (July 2026). If work continued after that, some items may already be resolved — **re-check against the current code before acting.**

**How to read the tier tag on each item:**

- **[verified]** — the fact that this is open, and why, is confirmed in the source at `a7ad9e0b`. Trust it (subject to the snapshot caveat).
- **[recommended]** — a reasoned proposal from this work, *not* a settled decision. Treat it as "the previous engineer's recommendation, to confirm," not as decided.
- **[external]** — the answer must come from outside the code (a systems/supervisor decision, a launch-provider value, or the runtime module-activation configuration). The code only hints at it.

---

## 11.1 Decisions that block work

| # | Decision / blocker | What it blocks — and why | Tier | Resolved by | Ref |
|---|---|---|---|---|---|
| 1 | **COL-01 — UART peripheral assignment** | PH13 drives both the RITA RS-485 output and the GNSS TTL line, so they can't coexist. Blocks RITA relocation *and* the GNSS transport. Options: **A** (GNSS→UART7, RITA→UART6-RS485) or **C** (GNSS→UART6, RITA stays UART4, needs a payload-side transceiver). One confirming Saleae capture at 1.65 V is also outstanding. | verified | Systems / hardware assignment | [§7](07-interfaces.md) |
| 2 | **GNSS architecture fork** — background module vs single payload handler | Upstream of *all* other GNSS work (transport, registration, PVT). Recommendation: a single `pl_gnss_ro` handler now, with PVT ingestion addable later; the PVT→attitude feed is conditional on #7. | recommended | Systems / supervisor (recommendation made) | [§6](06-gnss-ro-payload.md) |
| 3 | **Deployer CSP node addresses** | Currently `0`; `pl_deployments` cannot address a real deployer node until assigned — the sequencer logic is built but non-functional without them. | verified | Systems / design decision | [§5](05-rita-payload.md), [§7](07-interfaces.md) |
| 4 | **Deployment-inhibit delay** *(flight-critical)* | `PL_DEPLOY_MIN_POSTLAUNCH_DELAY_MS` is `0U` (hold skipped entirely). The in-source target is **1,800,000 ms (30 min)**, but the authoritative figure is the launch provider's — confirm and set before any flight build. | external | Launch-provider mandate / supervisor | [§8](08-system-findings.md), [§7](07-interfaces.md) |
| 5 | **Task-supervisor (`taskmon`) policy** *(safety)* | The five project services aren't registered with `taskmon`, so a permanent hang is unrecovered. Naive registration would reset the OBC during the deployment-inhibit hold, so any fix must handle long-sleeping tasks. Decide whether/how to register. | verified | Whole-codebase policy (with supervisor) | [§8](08-system-findings.md) |
| 6 | **Beacon telemetry presets** *(mission)* | All presets are unconfigured, so beacons carry no health data. Define the per-mode content (or ship a safe minimum set as the NVM default) as a documented pre-launch step. | verified | Operations / commissioning | [§8](08-system-findings.md) |
| 7 | **Flight attitude-control approach** | The current build runs no attitude task (the module is inactive in the activation blob) — but this must **not** be assumed permanent. Also determines whether the GNSS PVT→attitude feed (#2) is real work. | external | Systems / supervisor | [§1](01-system-architecture.md), [§7](07-interfaces.md) |
| 8 | **Low-battery safing fix** *(safety)* | The owned low-battery fault is inert (reads an entry the active driver never writes). Two candidate fixes identified, neither chosen. Compounds with #6. | verified (gap) / recommended (fixes) | Systems | [§8](08-system-findings.md) |

## 11.2 Build work, unblocked or nearly so

| # | Work | State | Tier | Ref |
|---|---|---|---|---|
| 9 | **P60 channel commanding** | Write path built and address-confirmed, but never commanded. Before first use, verify the CH2/CH8 `out_on_boot` defaults and that CH0 is hardware-always-on. | verified | [§2](02-power-subsystem.md), [§7](07-interfaces.md) |
| 10 | **LoRa integration** | The one remaining radio item; interacts with the UART7/SPI5 mutual exclusion (VER-01). | verified | [§4](04-communications.md), [§7](07-interfaces.md) |
| 11 | **Wire a consumer for the reliable UHF dispatch** | The primitive is built with no caller; resolve the reused-FDIR-fault proportionality and the first-caller feasibility before wiring one. | verified | [§4](04-communications.md) |
| 12 | **GNSS DataCache PVT entry** | Buildable now (a new FIDL entry + regeneration); set its `Status_timeout` to the intended poll rate. | verified | [§6](06-gnss-ro-payload.md) |

---

The three system-level findings behind #4–#6 and #8 (the inert safing, the unmonitored services, the empty beacons — and how the first and third compound) are detailed in [§8](08-system-findings.md). Raise them together with the systems team.
