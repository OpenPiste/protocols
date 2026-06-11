# OpenPiste Remote Control Protocol (OPRCP)
## A Level 2 sub-specification for remote control of fencing scoring apparatus

**Status:** Draft — working towards v1.0
**Date:** May 2026
**Author:** Piet Wauters
**Protocol identifier:** `OPRCP1`
**Repository:** https://github.com/OpenPiste
**Website:** https://openpiste.org

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [Design principles](#2-design-principles)
3. [Relationship to OpenPiste Level 2](#3-relationship-to-openpiste-level-2)
4. [Architecture](#4-architecture)
   - 4.1 Command model
   - 4.2 State response model
   - 4.3 Pairing model
5. [Binary encoding](#5-binary-encoding)
   - 5.1 Frame structure
   - 5.2 Flags — UNDO and CYCLE
   - 5.3 Parameter encoding
   - 5.4 Time encoding
6. [JSON encoding](#6-json-encoding)
   - 6.1 Field mapping
   - 6.2 Examples
7. [Command reference](#7-command-reference)
   - 7.1 CLOCK (0x01)
   - 7.2 SCORE (0x02)
   - 7.3 CARD (0x03)
   - 7.4 MATCH (0x04)
   - 7.5 BREAK (0x05)
   - 7.6 COMPETITION (0x06)
   - 7.7 SYSTEM (0x07)
   - 7.8 SETTINGS (0x08)
   - 7.9 VENDOR (0x7F)
8. [Transport bindings](#8-transport-bindings)
   - 8.1 UDP
   - 8.2 MQTT
   - 8.3 Bluetooth LE
   - 8.4 Physical buttons
   - 8.5 Non-defined transports
9. [Pairing specification](#9-pairing-specification)
10. [Capability document](#10-capability-document)
11. [State response coverage](#11-state-response-coverage)
12. [Versioning and extensibility](#12-versioning-and-extensibility)
13. [Open items](#13-open-items)

---

## 1. Introduction

The OpenPiste Remote Control Protocol (OPRCP) defines how remote controls communicate with OpenPiste-compatible scoring apparatus. It is a sub-specification of OpenPiste Level 2, scoped specifically to the command channel between a referee or operator's remote control and a scoring apparatus. It does not define a general-purpose remote control standard — it is an OpenPiste protocol for OpenPiste apparatus.

Remote control of a scoring apparatus is a different problem from score distribution. A referee pressing a button needs the device to respond immediately and reliably in a sports hall with no network infrastructure. The same protocol must also work over MQTT in a fully connected competition venue. It must be implementable on a device as simple as a microcontroller with a radio module, and as rich as a touchscreen mobile application.

OPRCP solves this by defining a transport-agnostic command model: a compact binary encoding for constrained transports, a JSON encoding for MQTT and network transports, and a set of defined transport bindings. The command semantics are identical regardless of transport.

---

## 2. Design principles

**Commands express operator action, not operator belief about state.** A command expresses what the operator wants to happen — not what the operator believes the current state to be. `SCORE INCREMENT` means "add a point." It carries no assumption that the score is currently N, no target value, no condition. `CLOCK START` means "start the clock" — it does not mean "the clock should now be running." The apparatus receives the command, validates it against its own authoritative state, and decides whether and how to apply it. The distinction matters: a command that says "add a point" is always well-formed; a command that says "set score to 5" encodes an assumption about state that the remote has no authority to make.

**Remotes do not maintain state.** A remote MUST NOT maintain a local model of apparatus state derived from its own commands. It is incorrect, for example, to track the score locally by counting INCREMENT and DECREMENT commands issued, or to assume the clock is running because START was pressed. All state is authoritative on the apparatus and published via OpenPiste Level 2. A smart remote that displays apparatus state subscribes to those Level 2 publications — it does not compute or infer state from its own command history.

**No addressing in commands.** Commands carry no source or destination address. Addressing is the responsibility of the pairing mechanism and the transport layer. The protocol assumes an established pairing between a remote and a scoring apparatus.

**No acknowledgement in the protocol.** Delivery confirmation is the responsibility of the transport layer where available. The authoritative confirmation of a command's effect is the subsequent state update published by the apparatus via OpenPiste Level 2. Smart remotes that display apparatus state subscribe to those state topics — they do not rely on command-level acknowledgements.

**Dual encoding, single semantic model.** Binary encoding is compact and suitable for constrained transports. JSON encoding is human-readable and suitable for MQTT and network transports. Both encodings express identical semantics. The binary form is canonical; the JSON form is a defined binding.

**UNDO and CYCLE as structural modifiers.** Rather than defining separate inverse commands or enumerated cycle variants, UNDO and CYCLE are flag bits in the binary frame and explicit fields in the JSON encoding. Every command automatically has an invertible form where applicable, without expanding the command taxonomy.

**Offline operation is a first-class requirement.** Commands in the COMPETITION class require an active competition management link. All other commands operate fully locally. If the MQTT broker or competition management software is unavailable, the apparatus continues to function normally for all non-COMPETITION operations.

---

## 3. Relationship to OpenPiste Level 2

OPRCP is a Level 2 sub-specification. It defines the command channel — the path from operator to apparatus. OpenPiste Level 2 defines the state channel — the path from apparatus to all subscribers, including smart remotes.

The relationship is asymmetric by design:

```
Remote  --[OPRCP command]-->  Apparatus  --[OPP2 state/events]-->  All subscribers
                                                                    (including smart remotes)
```

A smart remote that wants to display current apparatus state subscribes to the relevant OpenPiste Level 2 topics — `score`, `clock`, `state`, and others as needed. It does not receive state via OPRCP. OPRCP is a one-way command channel.

When the transport is MQTT, OPRCP commands are published to a dedicated subtopic within the apparatus topic hierarchy. See Section 8.2 for the topic structure.

---

## 4. Architecture

### 4.1 Command model

An OPRCP command is an atomic expression of operator action directed at a paired scoring apparatus. Commands are:

- **Unidirectional** — remote to apparatus only
- **Stateless** — a command carries no assumption about current apparatus state, and the remote draws no conclusions about state from commands it has sent
- **Action-based** — `SCORE INCREMENT` not `SET SCORE TO 5`
- **Unconditional** — commands carry no preconditions or guards

The apparatus receives a command, validates it in the context of its own authoritative state, applies it if valid, and publishes the resulting state via OpenPiste Level 2. A command that is invalid in the current state — for example, `CLOCK START` when no bout is active — is silently ignored or logged. The apparatus does not send a rejection to the remote.

### 4.2 State response model

OPRCP defines no state response mechanism. The apparatus response to a command is the state update it publishes to OpenPiste Level 2 topics. Smart remotes subscribe to those topics independently of the command channel.

This has two practical consequences:

- On MQTT transport, a smart remote subscribes to `openpiste/{piste_id}/+` for state updates and publishes to `openpiste/{piste_id}/remote/{remote_id}` for commands. These are separate topic paths.
- On UDP transport, the apparatus sends a Level 2 JSON state snapshot to the remote's IP address and port after processing any command. The payload format is the same JSON the apparatus publishes to the relevant Level 2 state topics.

If no state update arrives after a command, the cause is transport failure, not command rejection. Smart remote implementations SHOULD display a stale-data indicator after a configurable timeout rather than assuming command success or inferring state from the command itself.

Section 11 maps every OPRCP command class to the Level 2 topics the apparatus is expected to publish in response.

### 4.3 Pairing model

OPRCP assumes an established pairing between a remote and a scoring apparatus before any command is sent. A paired remote has an unambiguous delivery path to exactly one apparatus. Commands carry no addressing information.

Pairing is transport-specific. The pairing mechanisms for each defined transport are described in Section 9. The SYSTEM class provides protocol-level support for remote identity and pairing lifecycle management — see Section 7.7.

---

## 5. Binary encoding

### 5.1 Frame structure

Every OPRCP command is encoded as a 32-bit unsigned integer.

```
 31      24 23 22 21    16 15             0
┌──────────┬──┬──┬────────┬───────────────┐
│  class   │U │C │  cmd   │   parameter   │
│  8 bits  │1 │1 │ 6 bits │   16 bits     │
└──────────┴──┴──┴────────┴───────────────┘
```

| Bits | Field | Description |
|------|-------|-------------|
| 31–24 | class | Command class (8 bits, 0x00–0xFF) |
| 23 | U | UNDO flag — 1 = inverse/undo this command |
| 22 | C | CYCLE flag — 1 = advance to next value in sequence |
| 21–16 | cmd | Command within class (6 bits, 0–63) |
| 15–0 | parameter | Command parameter (16 bits) |

The frame is transmitted big-endian (most significant byte first).

### 5.2 Flags — UNDO and CYCLE

**UNDO flag (bit 23)**

When set, the command is the inverse of its normal form. The inverse semantics are defined per command in Section 7. Examples:

| Normal command | With UNDO flag |
|----------------|----------------|
| `CLOCK START` | `CLOCK STOP` |
| `SCORE INCREMENT` | `SCORE DECREMENT` |
| `CARD YELLOW` | Remove last yellow card |
| `MATCH NEXT_PERIOD` | `MATCH PREV_PERIOD` |

Where UNDO is not applicable — for example `CLOCK RESET`, `MATCH RESET`, all SETTINGS commands — the flag is ignored. The command reference in Section 7 states explicitly whether UNDO applies for each command.

**CYCLE flag (bit 22)**

When set, the apparatus advances the current value of the addressed setting or mode to the next in a predefined sequence, wrapping at the end. The parameter field is ignored when the CYCLE flag is set.

CYCLE applies to SETTINGS commands and to `CLOCK START` (where it means toggle — see Section 7.1). On all other commands the flag is ignored.

Example cycle sequences:

| Command | Cycle sequence |
|---------|----------------|
| `CLOCK START` | start if stopped → stop if running → start if stopped … |
| `SETTINGS WEAPON` | épée → foil → sabre → automatic → épée |
| `SETTINGS MATCH_TYPE` | pool → DE → teams → training → pool |
| `SETTINGS PERIOD_COUNT` | 1 → 2 → 3 → 1 |

Both flags may be set simultaneously only where both are applicable. In practice this situation does not arise.

### 5.3 Parameter encoding

The 16-bit parameter field carries command-specific data. Its interpretation depends on the command class and command.

**Side encoding** — used by SCORE, CARD, and BREAK commands:

| Parameter high byte | Meaning |
|--------------------|---------|
| `0x01` | Left fencer |
| `0x02` | Right fencer |
| `0x00` | Both / not applicable |

The parameter low byte is unused (set to `0x00`) for side-encoded commands.

**Remote ID encoding** — used by SYSTEM PING:

The full 16-bit parameter field carries the remote identifier. `0x0000` means anonymous. See Section 9 for remote ID assignment guidance.

**Time encoding** — used by CLOCK SET and SETTINGS PERIOD_DURATION:

See Section 5.4.

**Settings value encoding** — used by SETTINGS commands when the CYCLE flag is not set:

| Command | Parameter high byte | Parameter low byte |
|---------|--------------------|--------------------|
| WEAPON | unused (0x00) | 0x01 épée, 0x02 foil, 0x03 sabre, 0x04 automatic |
| MATCH_TYPE | unused (0x00) | 0x01 pool, 0x02 DE, 0x03 teams, 0x04 training |
| PERIOD_COUNT | unused (0x00) | count value (1–9) |
| PERIOD_DURATION | time encoding — see Section 5.4 ||

### 5.4 Time encoding

The 16-bit parameter field encodes time values for `CLOCK SET` and `SETTINGS PERIOD_DURATION` using a dual-resolution scheme.

```
 15  14                              0
┌───┬────────────────────────────────┐
│ R │         time value             │
│1b │          15 bits               │
└───┴────────────────────────────────┘
```

| Bit 15 | Resolution | Range | Unit |
|--------|-----------|-------|------|
| `0` | Seconds | 0–32767 s | 1 second |
| `1` | Centiseconds | 0–999 | 0.01 second (10 ms) |

The boundary is 10 seconds (1000 centiseconds). Times of 10 seconds and above use second resolution. Times below 10 seconds use centisecond resolution.

Examples:

| Time | Bit 15 | Value | Encoded (hex) |
|------|--------|-------|---------------|
| 3:00 (180 s) | 0 | 180 | `0x00B4` |
| 1:30 (90 s) | 0 | 90 | `0x005A` |
| 10 s | 0 | 10 | `0x000A` |
| 9.99 s | 1 | 999 | `0x83E7` |
| 0.50 s | 1 | 50 | `0x8032` |
| 0.00 s | 1 | 0 | `0x8000` |

---

## 6. JSON encoding

### 6.1 Field mapping

The JSON encoding maps the binary frame fields to named JSON fields. It is used on MQTT and other text-capable transports.

| Binary field | JSON field | Type | Notes |
|-------------|-----------|------|-------|
| class | `"class"` | string | Class name, e.g. `"CLOCK"` |
| U flag | `"undo"` | boolean | Omitted when false |
| C flag | `"cycle"` | boolean | Omitted when false |
| cmd | `"cmd"` | string | Command name, e.g. `"START"` |
| parameter (side) | `"side"` | string | `"left"` or `"right"`, omitted if not applicable |
| parameter (time) | `"time_s"` or `"time_cs"` | integer | Seconds or centiseconds depending on resolution |
| parameter (remote ID) | `"remote_id"` | integer | SYSTEM PING only |
| parameter (settings value) | `"value"` | string or integer | Setting-specific |

Every JSON command also carries the OPRCP protocol identifier and a sequence counter for deduplication:

| JSON field | Type | Description |
|-----------|------|-------------|
| `"protocol"` | string | Always `"OPRCP1"` |
| `"seq"` | integer | Sender sequence counter, unsigned 32-bit, incremented per command |

### 6.2 Examples

**SCORE INCREMENT — left fencer**

Binary: `0x02 00 00 00 01 00`

```json
{
  "protocol": "OPRCP1",
  "seq":      142,
  "class":    "SCORE",
  "cmd":      "INCREMENT",
  "side":     "left"
}
```

**SCORE DECREMENT — left fencer (via UNDO flag)**

Binary: `0x02 80 00 00 01 00`

```json
{
  "protocol": "OPRCP1",
  "seq":      143,
  "class":    "SCORE",
  "cmd":      "INCREMENT",
  "undo":     true,
  "side":     "left"
}
```

**CLOCK TOGGLE (via CYCLE flag on START)**

Binary: `0x01 40 00 00 00`

```json
{
  "protocol": "OPRCP1",
  "seq":      144,
  "class":    "CLOCK",
  "cmd":      "START",
  "cycle":    true
}
```

**CLOCK SET — 1 minute 30 seconds**

Binary: `0x01 00 03 00 5A`

```json
{
  "protocol": "OPRCP1",
  "seq":      145,
  "class":    "CLOCK",
  "cmd":      "SET",
  "time_s":   90
}
```

**CLOCK SET — 9.50 seconds**

Binary: `0x01 00 03 83 E6`

```json
{
  "protocol": "OPRCP1",
  "seq":      146,
  "class":    "CLOCK",
  "cmd":      "SET",
  "time_cs":  950
}
```

**SETTINGS WEAPON CYCLE**

Binary: `0x08 40 00 00 00`

```json
{
  "protocol": "OPRCP1",
  "seq":      147,
  "class":    "SETTINGS",
  "cmd":      "WEAPON",
  "cycle":    true
}
```

**SYSTEM PING — remote ID 163**

Binary: `0x07 00 00 00 A3`

```json
{
  "protocol":  "OPRCP1",
  "seq":       1,
  "class":     "SYSTEM",
  "cmd":       "PING",
  "remote_id": 163
}
```

---

## 7. Command reference

### 7.1 CLOCK — class 0x01

Controls the bout stopwatch.

| cmd | cmd value | UNDO | CYCLE | Parameter | Description |
|-----|-----------|------|-------|-----------|-------------|
| START | 0x00 | STOP | TOGGLE | unused | Start the stopwatch |
| RESET | 0x01 | — | — | unused | Reset to period default duration |
| SET | 0x02 | — | — | time encoding (Section 5.4) | Set stopwatch to specific time |

`START` with no flags starts the stopwatch. With the UNDO flag it stops the stopwatch. With the CYCLE flag it toggles — starting if stopped, stopping if running. The CYCLE/toggle form is provided for physical remotes with a single clock button that have no knowledge of apparatus state. Smart remotes that display apparatus state SHOULD use `START` and `START+UNDO` rather than `START+CYCLE`, as the explicit forms make the intended action unambiguous.

`RESET` resets the stopwatch to the duration defined by `SETTINGS PERIOD_DURATION`, or to the weapon and match-type default if no duration has been configured.

---

### 7.2 SCORE — class 0x02

Controls fencer scores.

| cmd | cmd value | UNDO | CYCLE | Parameter | Description |
|-----|-----------|------|-------|-----------|-------------|
| INCREMENT | 0x00 | DECREMENT | — | high byte: side | Increment score by 1 |

The low byte of the parameter is unused and MUST be set to `0x00`.

There is no SET command. Absolute score assignment is not permitted via remote control. Corrections are made by issuing `INCREMENT` with the UNDO flag. This eliminates an entire class of remote control error and enforces the principle that remotes express actions, not state.

---

### 7.3 CARD — class 0x03

Issues or removes cards and records unwillingness to fight.

| cmd | cmd value | UNDO | CYCLE | Parameter | Description |
|-----|-----------|------|-------|-----------|-------------|
| YELLOW | 0x00 | Remove last yellow | — | high byte: side | Issue yellow card |
| RED | 0x01 | Remove last red | — | high byte: side | Issue red card |
| BLACK | 0x02 | Remove last black | — | high byte: side | Issue black card |
| P_YELLOW | 0x03 | Remove last P-yellow | — | high byte: side | Issue P-yellow card *(deprecated)* |
| P_RED | 0x04 | Remove last P-red | — | high byte: side | Issue P-red card |
| P_BLACK | 0x05 | Remove last P-black | — | high byte: side | Issue P-black card |
| UNWILLINGNESS | 0x06 | Cancel | — | high byte: side | Record unwillingness to fight |

The low byte of the parameter is unused and MUST be set to `0x00`.

Card sequence within a bout is tracked by the scoring apparatus, not the remote. The remote issues the card type and side only. The apparatus applies the applicable FIE rules and publishes the resulting card state via OpenPiste Level 2.

UNDO means "remove the most recent card of this type issued to this fencer." The apparatus resolves this against its own card history.

`UNWILLINGNESS` records that the referee has judged a fencer to be unwilling to fight. Under current FIE rules the apparatus can determine the resulting penalty autonomously from the unwillingness to fight state — see the `uw2f` message in OpenPiste Level 2. P-cards for passivity are therefore not normally issued via the CARD class. The P-card commands (P_YELLOW, P_RED, P_BLACK) are retained for situations where a referee needs to override or correct the apparatus's autonomous determination, and for compatibility with apparatus that do not implement autonomous UW2F tracking.

The offence classification for electronic score sheets is handled at competition management level and is not carried in the OPRCP command.

> **Deprecation notice:** `P_YELLOW` is deprecated as of the 2026/2027 season. It is retained in this specification for backwards compatibility with existing apparatus. Implementations SHOULD continue to accept it. It will be removed in a future revision of this specification.

---

### 7.4 MATCH — class 0x04

Controls local match flow. All commands in this class operate fully locally — no competition management link is required.

| cmd | cmd value | UNDO | CYCLE | Parameter | Description |
|-----|-----------|------|-------|-----------|-------------|
| NEXT_PERIOD | 0x00 | PREV_PERIOD | — | unused | Advance to next period |
| RESET | 0x01 | — | — | unused | Reset apparatus to default state |
| PRIORITY_LEFT | 0x02 | Remove priority | — | unused | Assign priority to left fencer |
| PRIORITY_RIGHT | 0x03 | Remove priority | — | unused | Assign priority to right fencer |
| PRIORITY_AUTO | 0x04 | Remove priority | — | unused | Random priority draw by apparatus |

`NEXT_PERIOD` advances the period counter. The apparatus determines the consequences — clock reset, score carry-over rules — based on the match type and current configuration.

`RESET` returns the apparatus to its default state: score zero, cards cleared, clock set to period default duration, period counter reset. This is a local recovery operation. It does not signal competition management software.

`PRIORITY_LEFT` and `PRIORITY_RIGHT` assign priority to the named fencer. Priority is an exclusive state — only one fencer may hold it. Issuing priority to one side implicitly clears it from the other.

`PRIORITY_AUTO` instructs the apparatus to perform a random priority draw and assign the result. UNDO removes priority from whichever side was drawn — it does not re-draw. If a re-draw is required, the referee issues `PRIORITY_AUTO` again.

---

### 7.5 BREAK — class 0x05

Records a match interruption and its reason.

| cmd | cmd value | UNDO | CYCLE | Parameter | Description |
|-----|-----------|------|-------|-----------|-------------|
| VIDEO | 0x00 | Cancel break | — | unused | Video review break |
| MEDICAL | 0x01 | Cancel break | — | unused | Medical stoppage |
| EQUIPMENT | 0x02 | Cancel break | — | unused | Equipment check |
| OPERATOR | 0x03 | Cancel break | — | unused | Operator-requested break |
| OTHER | 0x04 | Cancel break | — | unused | Unspecified break |

The break reason is published in the OpenPiste Level 2 `control` topic for consumption by competition management software and score sheet applications. The apparatus records the reason and updates its state accordingly.

Breaks are always initiated by the referee via the remote control. Competition management software does not push breaks to the apparatus.

---

### 7.6 COMPETITION — class 0x06

Controls competition flow in coordination with competition management software.

**All commands in this class except SWAP require an active competition management link.** If the link is unavailable, the apparatus MUST ignore those commands and SHOULD publish a `control` topic message indicating the failure. Local match operation (MATCH class) continues unaffected.

**SWAP** is an exception: it has a valid local use regardless of competition management link availability. Its local effect — swapping the apparatus's internal left/right fencer assignment — always applies. Its upstream effect — notifying competition management software — applies only when the link is active.

| cmd | cmd value | UNDO | CYCLE | Parameter | Description |
|-----|-----------|------|-------|-----------|-------------|
| BEGIN | 0x00 | — | — | unused | Activate current match on this piste |
| END | 0x01 | — | — | unused | Close match, submit result |
| NEXT | 0x02 | PREVIOUS | — | unused | Advance to next bout or team round |
| RESERVE | 0x03 | Cancel | — | unused | Signal reserve fencer substitution |
| SWAP | 0x04 | SWAP (self-inverse) | — | unused | Swap left/right fencer assignment |

`BEGIN` signals to competition management software that the current assigned match is now active on this piste. Requires competition management link.

`END` closes the current match and submits the result to competition management software. The apparatus enters the Ending state (see OpenPiste Level 2 Section 13) and awaits ACK or NAK from the software. Requires competition management link.

`NEXT` and its inverse `PREVIOUS` navigate the bout sequence within the current pool or DE table, and advance or step back rounds within a team match. The next match context — fencer identities, weapon, scheduled time, or team round assignment — is pushed to the apparatus by competition management software via OpenPiste Level 2. Requires competition management link.

`RESERVE` signals that a reserve fencer substitution is taking place in a team match. Fencer identity is known to the competition management software — the remote carries no fencer data. The software updates the score sheet accordingly. Requires competition management link.

`SWAP` swaps the apparatus's left/right fencer assignment. This may be used any time the referee decides that the fencers should exchange sides, with or without a competition management link. When the link is active, the swap is also published via OpenPiste Level 2 so that competition management software updates its records.

---

### 7.7 SYSTEM — class 0x07

Protocol-level commands for remote identity, pairing lifecycle, and capability discovery.

| cmd | cmd value | UNDO | CYCLE | Parameter | Description |
|-----|-----------|------|-------|-----------|-------------|
| PING | 0x00 | — | — | remote ID (16 bits) | Announce remote presence |
| CAPABILITY_QUERY | 0x01 | — | — | unused | Request apparatus capability publication |
| UNPAIR | 0x02 | — | — | unused | Release current pairing |

`PING` announces the remote's presence to the apparatus, triggering a state snapshot in response, and carries the remote's identity for pairing. See Section 9 for the full pairing flow.

`CAPABILITY_QUERY` requests the apparatus to publish its capability document to the OpenPiste Level 2 topic `openpiste/{piste_id}/capabilities`. Smart remotes use this to adapt their UI to the apparatus's supported feature set — see Section 10.

`UNPAIR` releases the current pairing. The apparatus returns to unpaired state and accepts the next PING as a new pairing. This is a sensitive operation. Implementations SHOULD require physical confirmation on the apparatus before acting on UNPAIR, to prevent accidental or deliberate unpairing during a match. See Section 9.6.

---

### 7.8 SETTINGS — class 0x08

Configures match parameters before or during a match.

UNDO is not applicable to any SETTINGS command. If a wrong setting is entered, the correct action is to issue the correct SETTINGS command. CYCLE is applicable to all SETTINGS commands except PERIOD_DURATION.

| cmd | cmd value | UNDO | CYCLE | Parameter | Description |
|-----|-----------|------|-------|-----------|-------------|
| WEAPON | 0x00 | — | Yes | low byte: weapon code | Set weapon |
| MATCH_TYPE | 0x01 | — | Yes | low byte: match type code | Set match type |
| PERIOD_COUNT | 0x02 | — | Yes | low byte: count (1–9) | Set number of periods |
| PERIOD_DURATION | 0x03 | — | — | time encoding (Section 5.4) | Set period duration |

**Weapon codes:**

| Code | Weapon | CYCLE position |
|------|--------|----------------|
| 0x01 | Épée | 1 |
| 0x02 | Foil | 2 |
| 0x03 | Sabre | 3 |
| 0x04 | Automatic | 4 |

`Automatic` instructs the apparatus to detect the weapon from the electrical characteristics of the connected weapon and body wire. This is an optional apparatus capability — see Section 10.

> **Note:** If the apparatus supports automatic weapon detection and it is active, the apparatus may override `SETTINGS WEAPON` autonomously based on the detected weapon. The current weapon is always reflected in the published OpenPiste Level 2 state, regardless of how it was determined.

**Match type codes:**

| Code | Match type | CYCLE position |
|------|-----------|----------------|
| 0x01 | Pool | 1 |
| 0x02 | DE (direct elimination) | 2 |
| 0x03 | Teams | 3 |
| 0x04 | Training | 4 |

---

### 7.9 VENDOR — class 0x7F

Reserved for non-standard, implementation-specific commands.

Commands in the VENDOR class are not part of the OpenPiste Remote Control Protocol. Interoperability across implementations is not guaranteed for VENDOR commands. Apparatus that receive VENDOR commands they do not recognise MUST ignore them silently.

Implementers using VENDOR commands are encouraged to document them publicly and to submit them for consideration in future revisions of this specification if they prove to be of general utility.

---

## 8. Transport bindings

OPRCP commands are transport-agnostic. The binary and JSON encodings defined in Sections 5 and 6 apply across all transports. This section defines the transport-specific details for each defined binding.

### 8.1 UDP

**Encoding:** Binary (Section 5) — 4 bytes per command.

**Addressing:** Pairing is established by network topology. In the recommended AP-per-apparatus deployment, the remote joins the apparatus Wi-Fi access point directly. The apparatus IP address is fixed and known. No addressing is required in the command payload.

**Reliability:** UDP provides no delivery guarantee. Operators observe the apparatus state response and reissue commands if necessary. Implementations MAY add a lightweight application-layer sequence and retransmit mechanism, but this is not required by the protocol.

**State response:** On processing a command, the apparatus sends a Level 2 JSON state snapshot to the remote's source IP and port. The payload format matches the relevant Level 2 state topic — see Section 11.

**Port:** Default OPRCP UDP port is **4242**. Implementations SHOULD use this port unless a specific alternative is configured.

### 8.2 MQTT

**Encoding:** JSON (Section 6).

**Topic structure:**

```
openpiste/{piste_id}/remote/{remote_id}
```

A remote paired to a specific apparatus publishes commands to the topic for that apparatus's piste ID, under its own remote ID subtopic. The apparatus subscribes to `openpiste/{piste_id}/remote/+` to receive commands from any paired remote.

```
openpiste/3/remote/163        # commands from remote 163 to piste 3
openpiste/podium/remote/7     # commands from remote 7 to the podium piste
```

**QoS:** QoS 1 (at least once). Commands MUST NOT be retained.

**State response:** The apparatus publishes state updates to the standard Level 2 topics. The remote subscribes to `openpiste/{piste_id}/+` for state.

**Pairing:** On MQTT transport, pairing is established by topic and SYSTEM PING. The remote publishes a `SYSTEM PING` on its first connection. The apparatus records the remote ID from the PING payload and restricts command acceptance to that remote's subtopic.

### 8.3 Bluetooth LE

**Encoding:** Binary (Section 5) — 4 bytes per command, transmitted as a BLE characteristic write.

**Addressing:** Pairing is established by the BLE connection. A BLE connection is inherently point-to-point — no addressing is required in the command payload.

**State response:** The apparatus notifies the remote via a dedicated BLE characteristic using the Level 2 JSON state format.

**Profile:** A formal BLE GATT profile for OPRCP (service UUID, characteristic UUIDs) is a planned addition — see Section 13.

### 8.4 Physical buttons

Physical button remotes implement a subset of the command taxonomy relevant to their button layout. The button press maps directly to a binary command frame transmitted via the chosen wireless transport (typically UDP or a proprietary RF link).

Physical button remotes have no state display capability and no feedback path. Operators using physical button remotes rely on the apparatus display or an external scoreboard to confirm command effects.

### 8.5 Non-defined transports

**Infrared (IR):** IR transport is not defined in this specification. While technically feasible, IR presents significant reliability limitations in competition environments — line-of-sight requirement, ambient light interference, no feedback path — and is considered unsuitable for production use. Implementers requiring low-cost wireless transport are directed to RF alternatives.

**ESP-NOW:** ESP-NOW is a viable transport for ESP32-based implementations, offering connectionless sub-millisecond latency without an access point or broker. It is not defined as a formal OPRCP transport binding because it is specific to the ESP32 platform, which would introduce a hardware dependency into an open standard. The OpenPiste CYD remote control reference implementation uses ESP-NOW — see https://github.com/pietwauters/CYDRemoteControl.

---

## 9. Pairing specification

OPRCP assumes a paired state. This section defines what pairing means, how it is established per transport, and how it is managed during operation.

### 9.1 What pairing means

A paired remote has:

- An unambiguous delivery path to exactly one scoring apparatus
- A remote identity known to that apparatus
- Exclusive command authority — the apparatus accepts OPRCP commands only from its paired remote

An apparatus in unpaired state accepts the first valid `SYSTEM PING` it receives and enters paired state with that remote.

### 9.2 Remote identity

Every smart remote SHOULD have a persistent 16-bit identifier carried in `SYSTEM PING`. The identifier SHOULD be:

- Hardware-derived (for example, the lower 16 bits of the device MAC address), or
- Randomly generated at first boot and stored in non-volatile memory

The identifier `0x0000` is reserved for anonymous remotes — physical button devices with no identity capability. An apparatus that receives a PING from an anonymous remote enters paired state but cannot enforce remote-specific filtering.

### 9.3 Pairing flow

```
1. Apparatus powers on → unpaired state
2. First SYSTEM PING received → apparatus records remote ID, enters paired state
3. All subsequent commands filtered: accepted only from paired remote ID
4. SYSTEM UNPAIR received from paired remote → apparatus returns to unpaired state
5. Next SYSTEM PING accepted as new pairing
```

On UDP transport, filtering is by remote ID carried in the PING. On MQTT transport, filtering is by topic — the apparatus accepts commands only from the subtopic of the paired remote ID. On Bluetooth LE, the BLE connection itself enforces exclusivity.

### 9.4 Pairing per transport

| Transport | Pairing mechanism | Remote ID enforcement |
|-----------|------------------|----------------------|
| UDP / AP model | Joining apparatus Wi-Fi AP + SYSTEM PING | By remote ID in PING |
| MQTT | Topic subscription + SYSTEM PING | By topic path |
| Bluetooth LE | BLE connection | By connection |
| Physical buttons | Fixed wiring or static RF channel | Not applicable |

### 9.5 Handling unauthorised commands

An apparatus that receives a command from an unrecognised remote SHOULD:

- Ignore the command silently
- Publish a `control` topic event (OpenPiste Level 2) indicating that an unauthorised command was received, for logging purposes

The apparatus MUST NOT act on commands from unrecognised remotes.

### 9.6 UNPAIR sensitivity

`SYSTEM UNPAIR` immediately disables the active remote if accepted. Implementations SHOULD require physical confirmation on the apparatus — for example, a dedicated button press on the device itself — before acting on UNPAIR. This prevents accidental unpairing and protects against a remote being used to disrupt a match in progress. The physical confirmation requirement SHOULD be documented in the apparatus implementation notes.

---

## 10. Capability document

On receiving `SYSTEM CAPABILITY_QUERY`, the apparatus publishes a capability document to the OpenPiste Level 2 topic:

```
openpiste/{piste_id}/capabilities
```

The capability document is a retained JSON object describing the apparatus's supported features. Smart remotes use this to adapt their UI — hiding controls for unsupported features, enabling controls for optional capabilities.

### Example capability document

```json
{
  "protocol":  "OPRCP1",
  "device":    "OpenPiste-ESP32",
  "version":   "1.0.0",
  "capabilities": {
    "weapon_auto_detect": true,
    "video_review":        true,
    "medical_timer":       true,
    "periods_max":         3,
    "teams":               true,
    "priority_auto":       true
  }
}
```

### Defined capability fields

| Field | Type | Description |
|-------|------|-------------|
| `weapon_auto_detect` | boolean | Apparatus detects weapon type automatically from electrical characteristics of the connected weapon and body wire |
| `video_review` | boolean | Apparatus supports video review tracking |
| `medical_timer` | boolean | Apparatus supports medical timeout countdown timer |
| `periods_max` | integer | Maximum number of periods supported |
| `teams` | boolean | Apparatus supports team match mode |
| `priority_auto` | boolean | Apparatus can perform a random priority draw |

Additional capability fields may be added by implementations. Field names in the `capabilities` object MUST be unique. Unknown fields MUST be ignored by receivers.

---

## 11. State response coverage

This section maps every OPRCP command class to the OpenPiste Level 2 topics the apparatus is expected to publish following a command. It serves as a verification reference for apparatus implementers.

| Command | Expected Level 2 topic(s) | Notes |
|---------|--------------------------|-------|
| CLOCK START / STOP / TOGGLE | `clock` | `running` field changes |
| CLOCK RESET | `clock` | `time_ms` resets to period default |
| CLOCK SET | `clock` | `time_ms` changes to set value |
| SCORE INCREMENT / DECREMENT | `score` | Score value for named side changes |
| CARD any | `score` | Card fields for named side change |
| MATCH NEXT_PERIOD / PREV_PERIOD | `match`, `clock` | `round` advances or steps back; clock resets |
| MATCH RESET | `score`, `clock`, `match` | All reset to defaults |
| MATCH PRIORITY_LEFT / RIGHT / AUTO | `score` | `priority` field changes |
| BREAK any | `control`, `state` | Break reason in `control`; apparatus state changes |
| COMPETITION BEGIN | `state` | Apparatus transitions to active match state |
| COMPETITION END | `state` | Apparatus enters Ending state (`"E"`), awaits ACK/NAK |
| COMPETITION NEXT / PREVIOUS | `match`, `fencers` | Bout metadata and fencer identities update |
| COMPETITION RESERVE | `fencers` | Fencer identity for named side updates |
| COMPETITION SWAP | `fencers` | Left/right fencer assignment swaps |
| SYSTEM PING | UDP: state snapshot to remote; MQTT: no publish | Pairing recorded internally |
| SYSTEM CAPABILITY_QUERY | `capabilities` | Capability document published |
| SYSTEM UNPAIR | none | Pairing state cleared internally |
| SETTINGS WEAPON | `match` | `weapon` field changes |
| SETTINGS MATCH_TYPE | `match` | `type` field changes |
| SETTINGS PERIOD_COUNT | `match` | Period count metadata changes |
| SETTINGS PERIOD_DURATION | `clock` | Default duration changes |

> **Open item:** The `state` topic values for break states (VIDEO, MEDICAL, EQUIPMENT, OPERATOR, OTHER) are not yet formally defined in OpenPiste Level 2. The current Level 2 state model uses `"H"` (Halt) for any stopped state. Whether break-specific state values should be added to Level 2, or whether break reason is communicated exclusively via the `control` topic, is to be resolved in a future revision. See also OpenPiste Level 2 Section 25.

> **Open item:** The full ACK/NAK state machine triggered by `COMPETITION END` is not yet formally specified in OpenPiste Level 2. OPRCP defers to that specification for the Ending state behaviour. See OpenPiste Level 2 Section 25.

---

## 12. Versioning and extensibility

### 12.1 Protocol identifier

Every JSON command carries `"protocol": "OPRCP1"`. A receiver SHOULD check this field and MAY ignore messages with an unrecognised identifier.

The binary encoding carries no explicit version field. The version is implied by the transport binding and the pairing negotiation. A formal binary versioning mechanism is a planned addition — see Section 13.

### 12.2 Adding commands

New commands may be added within an existing class using unassigned `cmd` values (0–63 per class). This is not a breaking change. Apparatus that receive unknown command values MUST ignore them silently.

### 12.3 Adding command classes

New command classes may be added using unassigned class values (0x09–0x7E). This is not a breaking change.

### 12.4 Breaking changes

Changes to the binary frame structure, flag semantics, or parameter encoding conventions constitute breaking changes and require a new protocol identifier (`OPRCP2`).

### 12.5 VENDOR class

The VENDOR class (0x7F) provides an escape hatch for non-standard commands without consuming reserved class values. See Section 7.9.

---

## 13. Open items

**Bluetooth LE GATT profile.** A formal GATT profile (service UUID, characteristic UUIDs, write and notify format) for OPRCP over BLE has not yet been defined. This is required before BLE implementations can interoperate across vendors.

**Binary versioning.** The binary encoding carries no explicit version field. A mechanism for version negotiation in the binary protocol — potentially via SYSTEM class commands — is under consideration.

**COMPETITION class error signalling.** The exact mechanism by which the apparatus signals a COMPETITION class command failure (no management link) to the remote and to the Level 2 `control` topic is not yet formally specified.

**Break state values in Level 2.** Whether the OpenPiste Level 2 `state` topic should carry break-specific state values, or whether break reason is communicated exclusively via the `control` topic, is an open item shared with the Level 2 specification. See Section 11.

**ACK/NAK state machine.** The full state machine around `COMPETITION END` and the Ending state is an open item in the Level 2 specification. OPRCP behaviour in the Ending state defers to that resolution.

**Security.** OPRCP inherits the security considerations of its transport. On MQTT transport, the OpenPiste Level 2 security model applies — see OpenPiste Level 2 Section 24. On UDP transport, no authentication mechanism is currently defined. Remote identity via SYSTEM PING provides basic pairing control but not cryptographic authentication. A formal security specification for OPRCP will be added in a future revision, aligned with the Level 2 security specification.

---

*OpenPiste Remote Control Protocol is released under the MIT licence.*
*Reference implementations and further documentation: https://openpiste.org*
