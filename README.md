# Event abstraction and timed norms for USPTO patent prosecution

This repository accompanies the anonymized article. It provides the two inputs that turn raw USPTO transaction records into an event log and check that log against the norms governing examination:

- `mapping/`: the event abstraction, which maps 96 PatEx transaction codes to 37 core and 19 friction activities (Table S1 of the article), and the mail-room codes that supply the mailing dates of examiner actions.
- `norms/`: the eleven timed norms of the article (Table 1) in machine-readable form, with the activation, target and superseding events and the deadline of each norm.

The analysis code is not included and is available from the author upon reasonable request.

## Data source

USPTO Patent Examination Research Dataset (PatEx), 2025 release, USPTO Open Data Portal, product ECOPAIR (<https://data.uspto.gov/bulkdata/datasets/ecopair>), files posted 25 September 2026. Event codes refer to the `event_code` field of `transactions_25.csv` and are described in `event_codes_25.csv`. Filing and issue dates come from `application_data_25.csv`.

## Contents

```
mapping/event_abstraction.csv   PatEx transaction code -> activity
mapping/mail_codes.csv          mail-room codes that date examiner actions
norms/norms.yaml                timed norms (L1, R1-R3, A1-A4, B, O1, O2)
```

## Event abstraction

| Column | Meaning |
|---|---|
| `event_code` | PatEx transaction code |
| `activity` | Abstracted activity |
| `category` | `core` for steps of the prosecution process, `friction` for events around it (IDSs, extensions, petitions, transfers, defect notices) |
| `tie_order` | Order of activities recorded on the same day |
| `note` | Remarks on the code |

The event log is built as follows.

1. Each transaction whose code appears in `event_abstraction.csv` becomes an event with the mapped activity. All other codes are dropped.
2. Two events are added from the application record: `Application Filed` at the filing date and `Patent Granted` at the issue date.
3. Events of the same case with the same activity on the same date are merged into one.
4. Events are ordered by date and, within a date, by `tie_order`, with `Application Filed` first and `Patent Granted` last.
5. Codes ABN9, BRCE and FRCE are recorded alongside a request for continued examination (RCEX). They are not mapped, so the disposal recorded by ABN9 is not treated as an abandonment.

Deadlines run from the mailing date of an action. For the examiner actions listed in `mail_codes.csv`, the record date is therefore replaced by the date of the first matching mail-room code on or after it, within 60 days. Actions without a match keep their record date.

## Timed norms

| Id | Source | Party | Activation -> target (deadline) |
|---|---|---|---|
| L1 | 35 U.S.C. 133 | Applicant | Action requiring a reply -> reply (6 months, hard stop) |
| R1 | 37 CFR 1.134, 1.136(a) | Applicant | Rejection -> reply after 3 months requires a paid extension |
| R2 | 37 CFR 1.114 | Applicant | RCE preceded by a closing action |
| R3 | 37 CFR 1.311 | Applicant | Allowance -> issue fee (3 months, non-extendable) |
| A1 | 35 U.S.C. 154(b)(1)(A)(i) | Office | Filing -> first action (14 months) |
| A2 | 35 U.S.C. 154(b)(1)(A)(ii) | Office | Reply or RCE -> next action (4 months) |
| A3 | 35 U.S.C. 154(b)(1)(A)(iii) | Office | Board decision reversing the examiner -> next action (4 months) |
| A4 | 35 U.S.C. 154(b)(1)(A)(iv) | Office | Issue fee -> grant (4 months) |
| B | 35 U.S.C. 154(b)(1)(B) | Office | Filing -> grant (36 months, gross and net of RCE time) |
| O1 | MPEP 706.07(a) | Office | Reply to a non-final rejection -> next action is not another non-final rejection |
| O2 | MPEP 710.06 | Office | Office action not withdrawn or vacated |

Each norm is a timed response constraint in the style of multi-perspective Declare. For an activation, the first later event in the target or superseding set is located. The activation is resolved if that event is a target and violated if it occurs after the deadline. Superseded activations are excluded, and the violation rate is the share of violated activations among resolved ones. Applicant deadlines carry three days of grace for periods that end on weekends or holidays. R1, R2, B, O1 and O2 depart from this template, and `norms.yaml` states their rules. Activity sets referenced with `$` are defined at the top of the file.

Released under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).
