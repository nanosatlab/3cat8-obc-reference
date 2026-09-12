# 3Cat-8 OBC Flight Software — Technical Documentation

**Author:** Yasser Abdellaoui · UPC NanoSat Lab
**Scope:** the On-Board Computer flight-software work carried out during the internship, on top of the EnduroSat SDK 7.1.0.
**Verified against:** the committed source at `a7ad9e0b` (branch `Yasser`, `3cat8-obc`).

This is a straight-to-the-point technical reference: what was added to the 3Cat-8 OBC, how each part works, how it was designed where design was needed, and what state it is in. It describes the system as it stands in the code — not the history of how it got there.

---

## Platform

- **Mission:** 3Cat-8 — a 6U CubeSat (UPC NanoSat Lab): ionospheric science (GNSS reflectometry and radio occultation), multispectral auroral imaging, and a deployable-antenna technology demonstration.
- **OBC:** EnduroSat OBC Type I, STM32H743 (Cortex-M7, 480 MHz, 2 MB flash), FreeRTOS.
- **SDK:** EnduroSat SDK 7.1.0. The work ports the inherited mission logic onto the SDK's service and payload-controller model.
- **Ground interface:** the Operations API (a REST interface over the SDK command model).

---

## What was added — at a glance

| Area | What was done | State |
|---|---|---|
| **Power subsystem** (`eps_m`) | rparam-over-CSP driver for the GomSpace P60; live telemetry cross-checked against an independent reference; channel-control write path | **Hardware-verified** (telemetry); channel write implemented, address-confirmed, never commanded |
| **Architecture** | inherited tasks re-expressed as SDK background services vs payload handlers; platform/payload layer separation; safety model preserved and made explicit | **Software-verified** |
| **Onboard scheduler** | adopted the SDK scheduler; documented the `.sch` format against firmware; wrote a ground-side builder; proved single- and multi-slot execution on hardware | **Hardware-verified** |
| **Communications** (`ttc`) | established that UHF needs no new firmware (SDK beacons downlink + standard MAC uplink); designed and built a reliable OBC↔UHF command-dispatch primitive | **Implemented** (build-verified, boots); LoRa remains |
| **High-speed downlink** (`hs_comms`) | X/S-band mission-mode handler (the mission-mode counterpart to `ttc`); drives the MB-RITA-bus transceivers via CSP (the native SDK S/X-band handler is inert here) | **Skeleton** — X/S-band driver TODO |
| **RITA payload** (`pl_payloads`) | power-and-supervise sequence for the autonomous imaging payload (channel-ordered power on/off with rollback) | **Implemented**; activation/retrieval remain |
| **Deployment sequencer** (`pl_deployments`) | post-launch-inhibit → sequential deploy state machine (SP+Monopoles, FZPA, CuPID) | **Skeleton** — logic built; blocked on deployer node addresses |
| **GNSS-R/RO payload** (`pl_gnss_ro`) | command layer (14-command API) and wire-protocol layer built; transport/registration/PVT designed | **Skeleton** (built, not registered); transport designed |
| **Camera** | enabled the ArduCam OV5642 service for deployment capture | **Software-verified**; module not connected |
| **Operations API** (ground) | authored the payload-control and AOCS REST endpoints; hardened the camera endpoint's enum parsing | **Software-verified** ([§10](10-ground-side.md)) |
| **Scheduler tooling** (ground) | the `.sch` builder and the multi-slot bench test | **Hardware-verified** ([§10](10-ground-side.md)) |
| **System findings** | three flight-relevant gaps in the current system (two safety, one mission), with reasoning and candidate fixes | Documented ([§8](08-system-findings.md)) |

---

## Contents

0. [Glossary](00-glossary.md) — terms, acronyms, and project-specific names (read first if the SDK vocabulary is unfamiliar).
1. [System architecture](01-system-architecture.md) — task model, operating modes, the safety mechanisms, DataCache.
2. [Power subsystem](02-power-subsystem.md) — the `eps_m` driver, the P60 protocol, and the verification method.
3. [Onboard scheduler](03-onboard-scheduler.md) — the `.sch` format, the ground tool, and the bench results.
4. [Communications](04-communications.md) — the UHF paths and the reliable command-dispatch design.
5. [Mission payload handlers](05-rita-payload.md) — the RITA power-and-supervise orchestration, and the deployment sequencer.
6. [GNSS-R/RO payload](06-gnss-ro-payload.md) — the built layers, the architecture decision, and the wire protocol.
7. [Interfaces](07-interfaces.md) — the single reference for buses, pins, addresses, and module activation.
8. [System-level findings](08-system-findings.md) — the three gaps, one of them safety-critical.
9. [Development](09-development.md) — build, flash, trace, and the rules to work by.
10. [Ground-side contributions](10-ground-side.md) — the Operations API endpoints and the scheduler ground tooling.
11. [Open decisions & roadmap](11-open-decisions.md) — what's unresolved and the build work that's unblocked, tagged by how much to trust each item.

---

## How to read this

Each subsystem document follows the same shape: **what it is**, **what was added**, **how it works**, **how it was designed** (where there was a design), **status**, and **where it lives in the code**. Interface facts (pins, buses, addresses) have a single home in [Interfaces](07-interfaces.md); everything else links there rather than restating.

A verification tag is used throughout: **HW** hardware-verified · **SW** software-verified on the OBC · **IMPL** implemented and building, not yet run on hardware · **SKEL** structure without functional behaviour · **OPEN** blocked on a decision or a measurement.

EnduroSat SDK and protocol terms (ESPS, rparam, FP/CP, FIDL, COBS, `comm_gw`, `sys_instancer`, …) are defined in the [Glossary](00-glossary.md); they are used without expansion in the body to keep it readable.

> **Flight-critical items and open decisions are called out where they arise, never buried.** The most urgent is the deployment-inhibit delay ([§8](08-system-findings.md), [§7](07-interfaces.md)).
