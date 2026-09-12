# 10. Ground-Side Contributions

Work outside the OBC firmware, in the ground repositories, that is part of the same effort — the REST endpoints and tools used to command and verify the OBC components. These live in separate repositories (`3cat8-operations-api`, `3cat8-gs`) from the flight firmware, and are recorded here so the contribution is complete.

---

## 10.1 Operations API (`3cat8-operations-api`)

The Operations API is a FastAPI REST service that fronts the SDK command model, so the OBC can be commanded and queried over HTTP (Swagger at `/swagger/`). Three endpoint modules are attributed to this work (by the `@author` / "Added by" headers in the source):

| File | Contribution | Endpoints |
|---|---|---|
| `app/api/endpoints/serial/payload_ctrl.py` | Authored (April 2026) — ground control of the payload-controller handlers | `GET /payload_ctrl/start_payload`, `/stop_payload`, `/get_payload_info` |
| `app/api/endpoints/serial/aocs.py` | Authored (May 2026) — ground control of the attitude proxy | `GET /aocs/get_aocs_state`, `/set_aocs_state` |
| `app/api/endpoints/serial/arducam.py` | Patched — robust response parsing | added `_safe_enum()` and safe enum parsing |

**`payload_ctrl`** is the ground counterpart to the mission-payload handlers ([§5](05-rita-payload.md)): it starts, stops, and queries a payload by instance ID. **`aocs`** reads and sets the attitude state exposed by `aocs_cntrl` ([§1](01-system-architecture.md)). The **`arducam` patch** adds `_safe_enum()`, which returns a readable value instead of failing when the OBC reports an uninitialized enum field (`None` → `"Unknown"`, an out-of-range value → the raw value) — so the camera-status endpoint responds cleanly even before the module is connected and its fields are populated.

**Status: SW** — the endpoints load and respond over the interface; they exercise the corresponding OBC components without real hardware attached.

## 10.2 Scheduler ground tooling (`3cat8-gs` and the OBC `other/scripts/`)

The tools that make the onboard scheduler ([§3](03-onboard-scheduler.md)) usable and testable from the ground:

- **`sch_builder.py`** (OBC `other/scripts/sch_builder/`) — builds a `.sch` schedule file from command or script slot definitions, with the correct per-slot CRC16 and the end-of-file handling (last slot's `next` = file size, never `0xFFFFFFFF`). This is the tool that produces every schedule the OBC runs.
- **`bench_multislot.py`** (`3cat8-gs`, header dated 2026-07-17) — the multi-slot bench test: it reads OBC time, builds a two-slot schedule (a CP command then a MicroPython script), uploads via the file manager, starts the scheduler, waits, then downloads and COBS-decodes the `.slog` to confirm both slots fired.

`sch_builder.py` and `bench_multislot.py` are attributed to this work — `sch_builder` by `scheduler_design.md`, `bench_multislot` by its 2026-07-17 header (within the internship). The repository's other `.slog` decode utilities (`decode_slog.py`, `grab_and_decode.py`) are used in the same workflow but carry no author marker or date and are not claimed here.

**Status: HW** — this tooling was used in the single- and multi-slot bench verifications ([§3](03-onboard-scheduler.md)).
