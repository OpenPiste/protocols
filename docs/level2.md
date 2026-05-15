# OpenPiste Protocol
## A proposal for modern open communication in fencing electronics

**Status:** Draft — working towards v1.0
**Author:** Piet Wauters
**Repository:** https://github.com/OpenPiste
**Website:** https://openpiste.org

---

Fencing electronics have long relied on two communication protocols: EFP1.1 (known as Cyrano), the dominant standard for communication between scoring apparatus and competition management software, and RS422-FPA, a serial protocol for driving external displays and scoreboards. Both were designed for their era and both served the community well. EFP1.1 has been in use since 2008. RS422-FPA traces its roots to 1995.

The world around them has changed. MQTT and JSON are now the lingua franca of connected devices. Libraries exist for every platform from microcontrollers to cloud services. The IoT ecosystem has solved, at scale, the same problems fencing electronics face: reliable message delivery, multiple subscribers, late-joining clients, structured extensible data. There is no longer a compelling reason to maintain a bespoke binary or CSV protocol when open, well-supported alternatives exist.

At the same time, there is a substantial installed base of EFP1.1-compatible apparatus and software. A new protocol that ignores this reality will not be adopted. Migration must be possible without requiring clubs and federations to replace working equipment overnight.

This document introduces the **OpenPiste Protocol**, a proposal for a modern, open communication standard for fencing electronics. It is structured in two levels:

**Level 1** addresses the transition. It defines how existing EFP1.1 payloads can be transported over MQTT without any change to the payload itself. A bridge — a simple piece of software — relays messages between the existing UDP network and an MQTT broker. Existing apparatus and software require no modification. New MQTT-native subscribers (displays, piste monitors, video tools) can immediately consume live scoring data from existing infrastructure. Level 1 is not a long-term target. It is a practical bridge that allows the ecosystem to move at its own pace.

**Level 2** is the destination. It defines a native JSON protocol designed from the ground up for MQTT, drawing on the field semantics of EFP1.1 and the message architecture of RS422-FPA while leaving behind the encoding constraints of both. It uses typed values, purpose-specific messages, retained state, and millisecond-precision timestamps. It is implementable on an ESP32 with standard open source libraries. It is designed to be genuinely open — any manufacturer, developer, club, or federation can implement it without restriction.

This is a working proposal, not a ratified standard. It is published in the hope that it will be useful, reviewed, and improved by the fencing electronics community. Comments, corrections, and contributions are welcome at https://github.com/OpenPiste.

---

# OpenPiste Protocol — Level 2
## Native JSON over MQTT

**Status:** Draft — working towards v1.0
**Date:** May 2026
**Author:** Piet Wauters
**Protocol identifier:** `OPP2`
**Repository:** https://github.com/OpenPiste
**Website:** https://openpiste.org

## Table of Contents

1. [Introduction](#1-introduction)
2. [Design principles](#2-design-principles)
3. [Relationship to prior protocols](#3-relationship-to-prior-protocols)
4. [Network and protocol stack](#4-network-and-protocol-stack)
5. [Topic structure](#5-topic-structure)
6. [Message overview](#6-message-overview)
7. [Message: lights](#7-message-lights)
8. [Message: clock](#8-message-clock)
9. [Message: blade\_contact](#9-message-blade_contact)
10. [Message: score](#10-message-score)
11. [Message: connection](#11-message-connection)
12. [Message: state](#12-message-state)
13. [Message: fencers](#13-message-fencers)
14. [Message: match](#14-message-match)
15. [Message: uw2f](#15-message-uw2f)
16. [Message: medical](#16-message-medical)
17. [Message: video\_review](#17-message-video_review)
18. [Message: control](#18-message-control)
19. [Field types and conventions](#19-field-types-and-conventions)
20. [Timestamp conventions](#20-timestamp-conventions)
21. [Versioning and compatibility](#21-versioning-and-compatibility)
22. [Security](#22-security)
23. [Open items](#23-open-items)

---

## 1. Introduction

Level 2 is the native JSON protocol of the OpenPiste platform. It is designed from the ground up for MQTT, taking full advantage of the broker's publish/subscribe model, topic hierarchy, and retained message capability. It does not carry forward the encoding constraints of EFP1.1.

Level 2 is intended to be a genuinely open standard — any apparatus manufacturer, software developer, club, or federation can implement it without restriction. The protocol identifier `OPP2` appears in every message so that receivers can identify the source and version unambiguously.

A JSON Schema for machine validation of all message types is maintained as a separate document in the OpenPiste repository. See `schemas/opp2/` at https://github.com/OpenPiste. *(Schema publication is a pending task — see Section 23.)*

---

## 2. Design principles

**Typed values.** Integers are integers. Booleans are booleans. No string-encoding of numeric or boolean fields.

**Purpose-specific messages.** Each message type carries only the data relevant to its purpose. A scoreboard that only needs lights and scores does not need to parse fencer names or competition metadata. A video tool that only needs blade contact timestamps does not need to process clock ticks.

**The broker is the single source of truth.** All state-bearing topics use retained messages. Any subscriber connecting at any point during a bout immediately receives the current state of every topic without waiting for the next publish cycle. No periodic heartbeat resends are needed.

**Timestamps on time-critical events.** The lights, clock, and blade contact messages carry a millisecond timestamp. This enables accurate synchronisation with video replay systems — a capability absent from both EFP1.1 and RS422-FPA. All timestamps are UTC. No local time, no timezone offsets, no daylight saving adjustments. See Section 20 for the encoding convention.

**Extensible control.** The control topic carries named command events. New commands can be added without changing the protocol version or breaking existing receivers.

**Implementable on constrained hardware.** The reference implementation runs on an ESP32 using the Arduino MQTT and ArduinoJson libraries, both freely available.

---

## 3. Relationship to prior protocols

Level 2 draws on two existing protocols for its design:

**EFP1.1 (Cyrano)** provides the field semantics: state values, weapon codes, priority values, card counts, fencer status codes. These are preserved in Level 2 where they make sense, so developers familiar with EFP1.1 will recognise the values.

**RS422-FPA** (version 3.04a, 2019) provides the architectural inspiration for typed messages. RS422-FPA demonstrated that splitting scoring data into purpose-specific messages with different transmission priorities is practical and well-understood in the fencing community. In Level 2, MQTT topics replace the RS422 serial bus, the broker's retained message mechanism replaces RS422-FPA's periodic resend strategy, and QoS levels replace RS422-FPA's explicit message priority ordering.

| RS422-FPA message | Level 2 topic | Notes |
|-------------------|--------------|-------|
| Msg 1 — lights | `lights` | Boolean fields; timestamp added |
| Msg 2 — clock | `clock` | Typed fields; timestamp added |
| Msg 3 — scores/cards | `score` | Integer fields; video review counts moved to `video_review` |
| Msg 4 — status | `state` + `connection` | Split into apparatus state and connection status |
| Msg 5+6 — competitor names | `fencers` | Restructured with left/right/common sections |
| Msg 7 — competition info | `match` | Match and competition metadata; round added |
| Msg 8 — UW2F | `uw2f` | Timer and P-cards |
| Msg 9 — bout control | `control` | Extensible command set |
| — | `blade_contact` | No RS422-FPA equivalent; blade contact with timestamp |
| — | `medical` | No RS422-FPA equivalent; medical timeout with countdown timer |
| — | `video_review` | No RS422-FPA equivalent; full call history and remaining counts |

---

## 4. Network and Protocol Stack

### 4.1 Broker

Any MQTT 3.1.1 compliant broker. Mosquitto is recommended for club and competition use — it is open source, lightweight, and runs on a laptop or Raspberry Pi.

### 4.2 Broker discovery

For club and small competition use, the broker host SHOULD be made discoverable via mDNS under the hostname:

```
openpiste.local
```

Any device on the local network can then reach the broker at `openpiste.local:1883` without IP address configuration. All OpenPiste-compatible devices SHOULD use this hostname as their default broker address, with fallback to a configurable IP address or hostname.

For larger competition setups with managed switches or multiple VLANs, mDNS may not propagate reliably across network boundaries. In these cases a static IP address or DHCP reservation for the broker is recommended, and the `openpiste.local` hostname may be configured in local DNS.

### 4.3 NTP

The broker host SHOULD also run a local NTP server. This allows all devices on the network to synchronise their clocks to UTC without requiring internet access. On Linux, `chrony` is recommended — it is lightweight and can serve NTP to local clients while itself operating without an upstream internet time source.

When all devices synchronise to the same local NTP server, timestamps in Level 2 messages are comparable across apparatus, displays, and video tools — enabling accurate video synchronisation on a fully self-contained competition network.

See Section 20 for the timestamp encoding convention, including the fallback behaviour when NTP is unavailable.

### 4.4 QoS

| QoS | Applied to | Rationale |
|-----|-----------|-----------|
| 0 (at most once) | clock, blade_contact | High frequency or latency-critical. A missed clock tick self-corrects within one second. Blade contact retransmission latency would degrade timestamp precision for video sync. |
| 1 (at least once) | lights, score, connection, state, fencers, match, uw2f, medical, video_review, control | State changes and commands that must not be silently lost. A missed lights message would leave subscribers with incorrect light state indefinitely. |

### 4.5 Retained messages

All topics use retained messages **except** blade_contact and control.

Retained messages mean the broker holds the last published value on each topic. A subscriber connecting after the apparatus is online immediately receives the current state of every retained topic. Combined with QoS 1 on all state-bearing topics, this eliminates the need for periodic heartbeat resends.

**blade_contact** is not retained because it is a point-in-time event. A retained blade contact message would cause a late subscriber to receive a contact notification with no way to know it was already resolved.

**control** is not retained because commands are one-shot. A late subscriber must not act on a BEGIN or NEXT command that was issued before it connected.

### 4.6 Last Will and Testament

Every apparatus MUST configure a Last Will and Testament (LWT) message when connecting to the broker. The LWT is set in the MQTT CONNECT packet — it is not published by the apparatus directly, but by the broker automatically if the apparatus disconnects unexpectedly.

The LWT MUST be configured as follows:
- **Topic:** `openpiste/{piste_id}/connection`
- **Payload:** `{"online": false}`
- **QoS:** 1
- **Retain:** true

This ensures all subscribers are notified promptly when an apparatus goes offline unexpectedly.

### 4.7 Port

Standard MQTT port 1883 (unencrypted) or 8883 (TLS).

---

## 5. Topic structure

All Level 2 topics follow this pattern:

```
openpiste/{piste_id}/{message_type}
```

| Segment | Description | Examples |
|---------|-------------|---------|
| `openpiste` | Fixed platform prefix | — |
| `{piste_id}` | Piste identifier — number, name, or colour | `17`, `podium`, `rouge`, `vert` |
| `{message_type}` | Message type as defined in Section 6 | `lights`, `clock`, `score` |

Useful subscription patterns:

```
openpiste/#                  # all messages from all pistes
openpiste/17/#               # all messages from piste 17
openpiste/+/lights           # lights from all pistes
openpiste/+/score            # scores from all pistes
openpiste/+/control          # control events from all pistes
openpiste/+/connection       # connection status from all pistes
```

---

## 6. Message overview

| Topic suffix | QoS | Retained | Published when |
|-------------|-----|----------|---------------|
| `lights` | 1 | Yes | On any light change |
| `clock` | 0 | Yes | Every second while running; on any clock state change |
| `blade_contact` | 0 | No | On blade contact event |
| `score` | 1 | Yes | On score, card, or priority change |
| `connection` | 1 | Yes | On connection or disconnection (including LWT) |
| `state` | 1 | Yes | On apparatus state change |
| `fencers` | 1 | Yes | On fencer, coach, or referee identity change |
| `match` | 1 | Yes | On match or competition metadata change |
| `uw2f` | 1 | Yes | On UW2F timer or P-card change |
| `medical` | 1 | Yes | On medical timeout event or timer update |
| `video_review` | 1 | Yes | On video review request or resolution |
| `control` | 1 | No | On remote control event |

---

## 7. Message: lights

**Topic:** `openpiste/{piste_id}/lights`
**QoS:** 1
**Retained:** Yes

Published immediately on any change to the light state. This is the highest-priority message — published before any other pending message when a light state changes. QoS 1 ensures that a missed lights message is retransmitted, preventing subscribers from holding a permanently incorrect light state.

Light colour conventions apply across all weapons:
- **Red** light: left fencer scored (on target)
- **Green** light: right fencer scored (on target)
- **White** light: off-target hit (foil) or broken circuit (sabre)

### Payload

```json
{
  "protocol": "OPP2",
  "right": {
    "green": false,
    "white": true
  },
  "left": {
    "red":   true,
    "white": false
  },
  "ts": 1715539200123
}
```

### Fields

| Field | Type | Description |
|-------|------|-------------|
| `protocol` | string | Always `"OPP2"` |
| `right.green` | boolean | Right fencer on-target light |
| `right.white` | boolean | Right fencer white (off-target / broken circuit) light |
| `left.red` | boolean | Left fencer on-target light |
| `left.white` | boolean | Left fencer white (off-target / broken circuit) light |
| `ts` | integer | Timestamp — see Section 20 |

---

## 8. Message: clock

**Topic:** `openpiste/{piste_id}/clock`
**QoS:** 0
**Retained:** Yes

Published once per second while the stopwatch is running. Also published immediately on any clock state change (start, stop, reset). QoS 0 is appropriate — a missed clock tick self-corrects within one second, and the retained message ensures late-joining subscribers receive the current time immediately.

### Payload

```json
{
  "protocol": "OPP2",
  "running":  true,
  "time_ms":  89250,
  "time":     "1:29.25",
  "ts":       1715539200123
}
```

### Fields

| Field | Type | Description |
|-------|------|-------------|
| `protocol` | string | Always `"OPP2"` |
| `running` | boolean | `true` if the stopwatch is currently running |
| `time_ms` | integer | Current stopwatch value in milliseconds |
| `time` | string | Formatted as `"M:SS"` or `"M:SS.cc"`. Hundredths mandatory below 10 seconds. |
| `ts` | integer | Timestamp — see Section 20 |

---

## 9. Message: blade_contact

**Topic:** `openpiste/{piste_id}/blade_contact`
**QoS:** 0
**Retained:** No

Published on blade contact events. The primary purpose of this message is to provide a precise timestamp for synchronisation with video replay systems. Not every blade contact is a scoring touch or a parry — this message records the raw electrical event. It enables referees and AI tools to determine whether a scoring action involved genuine blade contact.

QoS 0 is used because retransmission latency would degrade timestamp precision, which is the primary value of this message.

> **Note:** The full semantics of this message are not yet finalised — see Section 23.

### Payload

```json
{
  "protocol": "OPP2",
  "active":   true,
  "ts":       1715539200089
}
```

### Fields

| Field | Type | Description |
|-------|------|-------------|
| `protocol` | string | Always `"OPP2"` |
| `active` | boolean | `true` — blade contact detected; `false` — contact cleared |
| `ts` | integer | Timestamp — see Section 20 |

---

## 10. Message: score

**Topic:** `openpiste/{piste_id}/score`
**QoS:** 1
**Retained:** Yes

Published on any change to scores, cards, or priority.

### Payload

```json
{
  "protocol": "OPP2",
  "right": {
    "score":       8,
    "status":      "V",
    "yellow_card": false,
    "red_cards":   1
  },
  "left": {
    "score":       6,
    "status":      "D",
    "yellow_card": false,
    "red_cards":   0
  },
  "priority": "N"
}
```

### Fields

| Field | Type | Description |
|-------|------|-------------|
| `protocol` | string | Always `"OPP2"` |
| `right.score` | integer | Right fencer score |
| `right.status` | string | Right fencer match status — see values below |
| `right.yellow_card` | boolean | Right fencer yellow card |
| `right.red_cards` | integer | Right fencer red card count (0–9) |
| `left.score` | integer | Left fencer score |
| `left.status` | string | Left fencer match status |
| `left.yellow_card` | boolean | Left fencer yellow card |
| `left.red_cards` | integer | Left fencer red card count (0–9) |
| `priority` | string | `"N"` none, `"R"` right, `"L"` left |

**Status values:**

| Value | Meaning |
|-------|---------|
| `"U"` | Undefined |
| `"V"` | Victory |
| `"D"` | Defeat |
| `"A"` | Abandonment |
| `"E"` | Exclusion |
| `"DNS"` | Did not show |

---

## 11. Message: connection

**Topic:** `openpiste/{piste_id}/connection`
**QoS:** 1
**Retained:** Yes

Indicates whether the apparatus is currently connected to the broker. This topic is the target of the LWT configured by the apparatus at connect time (see Section 4.6). The broker publishes `"online": false` automatically if the apparatus disconnects unexpectedly.

### Payload — apparatus online

```json
{
  "protocol": "OPP2",
  "online":   true,
  "device":   "OpenPiste-ESP32",
  "version":  "1.0.0"
}
```

### Payload — offline (LWT, published by broker)

```json
{
  "online": false
}
```

### Fields

| Field | Type | Description |
|-------|------|-------------|
| `protocol` | string | Always `"OPP2"` (omitted in LWT payload) |
| `online` | boolean | `true` — apparatus connected; `false` — offline |
| `device` | string | Device model or identifier (optional) |
| `version` | string | Firmware or software version (optional) |

---

## 12. Message: state

**Topic:** `openpiste/{piste_id}/state`
**QoS:** 1
**Retained:** Yes

Indicates the current operational state of the scoring apparatus. Published on every state transition.

### Payload

```json
{
  "protocol": "OPP2",
  "state":    "F"
}
```

### Fields

| Field | Type | Description |
|-------|------|-------------|
| `protocol` | string | Always `"OPP2"` |
| `state` | string | Apparatus state — see values below |

**State values** (inherited from EFP1.1):

| Value | Meaning |
|-------|---------|
| `"F"` | Fencing — stopwatch running |
| `"H"` | Halt — stopwatch stopped, bout in progress |
| `"P"` | Pause — between periods |
| `"W"` | Waiting — no active bout |
| `"E"` | Ending — awaiting ACK from software |

---

## 13. Message: fencers

**Topic:** `openpiste/{piste_id}/fencers`
**QoS:** 1
**Retained:** Yes

Published when any participant identity information changes. In team competitions, republished at the end of each round when fencer assignments change. The message is structured in three sections: `left`, `right`, and `common`.

### Payload

```json
{
  "protocol": "OPP2",
  "left": {
    "fencer": {
      "id":     "32",
      "name":   "B. Panini",
      "nation": "ITA"
    },
    "coach": {
      "id":     "c1",
      "name":   "M. Rossi",
      "nation": "ITA"
    }
  },
  "right": {
    "fencer": {
      "id":     "28",
      "name":   "P. Martin",
      "nation": "FRA"
    },
    "coach": {
      "id":     "c2",
      "name":   "J. Dupont",
      "nation": "FRA"
    }
  },
  "common": {
    "referee": {
      "id":     "132",
      "name":   "J. Smith",
      "nation": "GBR"
    },
    "video_official": {
      "id":     "ref002",
      "name":   "L. Dubois",
      "nation": "FRA"
    }
  }
}
```

### Fields

| Field | Type | Description |
|-------|------|-------------|
| `protocol` | string | Always `"OPP2"` |
| `left.fencer.id` | string | Left fencer identifier |
| `left.fencer.name` | string | Left fencer name |
| `left.fencer.nation` | string | IOC 3-letter nation code |
| `left.coach.id` | string | Left fencer coach identifier (optional) |
| `left.coach.name` | string | Left fencer coach name (optional) |
| `left.coach.nation` | string | Left fencer coach nation (optional) |
| `right.fencer.id` | string | Right fencer identifier |
| `right.fencer.name` | string | Right fencer name |
| `right.fencer.nation` | string | IOC 3-letter nation code |
| `right.coach.id` | string | Right fencer coach identifier (optional) |
| `right.coach.name` | string | Right fencer coach name (optional) |
| `right.coach.nation` | string | Right fencer coach nation (optional) |
| `common.referee.id` | string | Referee identifier (optional) |
| `common.referee.name` | string | Referee name (optional) |
| `common.referee.nation` | string | Referee nation (optional) |
| `common.video_official.id` | string | Video review official identifier (optional) |
| `common.video_official.name` | string | Video review official name (optional) |
| `common.video_official.nation` | string | Video review official nation (optional) |

If a field is not available it is omitted. Receivers MUST handle missing fields gracefully.

---

## 14. Message: match

**Topic:** `openpiste/{piste_id}/match`
**QoS:** 1
**Retained:** Yes

Published when match or competition metadata changes, including round changes during team competitions.

### Payload

```json
{
  "protocol":    "OPP2",
  "piste":       "17",
  "weapon":      "E",
  "type":        "I",
  "competition": "efj-eq",
  "phase_type":  "DE",
  "phase":       "3",
  "poule":       "A32",
  "match":       12,
  "round":       1,
  "scheduled":   "13:15"
}
```

### Fields

| Field | Type | Description |
|-------|------|-------------|
| `protocol` | string | Always `"OPP2"` |
| `piste` | string | Piste identifier, matching the topic |
| `weapon` | string | `"F"` foil, `"E"` épée, `"S"` sabre |
| `type` | string | `"I"` individual, `"T"` team |
| `competition` | string | Competition identifier |
| `phase_type` | string | Phase type — see values below |
| `phase` | string | Phase identifier (e.g. round of poules, tableau size) |
| `poule` | string | Poule or tableau identifier |
| `match` | integer | Match number |
| `round` | integer | Current round or period number (team: 1–9; individual: 1–3) |
| `scheduled` | string | Scheduled start time as `"HH:MM"` (optional) |

**Phase type values:**

| Value | Meaning |
|-------|---------|
| `"pool"` | Pool / poule round |
| `"DE"` | Direct elimination |
| `"repechage"` | Repechage |
| `"classification"` | Classification round |

Additional phase type values may be defined in future revisions without a protocol version change.

---

## 15. Message: uw2f

**Topic:** `openpiste/{piste_id}/uw2f`
**QoS:** 1
**Retained:** Yes

Published on any change to the unwillingness-to-fight (passivity) timer or P-card state. The UW2F timer counts upward from zero.

### Payload

```json
{
  "protocol": "OPP2",
  "time_ms":  60000,
  "time":     "1:00",
  "right": {
    "p_card": 1
  },
  "left": {
    "p_card": 0
  }
}
```

### Fields

| Field | Type | Description |
|-------|------|-------------|
| `protocol` | string | Always `"OPP2"` |
| `time_ms` | integer | UW2F timer value in milliseconds, counting up from zero |
| `time` | string | UW2F timer formatted as `"M:SS"` |
| `right.p_card` | integer | Right fencer P-card status — see values below |
| `left.p_card` | integer | Left fencer P-card status |

At least one of `time_ms` or `time` MUST be present. Implementations MAY include both.

**P-card values:**

| Value | Meaning |
|-------|---------|
| `0` | No P-card |
| `1` | First P-card |
| `2` | Second P-card |
| `3` | Third P-card |
| `4` | Fourth P-card |
| `5` | Fifth P-card |

P-card semantics (which card type corresponds to which ordinal) are defined by the applicable rulebook and may change between rule editions. The protocol records only the ordinal position.

---

## 16. Message: medical

**Topic:** `openpiste/{piste_id}/medical`
**QoS:** 1
**Retained:** Yes

Published when a medical timeout is granted and on every subsequent timer update. The medical timeout is initiated via a control command (see Section 18) issued by the apparatus when the referee grants the timeout. The countdown timer runs from the duration specified in the initiating control command.

### Payload — timeout active

```json
{
  "protocol":    "OPP2",
  "active":      true,
  "side":        "left",
  "duration_ms": 300000,
  "remaining_ms": 247000,
  "remaining":   "4:07"
}
```

### Payload — timeout ended or cleared

```json
{
  "protocol": "OPP2",
  "active":   false,
  "side":     "left"
}
```

### Fields

| Field | Type | Description |
|-------|------|-------------|
| `protocol` | string | Always `"OPP2"` |
| `active` | boolean | `true` — timeout in progress; `false` — timeout ended or cleared |
| `side` | string | `"left"` or `"right"` — the fencer who requested the timeout |
| `duration_ms` | integer | Total timeout duration in milliseconds as specified at initiation (present when active) |
| `remaining_ms` | integer | Remaining time in milliseconds, counting down (present when active) |
| `remaining` | string | Remaining time formatted as `"M:SS"` (present when active) |

Timer resolution is 1 second. At least one of `remaining_ms` or `remaining` MUST be present when active.

---

## 17. Message: video\_review

**Topic:** `openpiste/{piste_id}/video_review`
**QoS:** 1
**Retained:** Yes

Published when a video review is requested or resolved. Carries both the current remaining call counts and the full call history for the bout. The call history allows video referee tools and AI systems to reconstruct the complete sequence of review events.

### Payload

```json
{
  "protocol": "OPP2",
  "left": {
    "remaining": 1,
    "calls": [
      {
        "id":      1,
        "round":   1,
        "time_ms": 89250,
        "granted": false
      }
    ]
  },
  "right": {
    "remaining": 2,
    "calls": []
  }
}
```

### Fields

| Field | Type | Description |
|-------|------|-------------|
| `protocol` | string | Always `"OPP2"` |
| `left.remaining` | integer | Video review calls remaining for left fencer |
| `left.calls` | array | History of all video review calls made by left fencer this bout |
| `right.remaining` | integer | Video review calls remaining for right fencer |
| `right.calls` | array | History of all video review calls made by right fencer this bout |

**Call history object fields:**

| Field | Type | Description |
|-------|------|-------------|
| `id` | integer | Sequential call identifier, starting at 1 |
| `round` | integer | Round or period in which the call was made |
| `time_ms` | integer | Stopwatch value in milliseconds at the moment of the call |
| `granted` | boolean | `true` — call granted (fencer retains remaining calls); `false` — call denied (fencer loses one call). Absent if the call has not yet been resolved. |

The call history array grows as calls are made during the bout. A call without a `granted` field represents a call that is currently under review. Published on every request and every resolution.

**Initial call counts by phase:**
- Pool matches and team matches: 1 call per fencer
- Direct elimination: 2 calls per fencer

These counts reflect current FIE rules and may be subject to change. The apparatus or competition management software is responsible for initialising the correct count.

---

## 18. Message: control

**Topic:** `openpiste/{piste_id}/control`
**QoS:** 1
**Retained:** No

Published when a control event occurs. This topic is bidirectional — it carries commands from apparatus to software, from software or remote controls to apparatus, and match event notifications originating at the apparatus. A receiver that encounters an unknown command value SHOULD ignore it.

### Payload

```json
{
  "protocol": "OPP2",
  "command":  "MEDICAL",
  "side":     "left",
  "duration": 300,
  "source":   "apparatus",
  "ts":       1715539200456
}
```

### Fields

| Field | Type | Description |
|-------|------|-------------|
| `protocol` | string | Always `"OPP2"` |
| `command` | string | Command name — see defined values below |
| `side` | string | `"left"` or `"right"` — for side-specific commands (optional) |
| `duration` | integer | Duration in seconds — for MEDICAL command only (optional) |
| `source` | string | Origin: `"apparatus"`, `"software"`, `"remote"` (optional) |
| `ts` | integer | Timestamp when the command was issued — see Section 20 (optional) |

### Defined command values

| Command | Direction | Description |
|---------|-----------|-------------|
| `"NEXT"` | Apparatus → Software | Request next match or round |
| `"PREV"` | Apparatus → Software | Request previous match or round |
| `"END"` | Apparatus → Software | Signal end of match or round, awaiting ACK |
| `"MEDICAL"` | Apparatus → Software | Medical timeout granted; `side` and `duration` fields required |
| `"RESERVE"` | Apparatus → Software | Reserve fencer introduction; `side` field required |
| `"VIDEO_REVIEW_REQUEST"` | Apparatus → Software | Fencer requests video review; `side` field required |
| `"ACK"` | Software → Apparatus | Approve end of match or round |
| `"NAK"` | Software → Apparatus | Reject end of match or round |
| `"VIDEO_REVIEW_GRANTED"` | Software → Apparatus | Video review call granted; `side` field required |
| `"VIDEO_REVIEW_DENIED"` | Software → Apparatus | Video review call denied; `side` field required |
| `"BEGIN"` | Remote → Apparatus | Start the bout |
| `"HALT"` | Remote → Apparatus | Call halt |
| `"RESET"` | Remote → Apparatus | Reset the apparatus |
| `"VALIDATE"` | Remote → Apparatus | Confirm end of match |

Additional command values may be defined in future revisions without a protocol version change.

---

## 19. Field types and conventions

| Type | JSON representation | Notes |
|------|--------------------|----|
| Boolean | `true` / `false` | Never `"0"` / `"1"` or string-encoded |
| Integer | JSON number, no quotes | Scores, card counts, millisecond times |
| String | JSON string | Identifiers, names, nation codes, formatted times |
| Timestamp | JSON integer (64-bit) | See Section 20 for encoding convention |

**Formatted time strings** use `"M:SS"` or `"M:SS.cc"` format. Hundredths are mandatory when time is below 10 seconds, consistent with EFP1.1 convention.

**Nation codes** use IOC 3-letter codes (e.g. `"FRA"`, `"GBR"`, `"ITA"`).

**Missing fields:** If a field is not available it SHOULD be omitted rather than sent as null or empty string. Receivers MUST handle missing fields gracefully.

---

## 20. Timestamp conventions

### 20.1 UTC only

All timestamps in Level 2 are UTC. No local time, no timezone offsets, no daylight saving adjustments. Unix epoch milliseconds are by definition UTC — this is not a configuration choice, it is inherent to the format. Implementations MUST use UTC time sources and MUST NOT apply local timezone conversions.

### 20.2 Format

All timestamps are 64-bit unsigned integers. The upper byte (bits 63–56) carries a clock source flag. The lower 56 bits carry the time value in milliseconds.

| Bits | Field |
|------|-------|
| 63–56 | Clock source flag (upper byte) |
| 55–0 | Time value in milliseconds (lower 56 bits) |

### 20.3 Flag values

| Upper byte | Meaning | Lower 56 bits |
|------------|---------|---------------|
| `0x00` | NTP — UTC wall clock | Unix epoch milliseconds, NTP synchronised |
| `0x01` | Session — boot relative | Milliseconds since device boot (`millis()`) |
| `0x02`–`0xFF` | Reserved | — |

### 20.4 NTP timestamps

Current Unix epoch milliseconds are approximately `1.7 × 10¹²` (`0x0000018E...` in hex). The upper byte is naturally `0x00` for the foreseeable future. NTP timestamps therefore require no manipulation at the apparatus — the raw epoch millisecond value is correct.

### 20.5 Session timestamps

When NTP is unavailable, the apparatus SHOULD use milliseconds since device boot with the upper byte set to `0x01`:

```cpp
// NTP available — upper byte is naturally 0x00
uint64_t ts = (uint64_t)epochMillis;

// NTP not available
uint64_t ts = ((uint64_t)0x01 << 56) | (uint64_t)millis();
```

Session timestamps are useful for relative timing within a session but cannot be compared across devices or to wall-clock time.

### 20.6 Reading timestamps

```cpp
uint8_t  flag = (ts >> 56) & 0xFF;
uint64_t time = ts & 0x00FFFFFFFFFFFFFF;
// flag == 0x00: time is UTC Unix epoch milliseconds
// flag == 0x01: time is milliseconds since device boot
```

### 20.7 Video synchronisation

When using blade contact or lights timestamps to synchronise video overlays, both the apparatus and the video system SHOULD be synchronised to the same NTP server. Residual clock drift between devices is typically under 10ms on a well-managed local network.

---

## 21. Versioning and compatibility

### 21.1 Protocol identifier

Every message carries `"protocol": "OPP2"`. A receiver SHOULD check this field and MAY ignore messages with an unrecognised identifier.

### 21.2 Adding fields

New fields may be added to any message in a minor revision without changing the protocol identifier. JSON parsers silently ignore unknown keys, so existing receivers continue to operate correctly.

### 21.3 Breaking changes

Removing or renaming existing fields, or changing field types, constitutes a breaking change and requires a new protocol identifier (e.g. `"OPP3"`).

### 21.4 Adding control commands and phase types

New command values and new phase type values are not breaking changes. Receivers that encounter unknown values SHOULD ignore them.

---

## 22. Security

> **Open item — decision required before production deployment.**

Security for Level 2 has not yet been formally specified. The following considerations apply and will be resolved in a future revision:

**Asymmetric access model.** The appropriate model for most deployments is likely: subscribers (displays, monitors, video tools) may connect and subscribe without authentication on port 1883; publishers (apparatus, remote controls, competition software) SHOULD authenticate using MQTT username/password credentials over TLS on port 8883. This allows open read access while protecting the integrity of scoring data.

**Credential deployment.** For a club setup with a handful of devices, static credentials configured per device are acceptable. For a competition with many pistes and devices, a more automated credential management approach is needed. The operational burden of certificate and credential deployment at scale is a significant consideration and will influence the final recommendation.

**Local network isolation.** For deployments where authentication is not yet implemented, network isolation — restricting broker access to the local competition network — is the minimum acceptable control.

A formal security specification will be added in a future revision.

---

## 23. Open items

**Blade contact semantics.** The blade_contact message currently treats contact as a stateful on/off event. An alternative treats it as a momentary event — a single publish with no corresponding off message. The choice affects whether blade_contact should eventually become a retained message. This will be resolved based on feedback from video referee application developers.

**ACK/NAK state machine.** The exact behaviour expected of a Level 2 apparatus when it receives an ACK or NAK control command — and the full state machine around the Ending state — is not yet formally specified.

**JSON Schema.** A machine-readable JSON Schema for all message types is planned as a separate document at `schemas/opp2/` in the OpenPiste repository. Not yet published.

**Medical and reserve fields in EFP1.1.** The EFP1.1 fields `RMedical`, `LMedical`, `RReserve`, and `LReserve` are handled in Level 2 via the `medical` topic and the `RESERVE` control command respectively. The mapping is considered complete.

---

*OpenPiste Protocol Level 2 is released under the MIT licence.*
*Reference implementation and further documentation: https://openpiste.org*
