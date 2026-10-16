# OpenPiste Protocol — Level 3
## Tournament and Competition Data Layer

**Status:** Draft — working towards v1.0
**Date:** October 2026
**Author:** Piet Wauters
**Protocol identifier:** `OPP2`
**Repository:** https://github.com/OpenPiste
**Website:** https://openpiste.org

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [Relationship to Level 2](#2-relationship-to-level-2)
3. [Relationship to the FIE XML standard](#3-relationship-to-the-fie-xml-standard)
4. [Network and protocol stack](#4-network-and-protocol-stack)
5. [Topic structure](#5-topic-structure)
6. [Message overview](#6-message-overview)
7. [Common fields](#7-common-fields)
8. [Message: tournament/meta](#8-message-tournamentmeta)
9. [Message: tournament/pistes](#9-message-tournamentpistes)
10. [Message: tournament/officials](#10-message-tournamentofficials)
11. [Message: competition/meta](#11-message-competitionmeta)
12. [Message: competition/athletes](#12-message-competitionathletes)
13. [Message: phase/meta](#13-message-phasemeta)
14. [Message: phase/pool/composition](#14-message-phasepoolcomposition)
15. [Message: phase/pool/results](#15-message-phasepoolresults)
16. [Message: phase/ranking](#16-message-phaseranking)
17. [Message: phase/tableau](#17-message-phasetableau)
18. [Message: match](#18-message-match)
19. [The piste join — connecting Level 3 to Level 2](#19-the-piste-join--connecting-level-3-to-level-2)
20. [The federation bridge model](#20-the-federation-bridge-model)
21. [Status and enumerated values](#21-status-and-enumerated-values)
22. [Versioning and compatibility](#22-versioning-and-compatibility)
23. [Open items](#23-open-items)

---

## 1. Introduction

Level 3 defines the tournament and competition data layer of the OpenPiste Protocol. Where Level 2 addresses the live data produced on a single piste — lights, clock, score, and blade contact — Level 3 addresses the competition structure in which that piste participates: the tournament, the competitions it contains, the athletes registered, the phases they fence through, and the results that accumulate.

Level 3 is motivated by a specific and urgent problem in the fencing ecosystem. Live results from FIE events are not currently published in a machine-readable format by the event management services. The FIE maintains a formal XML data standard for competition results — FIE XML, currently at version 3.3 — but the competition management services that run events (FencingTime, Engarde, and others) do not expose live data feeds in this format. Spectators, media, and the FIE itself cannot access live results through a standardised channel. The FIE's own results pages are consequently poorly populated relative to what the event management software holds.

Level 3 addresses this gap by defining a publish/subscribe channel on the same MQTT broker infrastructure as Level 2. Competition management software that implements Level 3 publishes competition state as it changes — athlete registration, pool compositions, match progress, results, rankings — to well-known topics on the broker. Any subscriber — a spectator display, a federation statistics tool, a media feed, a live results website — can connect and consume that data without special access to CMS internals, without polling an API, and without depending on any single service provider.

A competition management service that publishes Level 3 data produces a vendor-neutral data stream that any compliant tool can consume. This is the same principle that motivates Level 2 for piste electronics: protocol compliance, not vendor agreement, is the integration point.

---

## 2. Relationship to Level 2

Level 2 and Level 3 operate on the same MQTT broker. They use different topic namespaces and address different concerns, but they share infrastructure, common fields, and a deliberate join point.

**Level 2** topics are piste-centric:
```
openpiste/{piste_id}/{publisher}/{message_type}
```
They carry the live operational state of one piste: lights, clock, score, blade contact. The piste is the authority for these topics. A competition management system contributes to the piste layer only for matters under its authority: fencer identity (`fencers` message), match metadata (`match` message), and score corrections.

**Level 3** topics are tournament-centric:
```
openpiste/{tournament_id}/...
```
They carry the competition structure, athlete data, phase progression, and results. Competition management software is the publisher for all Level 3 topics.

**The join point** between the two layers is the Level 3 match message (Section 18). A match record assigns a physical piste to a competition match. The `piste` field in the Level 3 match message carries the `{piste_id}` value that appears in Level 2 topic paths. A subscriber needing both competition context and live scoring data resolves the piste identifier from Level 3 and constructs the Level 2 topic path from it.

For example: a display showing live score alongside competition context subscribes to `openpiste/{tournament_id}/match/+` to learn that match `de-007` is assigned to piste `podium`. It then subscribes to `openpiste/podium/apparatus/score` for the live scoring data. The two subscriptions together provide the full picture.

Piste identifiers MUST be consistent between Level 2 and Level 3. When the same broker carries both layers, the `{piste_id}` values used in Level 2 topics and the `piste` field in Level 3 match messages MUST be identical strings.

---

## 3. Relationship to the FIE XML standard

The FIE publishes a formal XML data standard for competition data, currently at version 3.3 (2019). This standard is agreed across multiple competition management system vendors and is the FIE's authoritative format for competition results. Level 3 is designed to be semantically aligned with this standard.

The alignment is deliberate and practical. An organisation that wishes to import Level 3 data into a system that expects FIE XML, or to export Level 3 data as FIE XML, should be able to perform a mechanical field-by-field translation without semantic ambiguity. Where the FIE XML uses a term, Level 3 uses the same term in English. Where the FIE XML uses a code (a single letter or a defined abbreviation), Level 3 uses the same code.

The correspondence between FIE XML elements and Level 3 constructs is as follows:

| FIE XML element | Level 3 construct | Notes |
|-----------------|-------------------|-------|
| Root document | Tournament + Competitions | FIE XML is one file per competition; Level 3 groups them under a tournament |
| `CompetitionIndividuelle` | competition (`type: "I"`) | Direct map |
| `CompetitionParEquipes` | competition (`type: "T"`) | Direct map |
| `TourDePoules` | phase (`phase_type: "pool"`) | Pool round |
| `PhaseDeTableaux` | phase (`phase_type: "DE"`) | Direct elimination |
| `Poule` | pool | Direct map |
| `Tableau` | tableau within phase | DE bracket |
| `Match` | match | Both pool matches and DE matches |
| `Tireur` | athlete | Level 3 uses "athlete" to distinguish from the per-piste fencer assignment in Level 2 |
| `Equipe` | team | Team competitions |
| `Arbitre` | referee | Under tournament officials |
| `Officiel` | official | Under tournament officials |
| `Deroulement` (status codes) | `status` field values | P, R, F, L, O — see Section 21 |
| `Statut` (result codes) | `status` field in results | V, D, N, Q, X, DNF, DNS, MED, EXC — see Section 21 |
| `Arme` (weapon) | `weapon` | F, E, S |
| `Domaine` (domain) | `domain` | I, N, R, L, C, Z |
| `Sexe` (sex) | `sex` | M, F, X |

The FIE XML uses French field names throughout (`Tireur`, `Poule`, `Deroulement`, etc.). Level 3 uses English equivalents consistently. The mapping in the table above makes each correspondence explicit. Software translating between FIE XML and Level 3 MUST use this mapping and MUST NOT introduce new synonyms.

One deliberate difference: the FIE XML is organised per-competition (each export is one competition). Level 3 groups competitions under a tournament, reflecting the physical reality that a tournament runs multiple competitions on shared infrastructure with shared officials and shared pistes.

---

## 4. Network and protocol stack

Level 3 operates on the same MQTT broker as Level 2. All network and protocol stack considerations in Level 2 Section 4 apply to Level 3 as well.

### 4.1 Publisher

All Level 3 topics are published by competition management software. There is no `{publisher}` segment in Level 3 topic paths because the publisher role is unambiguous at this layer — competition structure data originates from the competition management system.

### 4.2 QoS and retained messages

All Level 3 topics use QoS 1 and retained messages. This ensures:
- Late-joining subscribers (spectator displays, results scrapers) immediately receive the current state of all competition topics on connect.
- Messages survive transient subscriber disconnections without data loss.
- The broker holds a complete snapshot of competition state at all times.

The `seq` field is present in all Level 3 messages and follows the same convention as Level 2 (Section 21 of Level 2).

### 4.3 Browser access

Web browsers can connect directly to the MQTT broker using MQTT over WebSockets (port 9001 by default, or port 443 with TLS). This means a live results website can be built as a static HTML/JavaScript page that subscribes directly to Level 3 topics, receiving updates in real time without any server-side component beyond the broker itself. No REST API, no polling, no intermediate data service is required.

This is a key architectural advantage for the FIE use case: the FIE can host a static results page that connects to the event's broker directly, displaying live results as they are published, without requiring any cooperation from the event management service beyond Level 3 compliance.

---

## 5. Topic structure

All Level 3 topics follow this pattern:

```
openpiste/{tournament_id}/...
```

The complete topic tree:

```
openpiste/{tournament_id}/meta
openpiste/{tournament_id}/pistes
openpiste/{tournament_id}/officials

openpiste/{tournament_id}/competition/{competition_id}/meta
openpiste/{tournament_id}/competition/{competition_id}/athletes

openpiste/{tournament_id}/competition/{competition_id}/phase/{phase_id}/meta
openpiste/{tournament_id}/competition/{competition_id}/phase/{phase_id}/pool/{pool_id}/composition
openpiste/{tournament_id}/competition/{competition_id}/phase/{phase_id}/pool/{pool_id}/results
openpiste/{tournament_id}/competition/{competition_id}/phase/{phase_id}/ranking
openpiste/{tournament_id}/competition/{competition_id}/phase/{phase_id}/tableau

openpiste/{tournament_id}/match/{match_id}
```

| Segment | Description | Examples |
|---------|-------------|---------|
| `openpiste` | Fixed platform prefix | — |
| `{tournament_id}` | Tournament identifier, unique per deployment | `wc-paris-2026`, `bel-senior-champs-2026` |
| `{competition_id}` | Competition identifier, unique within the tournament | `mens-epee-individual`, `womens-sabre-team` |
| `{phase_id}` | Phase identifier, unique within the competition | `1`, `2`, `pools-a`, `de-64` |
| `{pool_id}` | Pool identifier, unique within the phase | `A1`, `B3`, `3` |
| `{match_id}` | Match identifier, unique within the tournament | `pool-a1-m1`, `de-007`, `003` |

**Identifier requirements:**
- All identifiers are strings.
- Identifiers MUST be unique at their declared scope (competition IDs unique within tournament, match IDs unique within tournament, pool IDs unique within phase).
- Identifiers SHOULD be URL-safe strings (lowercase letters, digits, hyphens). Spaces and special characters MUST be avoided.
- Identifiers SHOULD be stable — the same physical entity SHOULD carry the same identifier throughout the event and, ideally, across events. This enables historical linkage and audit.

The `{match_id}` is tournament-scoped (not competition-scoped) because the match assignment topic — which joins competition data to the physical piste — is at tournament level. All tools that track piste assignments need only one subscription pattern: `openpiste/{tournament_id}/match/+`.

**Useful subscription patterns:**

```
openpiste/{tournament_id}/#                                             # all Level 3 data for this tournament
openpiste/{tournament_id}/competition/#                                 # all competition data
openpiste/{tournament_id}/competition/{competition_id}/#                # all data for one competition
openpiste/{tournament_id}/competition/{competition_id}/phase/+/ranking  # all phase rankings for one competition
openpiste/{tournament_id}/match/+                                       # all match assignments (piste join layer)
```

---

## 6. Message overview

| Topic suffix | QoS | Retained | Published when |
|-------------|-----|----------|---------------|
| `{tournament_id}/meta` | 1 | Yes | On tournament creation; on any metadata change |
| `{tournament_id}/pistes` | 1 | Yes | On venue setup; on piste availability change |
| `{tournament_id}/officials` | 1 | Yes | On official accreditation; on any roster change |
| `{tournament_id}/competition/{id}/meta` | 1 | Yes | On competition creation; on status change |
| `{tournament_id}/competition/{id}/athletes` | 1 | Yes | On athlete registration; on any roster change |
| `{tournament_id}/competition/{id}/phase/{id}/meta` | 1 | Yes | On phase creation; on status change |
| `{tournament_id}/competition/{id}/phase/{id}/pool/{id}/composition` | 1 | Yes | On pool draw; once, at pool creation |
| `{tournament_id}/competition/{id}/phase/{id}/pool/{id}/results` | 1 | Yes | On each match result within the pool; on pool completion |
| `{tournament_id}/competition/{id}/phase/{id}/ranking` | 1 | Yes | On phase completion; on each ranking update |
| `{tournament_id}/competition/{id}/phase/{id}/tableau` | 1 | Yes | On DE draw; on each match result update within the tableau |
| `{tournament_id}/match/{id}` | 1 | Yes | On match creation and piste assignment; on status and result change |

---

## 7. Common fields

Every Level 3 message contains the following common fields, following the same convention as Level 2 (Section 7 of Level 2).

| Field | Type | M/O | Default | Description |
|-------|------|-----|---------|-------------|
| `protocol` | string | M | — | Always `"OPP2"` |
| `version` | string | M | — | Protocol version — e.g. `"1.0"`. See Section 22. |
| `seq` | integer | M | — | Global sequence counter — see Level 2 Section 21 |

The `ts` field (timestamp) is optional at Level 3. Publication timestamps for competition data are less time-critical than live piste events and are not used for video synchronisation. Implementations MAY include `ts` for audit purposes.

---

## 8. Message: tournament/meta

**Topic:** `openpiste/{tournament_id}/meta`
**QoS:** 1
**Retained:** Yes

Published when the tournament is created and on any change to tournament metadata. This is the entry point for any subscriber discovering a tournament — the meta topic identifies the event, its dates, and its venue.

### Payload

```json
{
  "protocol":     "OPP2",
  "version":      "1.0",
  "seq":          1,
  "tournament_id": "wc-paris-2026",
  "name":         "Paris World Cup 2026",
  "venue":        "Stade Roland Garros",
  "city":         "Paris",
  "nation":       "FRA",
  "start_date":   "2026-04-12",
  "end_date":     "2026-04-14",
  "year":         2026,
  "domain":       "I",
  "live_url":     "https://live.openpiste.org/wc-paris-2026",
  "video_url":    "https://fie.org/live/wc-paris-2026"
}
```

### Fields

| Field | Type | M/O | Default | Description |
|-------|------|-----|---------|-------------|
| `protocol` | string | M | — | Always `"OPP2"` |
| `version` | string | M | — | Protocol version, e.g. `"1.0"` |
| `seq` | integer | M | — | Global sequence counter |
| `tournament_id` | string | M | — | Tournament identifier — matches the `{tournament_id}` topic segment |
| `name` | string | M | — | Full tournament name |
| `venue` | string | O | — | Venue name |
| `city` | string | O | — | City |
| `nation` | string | O | — | Host nation — IOC 3-letter code |
| `start_date` | string | O | — | Start date as `"YYYY-MM-DD"` |
| `end_date` | string | O | — | End date as `"YYYY-MM-DD"` |
| `year` | integer | M | — | Year of the tournament |
| `domain` | string | O | — | Event domain — see Section 21 |
| `live_url` | string | O | — | URL of the live results page |
| `video_url` | string | O | — | URL of the live video stream |

---

## 9. Message: tournament/pistes

**Topic:** `openpiste/{tournament_id}/pistes`
**QoS:** 1
**Retained:** Yes

Published when the venue is configured and on any change to piste availability. Lists the piste identifiers in use at this tournament. These identifiers correspond directly to the `{piste_id}` segment in Level 2 topics and to the `piste` field in Level 3 match messages.

### Payload

```json
{
  "protocol": "OPP2",
  "version":  "1.0",
  "seq":      2,
  "pistes":   ["1", "2", "3", "4", "5", "6", "7", "8", "podium"]
}
```

### Fields

| Field | Type | M/O | Default | Description |
|-------|------|-----|---------|-------------|
| `protocol` | string | M | — | Always `"OPP2"` |
| `version` | string | M | — | Protocol version, e.g. `"1.0"` |
| `seq` | integer | M | — | Global sequence counter |
| `pistes` | array of string | M | — | List of piste identifiers in use at this tournament. Each value MUST match the `{piste_id}` used in Level 2 topic paths. |

---

## 10. Message: tournament/officials

**Topic:** `openpiste/{tournament_id}/officials`
**QoS:** 1
**Retained:** Yes

Published when the official and referee roster is established and on any change. Carries all referees and officials accredited for the tournament. This is the authoritative roster at tournament scope; individual competition referees (piste assignments) are resolved through match records (Section 18).

### Payload

```json
{
  "protocol":  "OPP2",
  "version":   "1.0",
  "seq":       3,
  "officials": [
    {
      "id":     "o1",
      "name":   "J. Smith",
      "nation": "GBR",
      "role":   "DT"
    },
    {
      "id":     "o2",
      "name":   "M. Dubois",
      "nation": "FRA",
      "role":   "DA"
    }
  ],
  "referees": [
    {
      "id":     "r1",
      "name":   "A. Fischer",
      "nation": "GER",
      "role":   "P"
    },
    {
      "id":     "r2",
      "name":   "L. Verdi",
      "nation": "ITA",
      "role":   "V"
    }
  ]
}
```

### Fields

| Field | Type | M/O | Default | Description |
|-------|------|-----|---------|-------------|
| `protocol` | string | M | — | Always `"OPP2"` |
| `version` | string | M | — | Protocol version, e.g. `"1.0"` |
| `seq` | integer | M | — | Global sequence counter |
| `officials` | array | O | `[]` | Tournament officials roster |
| `officials[].id` | string | M | — | Official identifier |
| `officials[].name` | string | M | — | Official full name |
| `officials[].nation` | string | M | — | IOC 3-letter nation code |
| `officials[].role` | string | M | — | Official role — see Section 21 |
| `referees` | array | O | `[]` | Referee roster |
| `referees[].id` | string | M | — | Referee identifier |
| `referees[].name` | string | M | — | Referee full name |
| `referees[].nation` | string | M | — | IOC 3-letter nation code |
| `referees[].role` | string | M | — | Referee role — see Section 21 |

---

## 11. Message: competition/meta

**Topic:** `openpiste/{tournament_id}/competition/{competition_id}/meta`
**QoS:** 1
**Retained:** Yes

Published when a competition is created and on any change to competition metadata, including status changes. A tournament contains one or more competitions. Each competition is a distinct event (individual or team, per weapon, per gender category).

### Payload

```json
{
  "protocol":       "OPP2",
  "version":        "1.0",
  "seq":            4,
  "competition_id": "mens-epee-individual",
  "tournament_id":  "wc-paris-2026",
  "weapon":         "E",
  "type":           "I",
  "category":       "Senior",
  "sex":            "M",
  "domain":         "I",
  "title":          "Men's Épée Individual",
  "status":         "R"
}
```

### Fields

| Field | Type | M/O | Default | Description |
|-------|------|-----|---------|-------------|
| `protocol` | string | M | — | Always `"OPP2"` |
| `version` | string | M | — | Protocol version, e.g. `"1.0"` |
| `seq` | integer | M | — | Global sequence counter |
| `competition_id` | string | M | — | Competition identifier — matches the `{competition_id}` topic segment |
| `tournament_id` | string | M | — | Parent tournament identifier |
| `weapon` | string | M | — | `"F"` foil, `"E"` épée, `"S"` sabre |
| `type` | string | M | — | `"I"` individual, `"T"` team |
| `category` | string | O | — | Age or eligibility category (e.g. `"Senior"`, `"U20"`, `"U17"`, `"Veteran"`) |
| `sex` | string | O | — | `"M"` men, `"F"` women, `"X"` mixed |
| `domain` | string | O | — | Competition domain — see Section 21 |
| `title` | string | M | — | Human-readable competition title |
| `status` | string | M | `"P"` | Competition status — see Section 21 (Deroulement values) |

---

## 12. Message: competition/athletes

**Topic:** `openpiste/{tournament_id}/competition/{competition_id}/athletes`
**QoS:** 1
**Retained:** Yes

Published when athlete registration is established and on any change to the roster. Carries the full list of athletes registered for this competition. For team competitions, carries the list of teams and their members.

The term "athlete" is used at Level 3 to distinguish from the per-piste "fencer" concept in Level 2. The Level 2 `fencers` message identifies who is physically on a given piste; the Level 3 `athletes` message identifies who is registered in the competition. The two are linked by the `id` field, which MUST be the same identifier in both contexts.

### Payload — individual competition

```json
{
  "protocol":  "OPP2",
  "version":   "1.0",
  "seq":       5,
  "athletes": [
    {
      "id":           "32",
      "name":         "PANINI",
      "first_name":   "Bruno",
      "nation":       "ITA",
      "licence":      "ITA001234",
      "club":         "CS Pino Torinese",
      "laterality":   "R",
      "initial_rank": 3,
      "final_rank":   null,
      "status":       null,
      "photo_url":    "https://fie.org/athletes/32.jpg"
    }
  ]
}
```

### Payload — team competition

```json
{
  "protocol": "OPP2",
  "version":  "1.0",
  "seq":      6,
  "teams": [
    {
      "id":     "ita-team",
      "name":   "ITA",
      "nation": "ITA",
      "status": null,
      "members": [
        {
          "id":         "32",
          "name":       "PANINI",
          "first_name": "Bruno",
          "licence":    "ITA001234",
          "laterality": "R"
        }
      ]
    }
  ]
}
```

### Fields — individual competition

| Field | Type | M/O | Default | Description |
|-------|------|-----|---------|-------------|
| `protocol` | string | M | — | Always `"OPP2"` |
| `version` | string | M | — | Protocol version, e.g. `"1.0"` |
| `seq` | integer | M | — | Global sequence counter |
| `athletes` | array | M | — | Athlete roster |
| `athletes[].id` | string | M | — | Athlete identifier — MUST match the `id` used in Level 2 `fencers` messages |
| `athletes[].name` | string | M | — | Family name, uppercase |
| `athletes[].first_name` | string | O | — | Given name |
| `athletes[].nation` | string | M | — | IOC 3-letter nation code |
| `athletes[].licence` | string | O | — | Federation licence number |
| `athletes[].club` | string | O | — | Club name |
| `athletes[].laterality` | string | O | — | `"R"` right-handed, `"L"` left-handed |
| `athletes[].initial_rank` | integer | O | — | Seeded rank at competition start |
| `athletes[].final_rank` | integer | O | — | Final rank after competition completion; `null` while competition is in progress |
| `athletes[].status` | string | O | — | Athlete result status — see Section 21. `null` while competition is in progress. |
| `athletes[].photo_url` | string | O | — | URL of athlete photograph |

### Fields — team competition

| Field | Type | M/O | Default | Description |
|-------|------|-----|---------|-------------|
| `protocol` | string | M | — | Always `"OPP2"` |
| `version` | string | M | — | Protocol version, e.g. `"1.0"` |
| `seq` | integer | M | — | Global sequence counter |
| `teams` | array | M | — | Team roster |
| `teams[].id` | string | M | — | Team identifier |
| `teams[].name` | string | M | — | Team display name (typically nation code) |
| `teams[].nation` | string | M | — | IOC 3-letter nation code |
| `teams[].status` | string | O | — | Team result status — see Section 21 |
| `teams[].members` | array | M | — | Team member list |
| `teams[].members[].id` | string | M | — | Athlete identifier |
| `teams[].members[].name` | string | M | — | Family name, uppercase |
| `teams[].members[].first_name` | string | O | — | Given name |
| `teams[].members[].licence` | string | O | — | Federation licence number |
| `teams[].members[].laterality` | string | O | — | `"R"` right-handed, `"L"` left-handed |

---

## 13. Message: phase/meta

**Topic:** `openpiste/{tournament_id}/competition/{competition_id}/phase/{phase_id}/meta`
**QoS:** 1
**Retained:** Yes

Published when a phase is created and on any change to phase status. A competition is structured as a sequence of phases. Typical individual competitions have a pool round phase followed by one or more direct elimination phases. Team competitions have a similar structure with team matches.

### Payload

```json
{
  "protocol":       "OPP2",
  "version":        "1.0",
  "seq":            7,
  "phase_id":       "1",
  "competition_id": "mens-epee-individual",
  "phase_type":     "pool",
  "title":          "Pool Round",
  "status":         "R",
  "nb_athletes":    200,
  "nb_qualifiers":  64
}
```

### Fields

| Field | Type | M/O | Default | Description |
|-------|------|-----|---------|-------------|
| `protocol` | string | M | — | Always `"OPP2"` |
| `version` | string | M | — | Protocol version, e.g. `"1.0"` |
| `seq` | integer | M | — | Global sequence counter |
| `phase_id` | string | M | — | Phase identifier — matches the `{phase_id}` topic segment |
| `competition_id` | string | M | — | Parent competition identifier |
| `phase_type` | string | M | — | Phase type — see values below |
| `title` | string | O | — | Human-readable phase title |
| `status` | string | M | `"P"` | Phase status — see Section 21 |
| `nb_athletes` | integer | O | — | Number of athletes participating in this phase |
| `nb_qualifiers` | integer | O | — | Number of athletes qualifying from this phase to the next |

**Phase type values** (consistent with Level 2 `match` message `phase_type` field):

| Value | FIE XML element | Meaning |
|-------|-----------------|---------|
| `"pool"` | `TourDePoules` | Pool round |
| `"DE"` | `PhaseDeTableaux` | Direct elimination |
| `"repechage"` | — | Repechage round |
| `"classification"` | — | Classification round |

---

## 14. Message: phase/pool/composition

**Topic:** `openpiste/{tournament_id}/competition/{competition_id}/phase/{phase_id}/pool/{pool_id}/composition`
**QoS:** 1
**Retained:** Yes

Published once when the pool draw is completed. Describes which athletes are in the pool, which piste it is assigned to, and when it is scheduled. This topic is retained and is not updated after publication (pool compositions do not change after the draw).

### Payload

```json
{
  "protocol":       "OPP2",
  "version":        "1.0",
  "seq":            8,
  "pool_id":        "A1",
  "phase_id":       "1",
  "competition_id": "mens-epee-individual",
  "piste":          "5",
  "scheduled":      "09:00",
  "status":         "P",
  "athletes":       ["32", "28", "45", "67", "89", "12"]
}
```

### Fields

| Field | Type | M/O | Default | Description |
|-------|------|-----|---------|-------------|
| `protocol` | string | M | — | Always `"OPP2"` |
| `version` | string | M | — | Protocol version, e.g. `"1.0"` |
| `seq` | integer | M | — | Global sequence counter |
| `pool_id` | string | M | — | Pool identifier — matches the `{pool_id}` topic segment |
| `phase_id` | string | M | — | Parent phase identifier |
| `competition_id` | string | M | — | Parent competition identifier |
| `piste` | string | O | — | Piste assignment — the Level 2 `{piste_id}` for this pool |
| `scheduled` | string | O | — | Scheduled start time as `"HH:MM"` |
| `status` | string | M | `"P"` | Pool status — see Section 21 |
| `athletes` | array of string | M | — | Ordered list of athlete identifiers assigned to this pool. For team competitions, team identifiers. |

---

## 15. Message: phase/pool/results

**Topic:** `openpiste/{tournament_id}/competition/{competition_id}/phase/{phase_id}/pool/{pool_id}/results`
**QoS:** 1
**Retained:** Yes

Published on each match result within the pool as it becomes available, and republished on pool completion with final totals. This topic evolves during the pool round — a subscriber connecting mid-round receives the results accumulated so far.

### Payload

```json
{
  "protocol":       "OPP2",
  "version":        "1.0",
  "seq":            9,
  "pool_id":        "A1",
  "phase_id":       "1",
  "competition_id": "mens-epee-individual",
  "status":         "R",
  "results": [
    {
      "athlete_id":        "32",
      "rank":              1,
      "victories":         5,
      "touches_scored":    25,
      "touches_received":  15,
      "status":            "V"
    },
    {
      "athlete_id":        "28",
      "rank":              2,
      "victories":         4,
      "touches_scored":    21,
      "touches_received":  16,
      "status":            "V"
    }
  ]
}
```

### Fields

| Field | Type | M/O | Default | Description |
|-------|------|-----|---------|-------------|
| `protocol` | string | M | — | Always `"OPP2"` |
| `version` | string | M | — | Protocol version, e.g. `"1.0"` |
| `seq` | integer | M | — | Global sequence counter |
| `pool_id` | string | M | — | Pool identifier |
| `phase_id` | string | M | — | Parent phase identifier |
| `competition_id` | string | M | — | Parent competition identifier |
| `status` | string | M | — | Pool status — see Section 21 |
| `results` | array | M | — | Per-athlete pool results |
| `results[].athlete_id` | string | M | — | Athlete identifier |
| `results[].rank` | integer | O | — | Rank within the pool after completion; `null` while pool is in progress |
| `results[].victories` | integer | M | `0` | Number of victories so far |
| `results[].touches_scored` | integer | M | `0` | Total touches scored (TD) |
| `results[].touches_received` | integer | M | `0` | Total touches received (TR) |
| `results[].status` | string | O | — | Athlete result status for this pool — see Section 21 |

---

## 16. Message: phase/ranking

**Topic:** `openpiste/{tournament_id}/competition/{competition_id}/phase/{phase_id}/ranking`
**QoS:** 1
**Retained:** Yes

Published when the ranking for a phase is established — typically on phase completion, but may be published during the phase for intermediate rankings. For pool rounds, this is the ranking after all pools are complete. For direct elimination phases, this is updated after each match eliminates an athlete.

### Payload

```json
{
  "protocol":       "OPP2",
  "version":        "1.0",
  "seq":            10,
  "phase_id":       "1",
  "competition_id": "mens-epee-individual",
  "status":         "F",
  "ranking": [
    {
      "rank":       1,
      "athlete_id": "32",
      "status":     "V"
    },
    {
      "rank":       2,
      "athlete_id": "28",
      "status":     "V"
    },
    {
      "rank":       null,
      "athlete_id": "45",
      "status":     "DNS"
    }
  ]
}
```

### Fields

| Field | Type | M/O | Default | Description |
|-------|------|-----|---------|-------------|
| `protocol` | string | M | — | Always `"OPP2"` |
| `version` | string | M | — | Protocol version, e.g. `"1.0"` |
| `seq` | integer | M | — | Global sequence counter |
| `phase_id` | string | M | — | Phase identifier |
| `competition_id` | string | M | — | Parent competition identifier |
| `status` | string | M | — | Phase status — see Section 21 |
| `ranking` | array | M | — | Ordered ranking entries |
| `ranking[].rank` | integer | O | — | Numeric rank; `null` for athletes not ranked (DNS, EXC, etc.) |
| `ranking[].athlete_id` | string | M | — | Athlete or team identifier |
| `ranking[].status` | string | O | — | Athlete result status — see Section 21 |

---

## 17. Message: phase/tableau

**Topic:** `openpiste/{tournament_id}/competition/{competition_id}/phase/{phase_id}/tableau`
**QoS:** 1
**Retained:** Yes

Published when a direct elimination (DE) tableau is drawn and updated on each match result. Carries the complete bracket structure with current state. Updated as matches are resolved, allowing subscribers to track bracket progression in real time.

For pool phases, this topic is absent — pool match structure is carried through the pool topics (Sections 14–15) and the match topic (Section 18).

### Payload

```json
{
  "protocol":       "OPP2",
  "version":        "1.0",
  "seq":            11,
  "phase_id":       "2",
  "competition_id": "mens-epee-individual",
  "size":           64,
  "title":          "Tableau of 64",
  "status":         "R",
  "matches": [
    {
      "match_id":            "de-001",
      "position":            1,
      "athlete_left_id":     "32",
      "athlete_right_id":    "28",
      "score_left":          null,
      "score_right":         null,
      "status":              "P",
      "winner_id":           null,
      "loser_destination":   "classification-of-32"
    },
    {
      "match_id":            "de-002",
      "position":            2,
      "athlete_left_id":     "45",
      "athlete_right_id":    "67",
      "score_left":          15,
      "score_right":         12,
      "status":              "F",
      "winner_id":           "45",
      "loser_destination":   "classification-of-32"
    }
  ]
}
```

### Fields

| Field | Type | M/O | Default | Description |
|-------|------|-----|---------|-------------|
| `protocol` | string | M | — | Always `"OPP2"` |
| `version` | string | M | — | Protocol version, e.g. `"1.0"` |
| `seq` | integer | M | — | Global sequence counter |
| `phase_id` | string | M | — | Phase identifier |
| `competition_id` | string | M | — | Parent competition identifier |
| `size` | integer | M | — | Tableau size (number of initial positions) — typically 8, 16, 32, 64, 128, 256 |
| `title` | string | O | — | Human-readable tableau title |
| `status` | string | M | `"P"` | Tableau status — see Section 21 |
| `matches` | array | M | — | Match entries in this tableau |
| `matches[].match_id` | string | M | — | Match identifier — same as used in `openpiste/{tournament_id}/match/{match_id}` |
| `matches[].position` | integer | M | — | Bracket position (1-indexed from left) |
| `matches[].athlete_left_id` | string | O | — | Athlete or team on the left side; `null` for byes or unresolved bracket positions |
| `matches[].athlete_right_id` | string | O | — | Athlete or team on the right side |
| `matches[].score_left` | integer | O | — | Final score for left athlete; `null` until match is complete |
| `matches[].score_right` | integer | O | — | Final score for right athlete; `null` until match is complete |
| `matches[].status` | string | M | `"P"` | Match status — see Section 21 |
| `matches[].winner_id` | string | O | — | Identifier of the winning athlete or team; `null` until match is complete |
| `matches[].loser_destination` | string | O | — | Identifier of the phase the losing athlete is directed to, if any |

---

## 18. Message: match

**Topic:** `openpiste/{tournament_id}/match/{match_id}`
**QoS:** 1
**Retained:** Yes

Published when a match is created and assigned to a piste, and updated on any status or result change. This is the primary join topic between Level 3 (competition layer) and Level 2 (piste layer). See Section 19 for the complete join semantics.

Every match in the tournament — pool matches, DE matches, team relay matches — has a record here. The `match_id` is the same identifier used in the pool composition, pool results, and tableau messages.

### Payload

```json
{
  "protocol":       "OPP2",
  "version":        "1.0",
  "seq":            12,
  "match_id":       "de-007",
  "competition_id": "mens-epee-individual",
  "phase_id":       "2",
  "phase_type":     "DE",
  "pool_id":        null,
  "piste":          "podium",
  "referee_id":     "r1",
  "scheduled":      "14:30",
  "status":         "R",
  "athlete_left_id":  "32",
  "athlete_right_id": "28",
  "round":          1,
  "score_left":     null,
  "score_right":    null,
  "winner_id":      null
}
```

### Fields

| Field | Type | M/O | Default | Description |
|-------|------|-----|---------|-------------|
| `protocol` | string | M | — | Always `"OPP2"` |
| `version` | string | M | — | Protocol version, e.g. `"1.0"` |
| `seq` | integer | M | — | Global sequence counter |
| `match_id` | string | M | — | Match identifier — matches the `{match_id}` topic segment |
| `competition_id` | string | M | — | Competition this match belongs to |
| `phase_id` | string | M | — | Phase this match belongs to |
| `phase_type` | string | M | — | Phase type — consistent with Level 2 `match` message and Section 13 |
| `pool_id` | string | O | — | Pool identifier for pool matches; `null` for DE matches |
| `piste` | string | O | — | Piste assignment — the Level 2 `{piste_id}` for this match. `null` if not yet assigned. |
| `referee_id` | string | O | — | Referee identifier — must match an entry in the tournament officials roster |
| `scheduled` | string | O | — | Scheduled start time as `"HH:MM"` |
| `status` | string | M | `"P"` | Match status — see Section 21 |
| `athlete_left_id` | string | M | — | Athlete or team on the left side of the piste |
| `athlete_right_id` | string | M | — | Athlete or team on the right side of the piste |
| `round` | integer | O | `1` | Current relay round for team matches (1–9); always 1 for individual matches |
| `score_left` | integer | O | — | Final score for left athlete or team; `null` until match is complete |
| `score_right` | integer | O | — | Final score for right athlete or team; `null` until match is complete |
| `winner_id` | string | O | — | Identifier of the winning athlete or team; `null` until match is complete |

---

## 19. The piste join — connecting Level 3 to Level 2

The `piste` field in the Level 3 match message (Section 18) is the link between the competition layer and the piste layer. It carries the same string as the `{piste_id}` segment in Level 2 topic paths.

**Example scenario:** The quarter-final between PANINI and MARTIN is match `de-007`, assigned to the podium piste.

Level 3 match record:
```
Topic:   openpiste/wc-paris-2026/match/de-007
Payload: { ..., "piste": "podium", "athlete_left_id": "32", "athlete_right_id": "28", ... }
```

The corresponding live Level 2 score topic:
```
openpiste/podium/apparatus/score
```

The corresponding Level 2 fencer identity topic (published by the CMS when the match starts):
```
openpiste/podium/software/fencers
```

A subscriber needing both the competition context and the live scoring data follows this procedure:
1. Subscribe to `openpiste/{tournament_id}/match/+` — receives all match records for the tournament.
2. When a match record arrives with `"status": "R"` (running), extract the `piste` field.
3. Construct the Level 2 score subscription: `openpiste/{piste}/apparatus/score`.
4. The retained score message is immediately available.

A scoreboard needing to show fencer names alongside live score follows the same procedure, additionally subscribing to `openpiste/{piste}/software/fencers`. The `athlete_left_id` and `athlete_right_id` in the match record correspond to the `id` fields in the Level 3 athletes roster (Section 12) and to the `id` fields in the Level 2 fencers message (Section 14 of Level 2).

**Consistency requirement:** Competition management software that publishes both Level 2 and Level 3 data MUST ensure that:
- The `piste` value in Level 3 match records exactly matches the `{piste_id}` used in the Level 2 topic path for that piste.
- The `athlete_left_id` and `athlete_right_id` in Level 3 match records exactly match the `id` fields in the Level 2 `fencers` message for the same match.
- The `competition_id` in Level 3 match records exactly matches the `competition` field in the Level 2 `match` message for that piste.

These cross-layer identifiers are the load-bearing joins of the platform architecture. Inconsistency here breaks subscriber applications that rely on them.

---

## 20. The federation bridge model

Level 2 defines a piste-local topic namespace without tournament context. A local competition venue publishes under `openpiste/{piste_id}/...` on a local broker. Level 3 data is published under `openpiste/{tournament_id}/...` on the same broker.

For access from outside the venue — by the FIE, by media, by remote spectators — the broker may be exposed to the internet, or a bridge may forward data to a cloud broker. The bridge model requires no changes to the apparatus firmware or local Level 2 publishers.

**Local broker (venue):**
```
openpiste/{piste_id}/apparatus/score         ← apparatus publishes here
openpiste/{tournament_id}/match/{match_id}   ← CMS publishes here
openpiste/{tournament_id}/competition/...    ← CMS publishes here
```

**Cloud broker (federation):** The bridge subscribes to all local topics and re-publishes under the same topic paths on the cloud broker. No transformation is required — the topic structure is already globally unique by tournament ID. The bridge is responsible for authentication to the cloud broker; local devices need no configuration change.

For venues that run multiple concurrent tournaments (uncommon but possible), each tournament has a distinct `{tournament_id}` and therefore a distinct topic namespace. There is no ambiguity.

**Competition ID uniqueness:** Competition IDs must be unique within a tournament. They need not be globally unique — the `{tournament_id}` prefix in the topic provides global uniqueness. A competition ID of `"mens-epee-individual"` is unambiguous within `wc-paris-2026` even if the same string is used in `bel-senior-champs-2026`.

**Match ID uniqueness:** Match IDs must be unique within a tournament. This is a stronger requirement than competition-scoped uniqueness, but it enables the flat `openpiste/{tournament_id}/match/{match_id}` topic structure that makes the piste join subscription simple.

---

## 21. Status and enumerated values

### 21.1 Deroulement — lifecycle status

Used in competition, phase, pool, tableau, and match status fields.

| Value | FIE XML | Meaning |
|-------|---------|---------|
| `"P"` | P | Planned — not yet started |
| `"R"` | R | Running — in progress |
| `"F"` | F | Finished — complete |
| `"L"` | L | (Competition management software defined — typically "paused") |
| `"O"` | O | (Competition management software defined — typically "abandoned") |

### 21.2 Statut — athlete/team result status

Used in athlete roster, pool results, and ranking entries.

| Value | FIE XML | Meaning |
|-------|---------|---------|
| `"V"` | V | Victory |
| `"D"` | D | Defeat |
| `"N"` | N | Null / not determined |
| `"Q"` | Q | Qualified |
| `"X"` | X | Eliminated |
| `"DNF"` | DNF | Did not finish |
| `"DNS"` | DNS | Did not show |
| `"MED"` | MED | Medical withdrawal |
| `"EXC"` | EXC | Excluded |

### 21.3 Weapon

| Value | FIE XML (`Arme`) | Meaning |
|-------|-----------------|---------|
| `"F"` | F | Foil |
| `"E"` | E | Épée |
| `"S"` | S | Sabre |

These values are identical to the `weapon` field in Level 2 match messages (Section 15 of Level 2).

### 21.4 Domain

| Value | FIE XML (`Domaine`) | Meaning |
|-------|---------------------|---------|
| `"I"` | I | International |
| `"N"` | N | National |
| `"R"` | R | Regional |
| `"L"` | L | Local |
| `"C"` | C | Club |
| `"Z"` | Z | Other |

### 21.5 Sex

| Value | FIE XML (`Sexe`) | Meaning |
|-------|-----------------|---------|
| `"M"` | M | Men |
| `"F"` | F | Women |
| `"X"` | — | Mixed |

### 21.6 Referee role

| Value | FIE XML (`RoleArbitre`) | Meaning |
|-------|------------------------|---------|
| `"P"` | P | Piste referee |
| `"A"` | A | Assistant referee |
| `"V"` | V | Video official |

### 21.7 Official role

| Value | FIE XML (`RoleOfficiel`) | Meaning |
|-------|-------------------------|---------|
| `"DT"` | DT | Director of Tournament (Directeur de Tournoi) |
| `"DA"` | DA | Assistant Director |
| `"S"` | S | Secretary |

### 21.8 Laterality

| Value | FIE XML (`Lateralite`) | Meaning |
|-------|------------------------|---------|
| `"R"` | D | Right-handed |
| `"L"` | G | Left-handed |

Note: The FIE XML uses D (Droitier) and G (Gaucher); Level 3 uses R and L for consistency with English convention. Translation software MUST map between these.

---

## 22. Versioning and compatibility

Level 3 uses the same versioning model as Level 2. All Level 3 messages carry `"protocol": "OPP2"` and a `"version"` string. The `version` field governs which fields are mandatory for a sender claiming compliance with that version.

Rules for adding fields, breaking changes, and enumerated value extension are identical to those in Level 2 Section 23. In particular:

- New optional fields may be added in a minor revision without a version increment.
- New enumerated values (status codes, phase types, etc.) may be added without a version increment; receivers MUST ignore unknown values.
- Removing or renaming mandatory fields, or changing field types, requires a new protocol identifier.

Level 2 and Level 3 share the `"OPP2"` protocol identifier and a common version space. A change that increments the version in Level 2 increments it in Level 3 as well. A receiver checking the `version` field sees the same version string in both layers and can reason uniformly about compatibility.

---

## 23. Open items

**Team match relay structure.** Team matches consist of up to nine relay bouts between individual fencer pairs. The current Level 3 model treats a team match as a single match record with a `round` field. A more detailed model would publish a separate match record for each relay bout within the team match, enabling piste join for each individual relay. The right granularity will be confirmed based on use cases from team competition deployments.

**Live score integration.** The Level 3 match record carries `score_left` and `score_right` as final scores after match completion. Whether the Level 3 match topic should also carry live score updates (populated from the Level 2 apparatus score) is an open question. One option is for the bridge to update the Level 3 match record in real time. Another is to leave live scoring exclusively to Level 2 and use Level 3 only for final results. The current design favours the latter — it keeps the publisher separation clean — but this will be reviewed based on subscriber requirements.

**Historical data and pagination.** The retained message model works well for current competition state but is not designed for historical queries (e.g. "all matches completed today"). A REST API for historical access is under consideration as a complement to the MQTT live feed. This would be consistent with the principle that MQTT serves live streams and REST serves management/historical queries, as stated in the Level 2 rationale document.

**JSON Schema.** A machine-readable JSON Schema for all Level 3 message types is planned as a separate document at `schemas/opp3/` in the OpenPiste repository. Not yet published.

**FIE XML export.** A reference implementation translating between Level 3 topics and FIE XML export files (one file per competition, as the FIE standard specifies) is planned. The field mapping in Section 3 is the basis for this translation.

---

*OpenPiste Protocol Level 3 is released under the MIT licence.*
*Reference implementation and further documentation: https://openpiste.org*
