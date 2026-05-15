# OpenPiste Protocol

**Status:** Draft — working towards v1.0
**Author:** Piet Wauters
**Website:** https://openpiste.org

---

Fencing electronics have long relied on two communication protocols: EFP1.1 (known as Cyrano), the dominant standard for communication between scoring apparatus and competition management software, and RS422-FPA, a serial protocol for driving external displays and scoreboards. Both were designed for their era and both served the community well. EFP1.1 has been in use since 2008. RS422-FPA traces its roots to 1995.

The world around them has changed. MQTT and JSON are now the lingua franca of connected devices. Libraries exist for every platform from microcontrollers to cloud services. The IoT ecosystem has solved, at scale, the same problems fencing electronics face: reliable message delivery, multiple subscribers, late-joining clients, structured extensible data. There is no longer a compelling reason to maintain a bespoke binary or CSV protocol when open, well-supported alternatives exist.

At the same time, there is a substantial installed base of EFP1.1-compatible apparatus and software. A new protocol that ignores this reality will not be adopted. Migration must be possible without requiring clubs and federations to replace working equipment overnight.

The **OpenPiste Protocol** is a proposal for a modern, open communication standard for fencing electronics. It is structured in two levels:

**Level 1** addresses the transition. It defines how existing EFP1.1 payloads can be transported over MQTT without any change to the payload itself. A bridge relays messages between the existing UDP network and an MQTT broker. Existing apparatus and software require no modification. New MQTT-native subscribers — displays, piste monitors, video tools — can immediately consume live scoring data from existing infrastructure.

**Level 2** is the destination. It defines a native JSON protocol designed from the ground up for MQTT, drawing on the field semantics of EFP1.1 and the message architecture of RS422-FPA while leaving behind the encoding constraints of both. It uses typed values, purpose-specific messages, retained state, and millisecond-precision timestamps. It is implementable on an ESP32 with standard open source libraries.

This is a working proposal, not a ratified standard. It is published in the hope that it will be useful, reviewed, and improved by the fencing electronics community.

**Remark**: A middle step was considered — parsing EFP1.1 into JSON using the same field names as keys and publishing the result over MQTT. This was discarded for four reasons: it adds no meaningful value over Level 1, since consumers must still understand EFP1.1 field semantics; it requires two parsing steps (CSV at the bridge, JSON at every consumer) making the bridge more complex than necessary; it inherits EFP1.1's structural limitations — string-encoded integers, awkward field names, flat structure — in a format where consumers would reasonably expect clean typed data; and it creates a false impression of a modern protocol while delivering the constraints of a legacy one. Level 1 is honest about what it is. Level 2 makes a clean break. The middle step is neither.

---

## Documents

| Document | Description |
|----------|-------------|
| [docs/level1.md](docs/level1.md) | Level 1 — EFP1.1 over MQTT |
| [docs/level2.md](docs/level2.md) | Level 2 — Native JSON over MQTT |

---

## How to participate

This specification is developed openly. All feedback is welcome — whether you build scoring apparatus, competition software, displays, video referee tools, or you simply use fencing electronics and have an opinion about how they should work.

**To contribute:**

- **Open an issue** to ask a question, report an error, or propose a change
- **Start a discussion** in the [Discussions](../../discussions) tab for broader topics
- **Submit a pull request** if you want to propose specific wording changes to the spec

There is no requirement to be a developer to contribute. Domain knowledge from armourers, referees, competition organisers, and federation officials is equally valuable.

Please read [CONTRIBUTING.md](CONTRIBUTING.md) before opening your first issue or pull request.

---

## Status

| Level | Status | Blocking issues |
|-------|--------|----------------|
| Level 1 | Draft | Security model (open item) |
| Level 2 | Draft | Blade contact semantics; ACK/NAK state machine; Security model; JSON Schema |

---

## Licence

MIT — see [LICENSE](LICENSE)
