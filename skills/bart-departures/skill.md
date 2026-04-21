---
name: bart-departures
description: Look up real-time BART train departures and find all viable routes (including transfers) from any station to any destination by a deadline
version: 2.2.0
tags: [api, transit, real-time]
author: shubhampatilsd
created: 2026-04-21
---

When the user asks for the next BART train, upcoming BART departures, or real-time BART schedule from a station, use the public BART API to fetch live departure estimates.

## API Details

- **Base URL:** `https://api.bart.gov/api/etd.aspx` (real-time) and `https://api.bart.gov/api/sched.aspx` (schedules)
- **Public API key:** `MW9S-E7SL-26DU-VV8V` (no registration required)
- **Station codes:** Use the 4-letter BART station abbreviation (e.g. `FRMT` for Fremont, `EMBR` for Embarcadero, `12TH` for 12th St Oakland, `DALY` for Daly City)
- **Time format:** The API accepts `time` in `H:MMam/pm` format (e.g. `3:46pm`). API responses return times as `HH:MM AM/PM`. Always normalize before passing back to the API.

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

Run this script, substituting `ORIG` and `DEST` with the correct station codes:

```python
import subprocess, json, sys, time

ORIG = "FRMT"   # change as needed
DEST = "EMBR"   # change as needed
KEY = "MW9S-E7SL-26DU-VV8V"
TRANSFERS = ["BAYF", "19TH", "MCAR", "WOAK"]

def fetch(url, retries=3):
    for i in range(retries):
        r = subprocess.run(["curl", "-s", url], capture_output=True, text=True)
        if r.stdout.strip():
            return json.loads(r.stdout)
        time.sleep(0.5)
    raise Exception(f"Empty response after {retries} retries: {url}")

def normalize_time(t):
    """Convert API response time '03:46 PM' -> '3:46pm' for use as API query param."""
    time_part, ampm = t.strip().rsplit(" ", 1)
    h, m = time_part.split(":")
    return f"{int(h)}:{m}{ampm.lower()}"

def get_trip(orig, dest, time_param="now"):
    url = f"https://api.bart.gov/api/sched.aspx?cmd=depart&orig={orig}&dest={dest}&date=now&time={time_param}&key={KEY}&json=y&b=0&a=1"
    data = fetch(url)
    trips = data["root"]["schedule"]["request"]["trip"]
    if isinstance(trips, dict): trips = [trips]
    t = trips[0]
    legs = t.get("leg", [])
    if isinstance(legs, dict): legs = [legs]
    return t, legs

def check_transfer(transfer):
    t1, legs1 = get_trip(ORIG, transfer)
    depart_orig = t1["@origTimeMin"]
    arrive_transfer = legs1[-1]["@destTimeMin"]
    train1 = legs1[-1]["@trainHeadStation"]

    # Use actual arrival time from leg 1 — never hardcode or estimate this
    t2, legs2 = get_trip(transfer, DEST, normalize_time(arrive_transfer))
    depart_transfer = t2["@origTimeMin"]
    arrive_dest = t2["@destTimeMin"]
    train2 = legs2[-1]["@trainHeadStation"]

    same_train = (train1 == train2)
    route_type = "direct (no transfer)" if same_train else f"transfer at {transfer}"
    return {
        "depart_orig": depart_orig,
        "arrive_transfer": arrive_transfer,
        "depart_transfer": depart_transfer,
        "arrive_dest": arrive_dest,
        "route_type": route_type,
        "valid": depart_transfer >= arrive_transfer  # False = missed connection
    }

def check_direct():
    t, legs = get_trip(ORIG, DEST)
    note = "direct" if len(legs) == 1 else f"BART planner transfer at {legs[0]['@destination']}"
    return {"depart_orig": t["@origTimeMin"], "arrive_dest": t["@destTimeMin"], "route_type": note, "valid": True}

results = []
for transfer in TRANSFERS:
    try:
        results.append(check_transfer(transfer))
    except Exception as e:
        print(f"Warning: skipping {transfer} — {e}", file=sys.stderr)
try:
    results.append(check_direct())
except Exception as e:
    print(f"Warning: direct failed — {e}", file=sys.stderr)

results.sort(key=lambda r: r["arrive_dest"])
print(f"\n{'Route':<30} {'Departs':>8} {'Arrives':>10}  Valid?")
print("-" * 60)
for r in results:
    valid = "✅" if r["valid"] else "❌ missed connection"
    print(f"{r['route_type']:<30} {r['depart_orig']:>8} {r['arrive_dest']:>10}  {valid}")
```

**How to interpret results:**
- `valid = False` means the connecting train departs before you arrive — discard these
- `direct (no transfer)` means the "transfer station" is just a stop on a through-train — deduplicate against the direct result
- Present only valid routes, ranked by earliest arrival

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
