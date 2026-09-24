# PROTOCOL — the shared contract

> **Status: FROZEN.** Changes require the team lead's sign-off and a message in the group, because a change here breaks four other people's work simultaneously.
>
> This document is what lets six people build in parallel. The simulator, the firmware, the gateway, the backend and the frontend all agree on exactly what a message looks like. When real hardware replaces the simulator, nothing downstream changes.

## 1. Identifiers

| Field | Format | Example |
|-------|--------|---------|
| `feeder_id` | `<UTILITY>-<SUBSTATION>-<FEEDER>` | `KSEB-TVM-F12` |
| `node_id` | `N-` + 3 digits, ascending downstream from the feeder head | `N-007` |
| `ts` | Unix epoch milliseconds, integer, UTC | `1757030400123` |

Node numbering is ordered along the feeder. `N-006` is immediately upstream of `N-007`. This ordering is what makes the fault span computable — **do not renumber nodes arbitrarily.**

## 2. Node states

| State | Meaning |
|-------|---------|
| `NORMAL` | E-field within adaptive baseline tolerance |
| `SUSPECT` | Collapse detected locally, awaiting neighbour votes |
| `CONFIRMED` | Quorum reached, fault asserted |
| `RECOVERED` | Was `SUSPECT`, field returned before quorum (false alarm rejected) |
| `OFFLINE` | No telemetry for > 30 s |

`RECOVERED` matters. It is the state that proves the system rejects false positives, and the frontend must display it prominently.

## 3. Transport

| Environment | Transport |
|-------------|-----------|
| Demo (simulator) | In-process pub/sub inside the backend, then WebSocket to browser |
| Production | Node → LoRa → Gateway → MQTT over LTE → Backend → WebSocket to browser |

MQTT topic scheme:

```
cc/feeder/{feeder_id}/node/{node_id}/telemetry
cc/feeder/{feeder_id}/vote
cc/feeder/{feeder_id}/command
cc/feeder/{feeder_id}/event
```

- QoS 1 for telemetry, QoS 2 for command and event.
- Retained flag OFF for telemetry, ON for the last event.

## 4. Message schemas

### 4.1 telemetry — every node, 2 Hz nominal

```json
{
  "node_id": "N-007",
  "feeder_id": "KSEB-TVM-F12",
  "ts": 1757030400123,
  "efield_rms": 4.82,
  "baseline": 4.90,
  "deviation_pct": -1.6,
  "battery_mv": 3720,
  "rssi": -94,
  "temp_c": 31.4,
  "state": "NORMAL",
  "seq": 88213
}
```

| Field | Type | Unit / range | Notes |
|-------|------|--------------|-------|
| `efield_rms` | float | kV/m, 0.0–20.0 | normalised probe output, 10-cycle RMS |
| `baseline` | float | kV/m | slow EWMA, τ ≈ 10 min |
| `deviation_pct` | float | −100.0 to +100.0 | (efield_rms − baseline) / baseline × 100 |
| `battery_mv` | int | 2800–4200 | LiFePO4 terminal voltage |
| `rssi` | int | −140 to −30 | dBm, last received LoRa packet |
| `temp_c` | float | −10.0 to 60.0 | enclosure internal |
| `seq` | int | monotonic | replay detection; resets to 0 on reboot |

### 4.2 vote — emitted when a node goes SUSPECT

```json
{
  "feeder_id": "KSEB-TVM-F12",
  "asserting_node": "N-007",
  "ts": 1757030400480,
  "votes": [
    { "node_id": "N-006", "agrees": false, "deviation_pct": -2.1 },
    { "node_id": "N-008", "agrees": true, "deviation_pct": -87.4 }
  ],
  "quorum_required": 2,
  "quorum_met": false
}
```

Quorum is evaluated over the nearest downstream neighbours. A break between `N-006` and `N-007` collapses the field at `N-007` and everything downstream, while `N-006` stays normal. That asymmetry is the signature — a global collapse across all nodes is a substation outage, not a break, and must not trip.

### 4.3 command — gateway to actuator

```json
{
  "feeder_id": "KSEB-TVM-F12",
  "action": "ISOLATE",
  "reason": "quorum_confirmed",
  "fault_span": ["N-006", "N-007"],
  "ts": 1757030400910,
  "latency_ms": 787,
  "issued_by": "GW-TVM-01",
  "mode": "AUTO"
}
```

- `action` ∈ `ISOLATE` | `RESTORE` | `LOCKOUT` | `TEST`
- `reason` ∈ `quorum_confirmed` | `manual` | `scheduled_test` | `watchdog`
- `mode` ∈ `AUTO` | `ALERT_ONLY` — per-feeder configuration, default `ALERT_ONLY`

`latency_ms` is measured from the first `SUSPECT` timestamp to command issue. Surface it in the UI on every event; it is the headline number of the whole project.

### 4.4 event — durable record for the timeline

```json
{
  "event_id": "evt_01J9X2K",
  "feeder_id": "KSEB-TVM-F12",
  "type": "CONDUCTOR_BREAK",
  "ts_detected": 1757030400123,
  "ts_confirmed": 1757030400910,
  "fault_span": ["N-006", "N-007"],
  "location": { "lat": 8.5241, "lng": 76.9366 },
  "isolated": true,
  "latency_ms": 787,
  "trail": [
    { "ts": 1757030400123, "node": "N-007", "state": "SUSPECT" },
    { "ts": 1757030400480, "node": "N-008", "state": "SUSPECT" },
    { "ts": 1757030400802, "node": "N-009", "state": "SUSPECT" },
    { "ts": 1757030400910, "node": "GW", "state": "QUORUM" }
  ],
  "acknowledged_by": null
}
```

`type` ∈ `CONDUCTOR_BREAK` | `FALSE_POSITIVE_REJECTED` | `NODE_OFFLINE` | `LOW_BATTERY` | `COMMS_LOSS`

## 5. decide() — the one shared function

This signature is implemented once and reused. Written in Python in `backend/app/arbiter.py`, ported to C in `gateway/` and `firmware/`. Do not write two divergent versions.

```python
def decide(
    nodes: list[NodeState],   # ordered upstream → downstream
    now_ms: int,
    config: ArbiterConfig,
) -> Decision:
    """
    Pure. No I/O, no clock reads, no globals.
    Same input must always give the same output — that is what makes it testable
    and what lets the gateway and the cloud agree.
    """
```

```python
@dataclass(frozen=True)
class ArbiterConfig:
    collapse_threshold_pct: float = -60.0  # deviation to enter SUSPECT
    sustain_cycles: int = 5                # consecutive samples required
    quorum_required: int = 2               # downstream neighbours that must agree
    quorum_window_ms: int = 1500           # votes older than this expire
    recovery_threshold_pct: float = -20.0  # deviation to return to NORMAL
    global_collapse_veto: bool = True      # all nodes down = outage, never trip

@dataclass(frozen=True)
class Decision:
    action: str                            # NONE | ALERT | ISOLATE
    reason: str
    fault_span: tuple[str, str] | None
    latency_ms: int | None
    confidence: float                      # 0.0–1.0
```

### Veto rules — the credibility of the project lives here

1. **Global collapse veto.** If every node on the feeder collapses simultaneously, that is a substation-side outage. Never trip.
2. **Upstream sanity.** The node immediately upstream of the asserted span must still read NORMAL. If it does not, the span is wrong.
3. **Comms loss ≠ fault.** A node going OFFLINE is a maintenance alert, never a vote toward isolation.
4. **Rate limit.** No more than one ISOLATE per feeder per 60 s without manual reset.
5. **Default mode is ALERT_ONLY.** Auto-isolation is opted into per feeder by the utility, not by us.

## 6. Versioning

Every message carries an implicit schema version via the topic. Breaking changes go to `cc/v2/...`. Until the finale, there is no v2.

## 7. Test vectors

`docs/vectors/` holds JSON fixtures every implementation must pass:

| File | Expected decide() output |
|------|--------------------------|
| `break_mid_feeder.json` | ISOLATE, span ["N-006","N-007"] |
| `break_at_tail.json` | ISOLATE, span ["N-010","N-011"] |
| `substation_outage.json` | NONE (global collapse veto) |
| `rain_burst.json` | NONE (recovers before sustain) |
| `vegetation_contact.json` | ALERT only, no isolate |
| `single_node_offline.json` | ALERT only |
| `switching_transient.json` | NONE |

If the Python and C implementations disagree on any vector, the build is broken.
