# SAP ALE Demo — Student EDI/IDoc Exchange (Ministry ⇄ Faculty)

A pedagogical but production-realistic SAP application that exchanges student data between two SAP systems through **ALE / IDoc**:

- a **Ministry** system that holds the national master list of students, and
- a **Faculty** system that manages the students assigned to it.

It uses a single custom **message type** `ZSTUD` carrying **two basic types** (the MATMAS-style model), with real transactions, objects and the full ALE chain — the same pitfalls you hit on a real project.

> System IDs are masked in this repository as `DEV` (Ministry) and `QAS` (Faculty). Replace them with your own SIDs.

---

## The two flows

| Flow | Direction | Basic type | Content |
| --- | --- | --- | --- |
| Assignment | Ministry → Faculty | `ZSTUD01` | The student and their assigned specialization |
| Events | Faculty → Ministry | `ZSTUD02` | Result, average, death, university transfer |

One message type `ZSTUD`, two basic types, two directions.

---

## Data model

**Application tables**
- `ZSTUD_STUDENT` — student base (master on the Ministry side, received on the Faculty side)
- `ZSTUD_EVENT` — event log (written by the Faculty, received by the Ministry)
- `ZSTUD_UNIV`, `ZSTUD_FAC`, `ZSTUD_SPEC` — reference data (universities / faculties / specializations)

**IDoc segments (WE31)** — all fields are CHAR
- `ZSTUD01`: `Z1STUHDR` (student header) + `Z1STUAFF` (assignment)
- `ZSTUD02`: `Z1EVTHDR` (event header) + `Z1EVTDAT` (event data)

---

## Platform

All objects were created on **SAP_BASIS** and **SAP_ABA 758 SP04** (ABAP Platform 2023).

---

## Repository contents

This repository holds the transport **cofiles** and **data files** exported from the source system:

```
K9005xx.DEV   (cofile — Workbench)
K9005yy.DEV   (cofile — Customizing)
R9005xx.DEV   (data   — Workbench)
R9005yy.DEV   (data   — Customizing)
```

The whole solution is packaged into **two transport requests**:

| Transport | Type | Content |
| --- | --- | --- |
| `DEVK900578` | Workbench | Tables, data elements, segments, basic types, function groups, function modules, reports, transactions |
| `DEVK900580` | Customizing | Message type, WE82 links, process code, FM/message link (WE57), inbound FM attributes (BD51), logical systems (BD54), distribution model (BD64) |

### Object inventory

**Workbench**
- Tables: `ZSTUD_STUDENT` (+ `SITUATION`, `MOY_ACTUELLE`), `ZSTUD_EVENT`, `ZSTUD_UNIV`, `ZSTUD_FAC`, `ZSTUD_SPEC`
- Data elements: `ZE_CODEUNIV`, `ZE_LIBUNIV`, `ZE_VILLE`, `ZE_CODEFAC`, `ZE_LIBFAC`, `ZE_CODESPEC`, `ZE_LIBSPEC`, `ZE_NIVEAU`, `ZE_ACTIF`
- Segments (WE31): `Z1STUHDR`, `Z1STUAFF`, `Z1EVTHDR`, `Z1EVTDAT`
- Basic types (WE30): `ZSTUD01`, `ZSTUD02`
- Function groups: `ZSTUD_IDOC`, `ZSTUD_REF`
- Function modules: `Z_IDOC_INPUT_ZSTUD`, `Z_STUD_VALIDATE_AFFECT`
- Reports: `Z_STUD_SEED`, `Z_STUD_SEED_REF`, `Z_STUD_SEND_ASSIGN`, `Z_STUD_SEND_EVENT`, `Z_STUD_STUD_ENTRY`, `Z_STUD_AFFECT`, `Z_STUD_EVENT_ENTRY`, `Z_STUD_MASS_IMPORT`, `Z_STUD_MASS_AFFECT`
- Transactions (SE93): `ZSTUD_STU_CRE`, `ZSTUD_AFFECT`, `ZSTUD_EVENT`, `ZSTUD_IMPORT`, `ZSTUD_MASS_AFF`

**Customizing / ALE**
- Message type `ZSTUD` (WE81), WE82 links (`ZSTUD + ZSTUD01`, `ZSTUD + ZSTUD02`)
- Process code `ZSTU` (WE42), FM/message link (WE57), inbound FM attributes (BD51)
- Logical systems (BD54): `DEVCLNT100`, `QASCLNT100`
- Distribution model (BD64): `ZSTUD_MDL`

---

## Installation

### 1. Import the transports (TMS / STMS)

1. Copy the cofiles (`K9005*.DEV`) into the transport directory `cofiles/` and the data files (`R9005*.DEV`) into `data/` of your transport domain.
2. Add both requests to the import queue (STMS → Import Overview → your system → Extras → Other Requests → Add).
3. **Import the Workbench request first** (`DEVK900578`), then the **Customizing request** (`DEVK900580`).
4. The objects must exist on **both** systems (Ministry and Faculty) for the exchange to work.

> If activation warns about objects not yet active, re-run the import or activate the function groups in SE80 once the tables are active.

### 2. Manual configuration (not transportable — create on each system)

These objects are **not** transported and must be created by hand on each system:

- **RFC destinations (SM59):** `QAS_UM` on DEV, `DEV_UM` on QAS (service user, type System)
- **tRFC ports (WE21):** one port on each side, pointing to the RFC destination, **IDoc 4.x format**
- **Partner profiles (WE20):**
  - Outbound: `ZSTUD / ZSTUD01` on DEV, `ZSTUD / ZSTUD02` on QAS
  - Inbound: `ZSTUD` with process code `ZSTU`, mode *Trigger Immediately*
- **Authorization role** and **technical RFC user** for the service connection (principle of least privilege: `S_RFC`, `B_ALE_RECV` with `EDI_MES = ZSTUD`)
- **Distribution model (BD64):** `ZSTUD_MDL` — define it manually on each system rather than distributing it automatically

---

## Usage

**Ministry side**
- Create a student — `ZSTUD_STU_CRE`
- Assign a student to a faculty/specialization (cascading F4 help + validation) — `ZSTUD_AFFECT`
- Mass import from CSV (with downloadable template) — `ZSTUD_IMPORT`
- Mass assignment from CSV (line-by-line validation + log) — `ZSTUD_MASS_AFF`

**Faculty side**
- Enter an event (result, average, transfer, death; dropdown lists) — `ZSTUD_EVENT`

Creating a student triggers no IDoc; the **assignment** is what sends the student to the faculty. Events entered on the Faculty side travel back and update the student's current situation (`SITUATION`, `MOY_ACTUELLE`) on the Ministry side.

---

## Monitoring & troubleshooting

- Send queue: `RSEOUT00` (flush), `BD87` (reprocess), `WE02` / `WE05` (IDoc list), `WE19` (test tool), `ST22` (dumps)

Common pitfalls:

| Symptom | Cause | Fix |
| --- | --- | --- |
| `No entry in conversion table for external segment name Z2...` (EA274) | Empty release → SAP falls back to IDoc 3.0/3.1 (no long names) | Release the **segments AND the basic type** (WE30 → Set release), regenerate a fresh IDoc |
| `Interface for function ... is incorrect` (WE19) | Mis-named TABLES parameter | Use `SERIALIZATION_INFO` (not `SERIAL_NUMBERS`); align with `IDOC_INPUT_ORDERS` |
| IDoc stuck in status 30 | Partner profile in *Collect* mode | Switch to *Transfer Immediately*, or run `RSEOUT00` |
| Inbound error 51: No FM for input process code | Process code (WE42) has no FM identification | Fill the FM in the process code identification on each receiving system |

**Cross-cutting rule:** any segment that travels through a port must be **released on both sides**.

---

## Notes

This is a demo/training solution. It runs on a plain ABAP Platform server (no S/4HANA required) and is meant as a reusable base for other ALE/IDoc exchange scenarios.
