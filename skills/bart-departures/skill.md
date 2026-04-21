---
name: bart-departures
description: Look up real-time BART train departures and find all viable routes (including transfers) from any station to any destination by a deadline
version: 2.0.0
tags: [api, transit, real-time]
author: shubhampatilsd
created: 2026-04-21
---

When the user asks for the next BART train, upcoming BART departures, or real-time BART schedule from a station, use the public BART API to fetch live departure estimates.

## API Details

- **Base URL:** `https://api.bart.gov/api/etd.aspx` (real-time) and `https://api.bart.gov/api/sched.aspx` (schedules)
- **Public API key:** `MW9S-E7SL-26DU-VV8V` (no registration required)
- **Station codes:** Use the 4-letter BART station abbreviation (e.g. `FRMT` for Fremont, `EMBR` for Embarcadero, `12TH` for 12th St Oakland, `DALY` for Daly City)

---

## Task 1: Real-time departures from a station

```bash
curl -s "https://api.bart.gov/api/etd.aspx?cmd=etd&orig=FRMT&key=MW9S-E7SL-26DU-VV8V&json=y" | python3 -c "
import json, sys
data = json.load(sys.stdin)
station = data['root']['station'][0]
print(f\"Station: {station['name']}\")
print(f\"Time: {data['root']['time']}\n\")
for etd in station['etd']:
    dest = etd['destination']
    for est in etd['estimate']:
        mins = est['minutes']
        platform = est['platform']
        color = est['color']
        print(f\"{dest} ({color}) — {mins} min — Platform {platform}\")
"
```

Group by platform and present as a table with columns: Destination, Line (color), Minutes until departure.

---

## Task 2: Route planning with a deadline (the robust approach)

**The BART trip planner API only returns one suggested route and often misses faster transfer options.** When the user wants to reach a destination by a deadline, use the multi-route strategy below instead.

### Step 1 — Get the BART route map to find candidate transfer stations

```bash
curl -s "https://api.bart.gov/api/route.aspx?cmd=routes&key=MW9S-E7SL-26DU-VV8V&json=y" | python3 -c "
import json, sys
data = json.load(sys.stdin)
for r in data['root']['routes']['route']:
    print(r['number'], r['name'], r['abbr'])
"
```

Then fetch the stops for each relevant route:

```bash
curl -s "https://api.bart.gov/api/route.aspx?cmd=routeinfo&route=ROUTE_NUMBER&key=MW9S-E7SL-26DU-VV8V&json=y" | python3 -c "
import json, sys
data = json.load(sys.stdin)
route = data['root']['routes']['route']
print(route['name'])
stops = route['config']['station']
print(' -> '.join(stops))
"
```

Use this to identify every station where the user's origin line intersects with lines that serve the destination. These are your **candidate transfer stations**.

### Step 2 — Check the BART trip planner for each candidate transfer

For each candidate transfer station T, check:
1. Origin → T (what time does the user arrive at T?)
2. T → Destination (what is the next train from T after arrival?)

```bash
# Check direct route + all transfer options in parallel
for TRANSFER in BAYF 19TH MCAR OAKS WOAK; do
  curl -s "https://api.bart.gov/api/sched.aspx?cmd=depart&orig=FRMT&dest=$TRANSFER&date=now&key=MW9S-E7SL-26DU-VV8V&json=y&b=0&a=2" | python3 -c "
import json, sys, os
data = json.load(sys.stdin)
trips = data['root']['schedule']['request']['trip']
if isinstance(trips, dict): trips = [trips]
for t in trips[:1]:
    legs = t.get('leg', [])
    if isinstance(legs, dict): legs = [legs]
    print(f'Via $TRANSFER: depart {t[\"@origTimeMin\"]} arrive $TRANSFER {legs[-1][\"@destTimeMin\"]}')
  " &
done
wait
```

Then for each arrival time at each transfer station, check the next train to the destination.

### Step 3 — Compare all options

Collect: (departure time, transfer station or "direct", arrival time at destination) for all viable options. Eliminate any that arrive after the deadline. Present as a ranked table, earliest arrival first.

**Always include the direct route** (no transfer) as a baseline, even if slower.

### Common BART transfer stations for East Bay → SF trips

| Transfer | Useful when |
|---|---|
| `BAYF` (Bay Fair) | Origin is on Orange; Blue line from Bay Fair is often faster to SF |
| `19TH` (19th St Oakland) | Orange → SFO/Millbrae line; 1-min same-platform transfer |
| `MCAR` (MacArthur) | Yellow/Orange convergence point |
| `WOAK` (West Oakland) | Last East Bay stop before Transbay Tube; all SF-bound trains stop here |

### Key insight

The BART trip planner's single suggestion missed the Bay Fair → Blue line transfer, which arrived at Embarcadero 13 minutes earlier than the "direct" suggestion. Always probe candidate transfer stations manually when the user has a hard deadline.

---

## Finding station codes

```bash
curl -s "https://api.bart.gov/api/stn.aspx?cmd=stns&key=MW9S-E7SL-26DU-VV8V&json=y" | python3 -c "
import json, sys
data = json.load(sys.stdin)
for s in data['root']['stations']['station']:
    print(s['abbr'], '-', s['name'])
"
```
