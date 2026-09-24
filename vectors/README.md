# Test vectors — shared fixtures (PROTOCOL §7)

Every `decide()` implementation (Python `backend/app/arbiter.py`, C `gateway/src/arbiter.c` / `firmware/src/arbiter.c`) must pass all fixtures. CI fails on any disagreement.

| File | Expected |
|------|----------|
| `break_mid_feeder.json` | ISOLATE N-006↔N-007 |
| `break_at_tail.json` | ISOLATE tail |
| `substation_outage.json` | NONE (global veto) |
| `rain_burst.json` | NONE |
| `vegetation_contact.json` | ALERT |
| `single_node_offline.json` | ALERT |
| `switching_transient.json` | NONE |
