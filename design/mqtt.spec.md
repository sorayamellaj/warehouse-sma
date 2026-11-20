# MQTT specification

## 1. Topic hierarchy

| Topic | Publisher | Role | QoS | Retained | Justification |
|---|---|---|---|---|---|
| warehouse/agv/{id}/pose | AGV sensor | Position and battery | 0 | no | Sent every second: losing one is harmless, the next replaces it |
| warehouse/agv/{id}/cmd | AGVAgent | Route or wait order to the actuator | 1 | no | Must arrive; an old order must not be replayed to a new subscriber |
| warehouse/agv/{id}/state | Actuator | Actual state of the robot | 1 | yes | Last known state must be available to late subscribers |
| warehouse/order/{id} | OrderAgent | New order to prepare | 1 | no | Must not be lost |
| warehouse/aisle/{id}/request | AGVAgent | Reservation request (segment, slot) | 1 | no | Must arrive; a request is an event, not a state |
| warehouse/aisle/{id}/reservation | AisleAgent | Current reservation table of the segment | 1 | yes | State: a new agent must see current reservations immediately |
| warehouse/charger/{id}/cfp | ChargingStationAgent | Call for proposals (Contract Net) | 1 | no | Event; must arrive |
| warehouse/charger/{id}/bid | AGVAgent | Bid for a charging slot | 1 | no | Event; must arrive |
| warehouse/charger/{id}/queue | ChargingStationAgent | Queue and awarded slots | 1 | yes | State: late agents must see the queue |
| sma/agent/{id}/statut | Agent / broker (LWT) | online / offline | 1 | yes | Failure detection must survive late subscribers |

QoS 2 is not used: duplicates are harmless here (a repeated reservation request is ignored), and QoS 2 adds extra message exchanges.

## 2. Wildcard subscriptions

| Subscriber | Subscription | Meaning |
|---|---|---|
| AGVAgent | warehouse/aisle/+/reservation | Reservation tables of all aisles |
| AisleAgent s2 | warehouse/aisle/s2/request | Requests for its own segment |
| ChargingStationAgent | warehouse/charger/c1/bid | Bids for its own station |
| All agents | sma/agent/+/statut | Status of every agent |
| Monitoring / logger | warehouse/# | Everything in the warehouse |

`+` replaces exactly one level, `#` replaces all following levels and must be last.

## 3. Last Will and Testament

Set when the agent connects:

- topic: sma/agent/{id}/statut
- payload: {"ts": "...", "source": "{id}", "value": "offline", "unit": "state"}
- QoS 1, retained
- keep-alive: 10 s (the broker declares the client dead after about 15 s of silence)

On connection, the agent publishes "online" (retained) on the same topic.
On a clean shutdown, the agent publishes "offline" itself. On a crash, the broker publishes the LWT.
Reaction: the AisleAgents release the future reservations of the dead agent and mark its current segment as blocked; the other robots restart the consensus.

## 4. Message format (JSON)

Common schema for all messages:

```json
{
  "ts": "2026-11-20T17:33:12Z",
  "source": "agv07",
  "value": { "x": 12.5, "y": 4.0, "battery": 46 },
  "unit": { "x": "m", "y": "m", "battery": "%" }
}
```

| Field | Type | Meaning |
|---|---|---|
| ts | string | ISO 8601 timestamp, UTC |
| source | string | Identifier of the sender |
| value | object or string | The payload (measurement, command, request) |
| unit | object or string | Unit of each value field |

Example on warehouse/aisle/s2/request:

```json
{
  "ts": "2026-11-20T17:33:14Z",
  "source": "agv07",
  "value": { "segment": "s2", "slot": 15 },
  "unit": { "segment": "id", "slot": "time slot" }
}
```

## 5. Security (bonus)

- Authentication: username and password per agent on the broker
- TLS on port 8883
- ACL per topic: an AGVAgent may publish only on warehouse/agv/{its own id}/# and warehouse/aisle/+/request, and may not write on another agent's statut topic