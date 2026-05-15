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

# OpenPiste Protocol — Level 1
## EFP1.1 over MQTT

**Version:** Draft
**Date:** May 2026
**Author:** Piet Wauters
**Repository:** https://github.com/OpenPiste
**Website:** https://openpiste.org

---

## 1. Purpose

Level 1 defines how existing EFP1.1 (Cyrano) payloads are transported over MQTT instead of UDP. The payload is unchanged. No parsing changes are required on either the apparatus or the software side beyond swapping the transport layer.

This is a migration bridge. It allows existing Cyrano-compatible equipment and software to participate in an MQTT network with minimal engineering effort, and enables developers to build MQTT-native subscribers (displays, piste monitors, video tools) that consume live scoring data from existing infrastructure without waiting for that infrastructure to adopt the full Level 2 protocol.

A known real-world use case: a competition running existing scoring apparatus and existing competition management software, with new MQTT-native displays developed independently. The bridge makes this possible without any changes to the existing equipment or software.

---

## 2. Relationship to EFP1.1

The EFP1.1 specification (technical name of the Cyrano protocol, version 1.1, October 2019) defines the payload format, field semantics, message types (HELLO, DISP, ACK, NAK, INFO, NEXT, PREV), and apparatus behaviour. All of that is unchanged in Level 1. This document specifies only the transport mapping: how EFP1.1 messages are carried over MQTT.

Implementers who are not familiar with EFP1.1 should read that specification alongside this document.

---

## 3. Why MQTT instead of UDP

EFP1.1 was designed for UDP in 2008. The choice made sense at the time: UDP is lightweight, has low overhead, and requires no connection setup. The trade-offs — no guaranteed delivery, no broker, fixed IP addressing — were acceptable for a point-to-point link between one apparatus and one software instance on a controlled local network.

MQTT addresses two practical problems that have emerged as fencing electronics ecosystems have grown:

**No fixed IP addresses.** In EFP1.1, the software must know the IP address of every apparatus, and every apparatus must know the IP address of the software. These addresses must be statically assigned or carefully managed with DHCP reservations. In an MQTT network, the only address that every device needs to know is the broker's. Apparatus and subscribers find each other through the broker without any knowledge of each other's addresses.

**Retained state for late-joining subscribers.** UDP delivers messages to whoever is listening at the moment of transmission. A display that reboots mid-bout, or a subscriber that connects after the bout has started, has no way to get the current state without waiting for the next periodic INFO message. MQTT retained messages solve this: the broker holds the last published value on the topic and delivers it immediately to any new subscriber.

Note: UDP broadcast can address the fan-out problem (delivering to multiple receivers simultaneously). The MQTT advantages listed above are independent of fan-out.

---

## 4. Transport

### 4.1 Broker

Any MQTT 3.1.1 compliant broker. Mosquitto is recommended — it is open source, lightweight, and runs on a laptop or Raspberry Pi with minimal configuration.

### 4.2 Broker discovery

For club and small competition use, the broker host SHOULD be made discoverable via mDNS under the hostname:

```
openpiste.local
```

This means any device on the local network can reach the broker at `openpiste.local:1883` without any IP address configuration. All OpenPiste-compatible devices SHOULD use this hostname as their default broker address.

For larger competition setups with managed switches or multiple VLANs, mDNS may not propagate reliably across network boundaries. In these cases a static IP address or DHCP reservation for the broker is recommended, and the `openpiste.local` hostname may be configured in local DNS.

### 4.3 NTP

The broker host SHOULD also run an NTP server. This allows all devices on the local network to synchronise their clocks to UTC without requiring internet access. On Linux, `chrony` is recommended — it is lightweight and can serve NTP to local clients while itself operating without an upstream internet time source if necessary.

When all devices synchronise to the same local NTP server, timestamps in Level 1 messages (see Section 7) are comparable across apparatus, software, and subscriber devices. This is particularly important for video referee applications.

### 4.4 Port

Standard MQTT port 1883 (unencrypted) or 8883 (TLS).

### 4.5 QoS

QoS 0 (at most once). This matches the original UDP behaviour and the high-frequency nature of INFO messages.

### 4.6 Retained messages

Yes. The broker retains the last published message on the topic so a late-joining subscriber immediately receives the current state.

---

## 5. Topic structure

```
openpiste/{piste_id}/efp1
```

The `{piste_id}` segment matches the Piste field in the EFP1.1 message (field 3). It may be a number, a name, or a colour — whatever the apparatus is configured to use.

Examples:
```
openpiste/17/efp1
openpiste/podium/efp1
openpiste/rouge/efp1
```

To subscribe to all pistes:
```
openpiste/#
```

To subscribe to a specific piste:
```
openpiste/17/#
```

---

## 6. Payload

The payload is the EFP1.1 message string, exactly as it would have been sent over UDP. UTF-8 encoded. No framing bytes added.

Example INFO message:
```
|EFP1.1|INFO|17|efj-eq|1|A32|12|2|10:30|3:00|I|S||H|132|J.Smith|GBR|%|28|P.Martin|FRA|8|V|0|1|1|0|0|N|%|32|B.Panini|ITA|6|D|0|1|0|0|0|N|%|
```

Example HELLO message (software → apparatus direction):
```
|EFP1.1|HELLO|17|efj-eq|%|
```

---

## 7. Optional timestamp extension

### 7.1 Motivation

EFP1.1 contains no timestamp. In the original UDP implementation this was acceptable because messages were sent point-to-point and latency was predictable. In an MQTT network, messages may be consumed by video referee applications, logging systems, or analytics tools where knowing the exact moment an event occurred at the apparatus is important.

A timestamp extension is defined for Level 1. It is **optional** — existing implementations that do not produce or consume it continue to operate correctly.

### 7.2 Format

The timestamp is appended as an additional field after the final `%` separator of the EFP1.1 message. It is a 64-bit unsigned integer encoding both the time value and a clock source flag in the upper byte.

Standard EFP1.1 message (no timestamp):
```
|EFP1.1|INFO|17|efj-eq|...|%|...|%|...|%|
```

Level 1 message with optional timestamp:
```
|EFP1.1|INFO|17|efj-eq|...|%|...|%|...|%|1715539200123|
```

### 7.3 Timestamp encoding

The 64-bit integer is structured as follows:

| Bits | Field | Description |
|------|-------|-------------|
| 63–56 | Flag (upper byte) | Clock source identifier |
| 55–0 | Time value (lower 56 bits) | Time value in milliseconds |

**Flag values:**

| Upper byte | Meaning | Lower 56 bits |
|------------|---------|---------------|
| `0x00` | NTP — UTC wall clock | Unix epoch milliseconds, NTP synchronised |
| `0x01` | Session — device boot relative | Milliseconds since device boot (`millis()`) |
| `0x02`–`0xFF` | Reserved | — |

All timestamps are UTC. No local time, no timezone offsets, no daylight saving adjustments.

**Construction on the device:**

```cpp
// NTP available — upper byte is naturally 0x00 for current epoch values
uint64_t ts = (uint64_t)epochMillis;

// NTP not available — set upper byte to 0x01
uint64_t ts = ((uint64_t)0x01 << 56) | (uint64_t)millis();
```

**Reading on the receiver:**

```cpp
uint8_t  flag = (ts >> 56) & 0xFF;
uint64_t time = ts & 0x00FFFFFFFFFFFFFF;
// flag == 0x00: time is UTC Unix epoch milliseconds
// flag == 0x01: time is milliseconds since device boot
```

**Note:** Current Unix epoch milliseconds are approximately `1.7 × 10¹²` (`0x0000018E...` in hex). The upper byte is naturally `0x00` for the foreseeable future, so NTP timestamps require no manipulation at the apparatus.

### 7.4 Compatibility

An EFP1.1 parser that reads fields positionally and stops at the final `%` will ignore the appended timestamp entirely. No existing parser is broken by its presence.

A Level 1 subscriber that wishes to consume the timestamp reads the content after the final `%` separator. If the field is absent or empty, the timestamp is unavailable.

### 7.5 Clock source

The timestamp reflects the apparatus local clock at the moment the event was detected — not the moment the message was published to the broker. See Section 4.3 for NTP server recommendations.

---

## 8. Bridging from UDP

A Level 1 bridge is a transparent bidirectional relay between the EFP1.1 UDP network and the MQTT broker. It handles traffic in both directions:

**Apparatus → MQTT (apparatus INFO, NEXT, PREV messages):**
1. Listen for EFP1.1 UDP datagrams on port 50100.
2. Optionally append a local timestamp after the final `%` (see Section 7).
3. Publish the payload to `openpiste/{piste_id}/efp1` on the MQTT broker.

The piste identifier for the MQTT topic is extracted from field 3 of the EFP1.1 message, or may be configured statically.

**Competition management software → MQTT (HELLO, DISP, ACK, NAK messages):**
1. Listen for EFP1.1 UDP datagrams sent by the competition management software.
2. Publish the payload to `openpiste/{piste_id}/efp1` on the MQTT broker.

**MQTT → UDP (for apparatus or software that only speaks UDP):**
1. Subscribe to `openpiste/{piste_id}/efp1`.
2. Forward received messages as UDP datagrams to the configured apparatus or software IP address.

This symmetric design means that MQTT-native subscribers (displays, piste monitors) receive all traffic from both the apparatus and the competition management software, without either needing to be modified.

A reference bridge implementation is available at https://github.com/OpenPiste.

---

## 9. Security

> **Open item — decision required before production deployment.**

Security for Level 1 has not yet been formally specified. The following considerations apply:

EFP1.1 over UDP uses no encryption or authentication. Existing competition infrastructure operates on this basis. Requiring TLS or authentication for Level 1 MQTT would make it harder to adopt than the protocol it replaces, which defeats the purpose of Level 1 as a migration bridge.

The likely appropriate position for Level 1 is: operate on port 1883 without TLS or authentication, with network isolation as the security boundary (the MQTT broker is only accessible on the local competition network). This is equivalent to the security model of EFP1.1.

A formal security recommendation will be added in a future revision. Implementers deploying Level 1 on networks accessible beyond the local venue should apply additional network-level controls.

---

## 10. Limitations

Level 1 inherits all limitations of EFP1.1:

- All values are string-encoded, including integers and booleans.
- Fields are positional — adding new fields in the middle of a section breaks existing parsers.
- No native support for typed messages with different QoS or retention policies.
- No structured event messages for blade contact, unwillingness-to-fight timer, or extensible remote control.

These limitations are the motivation for Level 2 — the native JSON protocol. Level 1 is a migration bridge, not a long-term target.

---

## 11. Summary

| Property | Value |
|----------|-------|
| Payload | EFP1.1 string, unchanged |
| Transport | MQTT 3.1.1 |
| Topic | `openpiste/{piste_id}/efp1` |
| QoS | 0 |
| Retained | Yes |
| Timestamp | Optional — appended after final `%`, 64-bit integer with upper-byte flag |
| Broker hostname | `openpiste.local` (mDNS, recommended) |
| Broker port | 1883 (unencrypted) / 8883 (TLS) |
| NTP | Broker host SHOULD run a local NTP server |
| Security | Open item — see Section 9 |

---

*OpenPiste Protocol is released under the MIT licence.*
*Reference implementation and further documentation: https://openpiste.org*
