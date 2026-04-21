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

- Base URL: https://api.bart.gov/api/etd.aspx
- Public API key: MW9S-E7SL-26DU-VV8V (no registration required, from bart.gov/schedules/developers/api)
- Station codes: 4-letter abbreviations, e.g. FRMT (Fremont), EMBR (Embarcadero), 12TH (12th St Oakland), DALY (Daly City)

## Fetching Departures

Use curl + python3 to call the API and parse the JSON response:

curl -s "https://api.bart.gov/api/etd.aspx?cmd=etd&orig=STATION_CODE&key=MW9S-E7SL-26DU-VV8V&json=y"

Parse the response: root.station[0].etd is a list of destinations, each with an estimate array containing minutes, platform, and color fields.

## Presenting Results

Group by platform, display as a table: Destination | Line (color) | Minutes until departure.

## Finding Station Codes

If the station code is unknown, fetch the full list:

curl -s "https://api.bart.gov/api/stn.aspx?cmd=stns&key=MW9S-E7SL-26DU-VV8V&json=y"

Parse root.stations.station[], each has abbr and name fields.
