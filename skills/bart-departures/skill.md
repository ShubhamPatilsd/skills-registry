---
name: bart-departures
description: Look up real-time BART train departures from any station using the public BART API
version: 1.0.0
tags: [api, transit, real-time]
author: shubhampatilsd
created: 2026-04-21
---

When the user asks for the next BART train, upcoming BART departures, or real-time BART schedule from a station, use the public BART API to fetch live departure estimates.

## API Details

- **Base URL:** `https://api.bart.gov/api/etd.aspx`
- **Public API key:** `MW9S-E7SL-26DU-VV8V` (no registration required)
- **Station codes:** Use the 4-letter BART station abbreviation (e.g. `FRMT` for Fremont, `EMBR` for Embarcadero, `12TH` for 12th St Oakland, `DALY` for Daly City)

## How to look up departures

Run this curl command, substituting the correct station code:

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

## Presenting results

Group departures by platform and present as a table with columns: Destination, Line (color), and minutes until departure.

## Finding station codes

If the user gives a station name but you don't know the code, fetch the full station list:

```bash
curl -s "https://api.bart.gov/api/stn.aspx?cmd=stns&key=MW9S-E7SL-26DU-VV8V&json=y" | python3 -c "
import json, sys
data = json.load(sys.stdin)
for s in data['root']['stations']['station']:
    print(s['abbr'], '-', s['name'])
"
```
