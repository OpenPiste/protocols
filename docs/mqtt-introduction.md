# MQTT — A Practical Introduction for Fencing Electronics

**Status:** Informational — not part of the OpenPiste Protocol specification
**Author:** Piet Wauters
**Repository:** https://github.com/OpenPiste/protocol
**Website:** https://openpiste.org

---

This document is for anyone familiar with fencing electronics — developers, armourers, club managers, federation officials — who has encountered the term MQTT and wants to understand what it is, why it matters, and whether it is a reasonable technology to build on. It is not a tutorial. It is an honest introduction.

---

## What is MQTT?

MQTT (Message Queuing Telemetry Transport) is a lightweight messaging protocol designed for connected devices. It was invented in 1999 by Andy Stanford-Clark of IBM and Arlen Nipper of Cirrus Link, originally to monitor oil pipelines over satellite links — an environment where bandwidth is scarce, connections are unreliable, and the software running on remote sensors must be as simple as possible.

Those constraints turn out to be remarkably similar to the constraints of fencing electronics: resource-limited microcontrollers, local WiFi networks that may not be perfect, and a need for multiple devices to share data in real time without complex coordination.

MQTT became an open standard in 2013 and an ISO/IEC international standard (ISO/IEC 20922) in 2016.

MQTT is one of the foundational protocols of the **Internet of Things (IoT)** — the broad term for the ecosystem of physical devices (sensors, actuators, controllers, displays) that are connected to networks and exchange data. Fencing electronics — scoring apparatus, remote controls, displays, weapon testers — are IoT devices in every meaningful sense of the term. The problems MQTT was designed to solve are exactly the problems fencing electronics face.

---

## Is this a niche technology?

No. MQTT is one of the most widely deployed IoT protocols in the world.

It is used by:
- **Facebook Messenger** — MQTT powers the mobile messaging infrastructure
- **Amazon AWS IoT** — the default protocol for connected device communication
- **Microsoft Azure IoT Hub** — supported as a primary protocol
- **Home Assistant and most home automation platforms** — the lingua franca of smart home devices
- **Industrial SCADA systems** — factory floor monitoring and control
- **Connected vehicles** — telemetry from cars, trucks, and logistics fleets
- **Medical devices** — remote patient monitoring systems
- **Smart energy grids** — meter reading and grid management

Client libraries exist for C, C++, Python, JavaScript, Java, Go, Rust, Delphi, and virtually every other language. The Arduino and ESP32 ecosystems have mature, well-maintained MQTT libraries that have been in production use for years.

When you implement MQTT in fencing electronics, you are building on the same technology that runs significant parts of the global IoT infrastructure. This is not an experiment.

---

## How does it work? The publish/subscribe model

MQTT uses a **publish/subscribe** model, which is fundamentally different from the point-to-point model used by Cyrano (EFP1.1) over UDP.

### The UDP model — point to point

With UDP, a sender addresses a specific machine and sends a packet directly to it. The scoring apparatus knows the IP address of the competition management software and sends its INFO messages there. If you want to add a scoreboard display, the apparatus needs to know the display's IP address and send to that too. If you want to add a piste monitor and a video referee tool, the apparatus needs to know all of them.

The sender is responsible for maintaining a list of all recipients and delivering to each one. As the ecosystem grows, the device gets more complex. And if a recipient is not listening when the packet arrives — because it just connected, or briefly lost network — the message is gone.

### The MQTT model — publish and subscribe

With MQTT, a sender **publishes** a message to a **topic** on a **broker**. The broker is a central relay — think of it as a community notice board. Anyone who has **subscribed** to that topic receives the message automatically.

The scoring apparatus publishes its lights state to `openpiste/17/apparatus/lights`. It does not know or care who is listening. One subscriber or twenty — the apparatus code does not change. The broker handles delivery to all of them.

This has a profound implication for embedded systems: **the complexity of managing multiple recipients moves entirely off the device and into the broker**. The ESP32 running the scoring apparatus does one thing — publish what it knows. It maintains no list of subscribers, sends no duplicate packets, and has no knowledge of the ecosystem around it. All of that coordination happens in the broker, which runs on hardware that can handle it comfortably.

A scoring device implemented today will work unchanged in an ecosystem ten times larger tomorrow. You add subscribers at the broker level, not at the device level.

### Subscribers receive only what they need

The reverse is equally important. With UDP broadcast, every device on the network receives every message and must discard what it does not need — wasting both network bandwidth and device processing time. A scoreboard receiving full Cyrano INFO strings must parse the entire message even if it only needs the score fields.

With MQTT, filtering happens at the broker before delivery. A scoreboard that only needs lights and score subscribes to exactly `openpiste/17/apparatus/lights` and `openpiste/17/apparatus/score` — and receives nothing else. A video referee tool that only needs blade contact timestamps subscribes to `openpiste/+/apparatus/blade_contact` across all pistes — and receives nothing else. Each subscriber declares exactly what it wants, and the broker enforces it. No wasted bandwidth, no wasted processing, no filtering code needed on the device.

---

## Comparison with UDP unicast, broadcast, and multicast

| | UDP unicast | UDP broadcast | UDP multicast | MQTT |
|---|---|---|---|---|
| **Addressing** | One specific IP address | All devices on the subnet | Group membership | Topic name on a broker |
| **Sender knows recipients?** | Yes — must know each IP | No — sends to all | Partial — group address | No — broker handles it |
| **Multiple recipients** | Sender must loop | All devices receive | Group members receive | Broker fans out automatically |
| **Late-joining subscriber** | Misses everything | Misses everything | Misses everything | Receives last retained value immediately |
| **Selective subscription** | Not possible | Not possible | Limited (group) | Full — subscribe to any topic pattern |
| **Delivery guarantee** | None | None | None | QoS 1: at least once |
| **Fixed IP addresses required** | Yes — for every device | No | Partial | Only the broker |
| **Current use in fencing** | Cyrano / EFP1.1 | Some display systems | Rare | OpenPiste |

The late-joining subscriber row is worth highlighting. With UDP, a scoreboard that reboots mid-bout has no way to get the current score until the apparatus happens to send the next periodic message. With MQTT retained messages, the broker holds the last published value on every topic and delivers it immediately to any new subscriber — the scoreboard reconstructs the current state the moment it reconnects, without waiting for anything.

---

## What is the broker, and how big does it need to be?

The broker is the only piece of infrastructure MQTT requires. For fencing, it is typically **Mosquitto** — an open source broker that is small, fast, and free.

Mosquitto running on a Raspberry Pi 4 can handle thousands of simultaneous connections and hundreds of thousands of messages per second. A fencing competition with 40 pistes publishing clock ticks once per second generates perhaps 200 messages per second total — a trivial load. Mosquitto on a modest laptop handles this without measurable CPU usage.

For a club setup, Mosquitto can run on the same laptop as the competition management software. For a major championship, a dedicated Raspberry Pi is more than sufficient. There is no requirement for cloud infrastructure, server farms, or paid services of any kind.

Mosquitto also has built-in **bridge** support — the ability to relay messages from a local broker to a remote cloud broker with a few lines of configuration. This is how live results reach the internet during a competition, without any changes to the local apparatus or software.

---

## What about reliability? UDP was chosen for a reason.

UDP was chosen for Cyrano in 2008 because it is lightweight and has low latency. Both remain true. But the trade-off — no delivery guarantee — was acceptable in 2008 because the ecosystem was simple: one apparatus, one software instance, point to point on a wired network.

MQTT runs over **TCP**, which provides reliable, ordered delivery. The concern with TCP in 2008 was overhead — connection setup, acknowledgements, retransmission. On a modern local WiFi network, this overhead is negligible. The latency difference between UDP and TCP on a local network is measured in microseconds, not milliseconds.

MQTT adds its own quality-of-service layer on top of TCP:
- **QoS 0** — fire and forget, like UDP. Used for clock ticks where a missed message is harmless.
- **QoS 1** — at least once delivery, with acknowledgement. Used for scores, lights, and control commands where a missed message would cause incorrect state.

The OpenPiste Protocol uses both, choosing the right level for each message type. High-frequency, low-consequence messages use QoS 0. State changes that must not be lost use QoS 1.

---

## Security

MQTT supports TLS encryption and username/password authentication. A broker can be configured so that anyone can subscribe (read) without credentials, while publishers (apparatus, remote controls, software) must authenticate. This maps naturally onto the fencing use case: open read access for displays and monitors, protected write access for devices that affect scoring.

For a local competition network with no external access, network isolation alone is often sufficient — the same security model as Cyrano over UDP.

---

## Summary

| Question | Answer |
|----------|--------|
| Is MQTT mature? | Yes — ISO standard since 2016, in production since 1999 |
| Is it widely used? | Yes — Facebook, Amazon, Microsoft, industrial systems |
| Does it require big infrastructure? | No — Mosquitto on a Raspberry Pi is more than enough |
| Is it harder to implement than UDP? | Marginally — one broker to configure, same libraries on the device |
| Does adding more subscribers require device changes? | No — the broker handles fan-out entirely |
| Can it reach the internet? | Yes — built-in bridge support in Mosquitto |
| Is it secure? | Yes — TLS and authentication built in |
| Is it appropriate for fencing electronics? | Yes — designed exactly for this class of problem |

---

## Further reading

- MQTT specification: https://mqtt.org
- Mosquitto broker: https://mosquitto.org
- Arduino/ESP32 MQTT library (PubSubClient): https://github.com/knolleary/pubsubclient
- OpenPiste Protocol specification: https://github.com/OpenPiste/protocol

---

*This document is released under the MIT licence.*
*OpenPiste: https://openpiste.org*
