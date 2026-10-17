# 94 Seconds Earlier

| | |
|---|---|
| **Category** | Forensics |
| **Points** | 73 |
| **Author** | Anonymous |
| **Solves** | 40 |
| **Flag format** | `CYBLTZ2026{<incident_ref>}` |

## Challenge

> The lobby controller and security desk logged the same 94-second outage with disagreeing clocks. Reconcile the incident and submit its incident reference as `CYBLTZ2026{<incident_ref>}`.

**Files provided**

- `sensor_window.csv`: raw lobby controller events (`controller_ms`, `zone`, `event`)
- `lobby_events.sqlite`: the security desk database, with an `incidents` table
- `incident_report.jpg`: a photo of two sticky notes on a monitor

## TL;DR

The controller clock runs **17342 ms ahead** of the desk DB. Subtract that offset from the controller's outage timestamps and only one row in the desk DB matches on zone, start time and duration: **`lobby-east`, 94 s, incident ref `d47e29ac836f10b2`**.

## Step 1: Read the photo

`incident_report.jpg` shows two sticky notes:

1. *"Lobby controller clock is FAST. 17342 ms ahead of our desk DB. Facilities told. Still not fixed."*
2. *"Outage drill Wed: log it as sev 5"*

The first note gives the clock offset: `desk_ms = controller_ms - 17342`.

The second note is a distractor. Every candidate incident around the outage is already severity 5, so severity can't be used to tell them apart.

## Step 2: Inspect the data

`sensor_window.csv` has about 60 events across several zones (`atrium`, `lobby-east`, `lobby-west`, `mezzanine`, and so on). The event types are `door-open`, `door-closed`, `occupancy-ping`, `lux-loss` and `lux-restored`.

The `lux-loss` / `lux-restored` pairs mark an outage, so the duration of each is the gap between them.

The database has a single table:

```sql
CREATE TABLE incidents(
    id INTEGER PRIMARY KEY,
    zone TEXT,
    start_ms INTEGER,
    duration_ms INTEGER,
    severity INTEGER,
    incident_ref TEXT
);
```

It holds several hundred rows, including many near-duplicates clustered around the target time, which are the decoys.

## Step 3: Compute outage durations per zone

Pairing each zone's `lux-loss` with its next `lux-restored` gives:

| Zone | Duration |
|---|---|
| roof-plant | 60 s |
| carpark-b1 | 87 s |
| atrium | 93 s |
| mezzanine | 94.5 s |
| **lobby-east** | **94 s** |
| **lobby-west** | **94 s** |
| loading-dock | 120 s |

Only the two lobby zones last exactly 94 s.

## Step 4: Apply the clock offset and match the DB

Shift each outage start by `-17342` ms and look for a row in `incidents` with the same zone, start and duration:

```python
import csv
import sqlite3

OFFSET = 17342  # controller clock is this many ms ahead of the desk DB

rows = list(csv.DictReader(open("sensor_window.csv")))
db = sqlite3.connect("lobby_events.sqlite")

for zone in sorted({r["zone"] for r in rows}):
    losses = [int(r["controller_ms"]) for r in rows
              if r["zone"] == zone and r["event"] == "lux-loss"]
    restores = [int(r["controller_ms"]) for r in rows
                if r["zone"] == zone and r["event"] == "lux-restored"]
    for loss in losses:
        later = [t for t in restores if t > loss]
        if not later:
            continue
        duration = later[0] - loss
        start = loss - OFFSET
        match = db.execute(
            "SELECT id, severity, incident_ref FROM incidents "
            "WHERE zone=? AND start_ms=? AND duration_ms=?",
            (zone, start, duration),
        ).fetchall()
        print(zone, start, duration, match)
```

Output (trimmed):

```
lobby-east   1782902525000  94000  [(7319, 5, 'd47e29ac836f10b2')]
lobby-west   1782902526000  94000  []
atrium       1782902527000  93000  []
carpark-b1   1782902525500  87000  []
...
```

Only `lobby-east` returns a match, id **7319**.

## Decoys in the database

The DB is seeded with near-misses, which is why the offset and the exact zone and duration all matter:

| Row | Why it's wrong |
|---|---|
| `lobby-west`, start 1782902525000, 94 s | Right time and duration, wrong zone. The corrected `lobby-west` start is `...526000`. |
| `lobby-east`, start 1782902525000, 87 s | Right zone and time, wrong duration. |
| `lobby-east`, starts `...524342`, `...525342`, `...526000` | Off by milliseconds or a full second. |
| `lobby-east`, start 1782902542342 | The raw, uncorrected controller timestamp. |

## Flag

```
CYBLTZ2026{d47e29ac836f10b2}
```

## Takeaways

- When two systems disagree on time, find the offset first and normalise everything to one clock before matching.
- Match on every field you can (zone, start, duration). A single field leaves several plausible candidates.
- Not every clue is useful. The "sev 5 drill" note was there to pull attention toward severity.
