# 2. Power Subsystem — `eps_m`

The OBC's driver for the GomSpace P60 power system. This is the one part of the work fully hardware-verified against an independent reference, and it is the template the other bus-connected devices are meant to follow.

**Status: HW** (telemetry) · channel control **IMPL** (address-confirmed, never commanded).
**Code:** `espf/arch/stm32h753iit/drivers/power/eps_m/`.

---

## 2.1 What it is and what was added

`eps_m` is a background service that polls the P60 PDU-200 over CSP-over-CAN every 10 s and publishes the readings into DataCache for telemetry and FDIR. It is built from the SDK's `eps_m` integration point and provides the rparam request/response handling, the housekeeping reads, the DataCache publication, and a channel-control write path.

The rparam handling is modelled on GomSpace's own rparam client (from the P60 SDK) and re-derived for the OBC over CSP-over-CAN, rather than linking the GomSpace library — which is why `eps_m` hand-rolls its request/response structs. The motive is a driver that is self-contained on the OBC with no dependency on the vendor ground SDK; the trade-off is that the wire format is tracked by hand, so a future option is to migrate to the GomSpace client if the parameter set grows.

## 2.2 How it works

The driver speaks GomSpace **rparam** over CSP/CAN to the P60 Dock:

- **Transport:** `csp_transaction_w_opts()` to **CSP node 4** (the Dock), **CSP port 7** (the OBC's own libcsp rparam service port). This is distinct from port 10, which is the path GomSpace's own `power` GOSH client uses to reach PDU channels through the Dock — the two are not interchangeable.
- **Wire format:** addresses are big-endian (`csp_hton16`); U16 replies need `csp_ntoh16`, U8 replies need no swap. The reply echoes the 2-byte requested address in every payload (a U16 read returns 14 bytes, not 12). Every request header carries the fixed magic checksum `0x0bb0` that skips the P60's server-side CRC validation; the bus runs at 1000 kbps and a mismatched rate fails silently (see [§7](07-interfaces.md)).
- **Housekeeping (table 4), every 10 s:** battery voltage `0x0074` (U16, mV), board temperature `0x0044` (I16, 0.1 °C, signed), battery current `0x0078` (I16, mA), battery mode `0x0056` (U8: 1=Crit 2=Safe 3=Normal 4=Full). Extended reads cover heater state and charge/discharge accumulators; a latchup scan runs on a slower cadence.
- **DataCache output:** each poll writes `dc_set_eps_0_data()` (`DC_DID_EPS_0_DATA`) with voltage, current, and temperature.
- **Fault reporting:** when the P60 stops answering on the bus, `eps_m` raises `FDIR_FAULT_EPS_PDM_CMD_EXEC_FAILURE` (agent `FDIR_AGENT_EPS_M`) and clears it on recovery — so a dead power link surfaces as a visible fault rather than silently stale telemetry. This is the one fault `eps_m` owns; the low-battery safing fault is owned separately by `eps_ctrl` (see [§1.3](01-system-architecture.md) and [§8](08-system-findings.md)).
- **Channel control (table 1):** `eps_m_set_channel_output()` writes `out_en[i]` at **`0x0068 + i`**. Channel-enable *readback* is a separate table-4 read at `0x0034 + i`.

The P60 is a **single CSP node** (node 4 = Dock); the PDU and ACU are not separate nodes — the Dock aggregates both. Node 3 does not exist on this bus; node 1 is the OBC's own placeholder identity and must not be targeted.

One 10-second poll cycle, and the two outcomes:

```mermaid
sequenceDiagram
    participant D as eps_m driver
    participant C as csp_transaction_w_opts
    participant P as P60 Dock (node 4, port 7)
    participant DC as DataCache
    participant F as FDIR
    loop every 10 s
        D->>C: rparam GET (addr big-endian, magic 0x0bb0)
        C->>P: CSP/CAN request @ 1000 kbps
        alt Dock answers
            P-->>C: reply (echoes 2-byte addr; U16 read = 14 B)
            C-->>D: payload
            D->>D: byte-swap U16 (csp_ntoh16), scale
            D->>DC: dc_set_eps_0_data() → DC_DID_EPS_0_DATA
            D->>F: clear FDIR_FAULT_EPS_PDM_CMD_EXEC_FAILURE
        else bus quiet / no reply
            D->>F: raise FDIR_FAULT_EPS_PDM_CMD_EXEC_FAILURE (agent EPS_M)
        end
    end
```

## 2.3 Addresses are the live-device values, not the manual's

The housekeeping and channel-control addresses in §2.2 are the values confirmed against the live Dock (firmware 2.2.9), not the vendor manual — and they must stay that way. The 2017 PDU-200 manual is unreliable for this firmware revision: its battery-voltage address lands on a channel-voltage field (~15.4 V), it exposes no VCC-voltage parameter in table 4, and its table-1 channel-enable address (`0x0048+i`) is the wrong layout — the Dock uses `0x0068+i`. **Rule for any P60 work: dump the live device (`rparam download <node> <table>`) and confirm the address before relying on it; do not trust the manual.**

## 2.4 Verification — the one hardware-verified result

The telemetry reads were cross-checked field-by-field against the GomSpace GOSH console on the bench, and every field matched within its resolution:

| Parameter | OBC (`eps_m`) | GOSH | Result |
|---|---|---|---|
| Battery voltage | 15414 mV | 15429 mV | Pass |
| Temperature | 28.0 °C | 280 (÷10) | Pass |
| Battery current | −37 mA | −37 mA | Pass |
| Battery mode | 3 (Normal) | 3 | Pass |

This is the first communication path in the 3Cat-8 OBC software verified end to end on real hardware.

**The reusable method** (documented for every future bus device): (1) implement against the spec, then review the implementation against an authoritative source *before* connecting hardware — protocol errors in flight software are silent, the transaction appears to succeed but the value is wrong; (2) map the live device empirically rather than trusting documentation; (3) cross-check every reading against an independent reference. A match on the independent reference is what turns "the code runs" into "the value is correct."

## 2.5 Channel control: implemented, address-confirmed, never commanded

The write path is built and its address is confirmed on the live Dock, but **no channel has ever been commanded** — no subsystem load is connected, so there is no reason to risk a live write. Before commanding any channel: verify the CH2/CH8 power-on-boot defaults, and note that CH0 (OBC + attitude) is the hardware-always-on rail with no software-meaningful enable bit. Channel-control writes are therefore tagged **OPEN** for flight use. See [§7](07-interfaces.md) for the full channel map.
