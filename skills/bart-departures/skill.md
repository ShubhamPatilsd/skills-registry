---
name: bart-departures
description: Look up real-time BART train departures and find all viable routes (including transfers) from any station to any destination by a deadline
version: 2.1.0
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

### Step 1 — Probe all candidate transfer stations in parallel

Run this single script. It fetches the first leg (origin → transfer), extracts the **actual arrival time** at the transfer, then immediately queries the onward leg (transfer → destination) using that arrival time. This avoids the ghost-train bug where a hardcoded time finds a train that already departed before you arrive.

```bash
python3 - << 'EOF'
import subprocess, json, sys
from concurrent.futures import ThreadPoolExecutor, as_completed

ORIG = "FRMT"
DEST = "EMBR"
KEY = "MW9S-E7SL-26DU-VV8V"
TRANSFERS = ["BAYF", "19TH", "MCAR", "WOAK"]

def fetch(url):
    r = subprocess.run(["curl", "-s", url], capture_output=True, text=True)
    return json.loads(r.stdout)

def check_transfer(transfer):
    # Leg 1: origin -> transfer
    data = fetch(f"https://api.bart.gov/api/sched.aspx?cmd=depart&orig={ORIG}&dest={transfer}&date=now&key={KEY}&json=y&b=0&a=1")
    trips = data["root"]["schedule"]["request"]["trip"]
    if isinstance(trips, dict): trips = [trips]
    t = trips[0]
    legs = t.get("leg", [])
    if isinstance(legs, dict): legs = [legs]
    depart_orig = t["@origTimeMin"]
    arrive_transfer = legs[-1]["@destTimeMin"]
    train1 = legs[-1]["@trainHeadStation"]

    # Leg 2: transfer -> dest, using ACTUAL arrival time from leg 1
    data2 = fetch(f"https://api.bart.gov/api/sched.aspx?cmd=depart&orig={transfer}&dest={DEST}&date=now&time={arrive_transfer}&key={KEY}&json=y&b=0&a=1")
    trips2 = data2["root"]["schedule"]["request"]["trip"]
    if isinstance(trips2, dict): trips2 = [trips2]
    t2 = trips2[0]
    legs2 = t2.get("leg", [])
    if isinstance(legs2, dict): legs2 = [legs2]
    depart_transfer = t2["@origTimeMin"]
    arrive_dest = t2["@destTimeMin"]
    train2 = legs2[-1]["@trainHeadStation"]

    # Detect if it's actually a direct ride (same train, no real transfer)
    same_train = train1 == train2
    route_type = "direct (no transfer)" if same_train else f"transfer at {transfer}"

    return {
        "depart_orig": depart_orig,
        "arrive_transfer": arrive_transfer,
        "depart_transfer": depart_transfer,
        "arrive_dest": arrive_dest,
        "route_type": route_type,
        "transfer": transfer,
        "valid": depart_transfer >= arrive_transfer  # connection is physically possible
    }

def check_direct():
    data = fetch(f"https://api.bart.gov/api/sched.aspx?cmd=depart&orig={ORIG}&dest={DEST}&date=now&key={KEY}&json=y&b=0&a=1")
    trips = data["root"]["schedule"]["request"]["trip"]
    if isinstance(trips, dict): trips = [trips]
    t = trips[0]
    legs = t.get("leg", [])
    if isinstance(legs, dict): legs = [legs]
    transfer_note = "direct" if len(legs) == 1 else f"BART planner transfer at {legs[0]['@destination']}"
    return {"depart_orig": t["@origTimeMin"], "arrive_dest": t["@destTimeMin"], "route_type": transfer_note, "valid": True}

results = []
with ThreadPoolExecutor() as ex:
    futs = {ex.submit(check_transfer, tr): tr for tr in TRANSFERS}
    futs[ex.submit(check_direct)] = "direct"
    for f in as_completed(futs):
        try:
            results.append(f.result())
        except Exception as e:
            print(f"Error: {e}", file=sys.stderr)

results.sort(key=lambda r: r["arrive_dest"])
print(f"\n{'Route':<30} {'Departs':>8} {'Arrives EMBR':>13} {'Valid?':>7}")
print("-" * 65)
for r in results:
    valid = "✅" if r["valid"] else "❌ missed connection"
    print(f"{r['route_type']:<30} {r['depart_orig']:>8} {r['arrive_dest']:>13}  {valid}")
EOF
```

### Step 2 — Validate connections and present results

The script already validates each connection by checking `depart_transfer >= arrive_transfer`. If this is false, the transfer is impossible — the connecting train leaves before you arrive. **Discard invalid routes.**

Also check for **phantom direct routes**: if leg 1 and leg 2 use the same train heading, it means the "transfer station" is just a stop on a through-train — not a real transfer. Label these as `direct (no transfer)` and deduplicate against the BART planner's direct result.

Present only valid routes, ranked by earliest arrival at destination.

### Common BART transfer stations for East Bay → SF trips

| Transfer | Useful when |
|---|---|
| `BAYF` (Bay Fair) | Origin is on Orange; Blue line from Bay Fair is often faster to SF |
| `19TH` (19th St Oakland) | Orange → SFO/Millbrae line; 1-min same-platform transfer |
| `MCAR` (MacArthur) | Yellow/Orange convergence point |
| `WOAK` (West Oakland) | Last East Bay stop before Transbay Tube; all SF-bound trains stop here |

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
