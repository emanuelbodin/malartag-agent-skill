# Mälartåg agent skill

An agent skill for watching **cancelled and delayed Mälartåg** on the full Mälartåg network (or a station subset the user named), alerting only when something new happens, and filing delay compensation **after explicit approval**.

No personal data belongs here. Ticket numbers, names, and national IDs live in the assistant's memory.

## What it does

1. Fetches cancelled and delayed departures from a Trafikverket announcements API.
2. Keeps Mälartåg (`productInformation` = Mälartåg, operator usually `TDEV`) that touch the watched stations.
3. Stays silent when nothing new happened. Pings only on a new cancellation or delay, or if the situation got materially worse.
4. Offers a compensation claim when a train is cancelled or about 20+ minutes late *on arrival*. Never posts without approval.

The recipe is in [`SKILL.md`](./SKILL.md) ([Agent Skills](https://agentskills.io) format). Station names, Trafikverket codes, and claim UUIDs are in [`stations.json`](./stations.json).

## Stations

Default scope is every station in `stations.json` (Arboga, Arlanda C, Eskilstuna C, …, Örebro Södra). An assistant can narrow that list in its own memory.

`code` is the announcements API path segment (URL-encode non-ASCII, e.g. `Gä`). `id` is the UUID for `departureStationId` / `arrivalStationId` on a claim.

## API

Reference implementation: [trafikverket-api](https://github.com/emanuelbodin/trafikverket-api)  
Base URL: `https://trafikverket-api-production.up.railway.app`

```
GET /api/announcements/departures/{station}?from={from}&to={to}&canceled=true
GET /api/announcements/departures/{station}?from={from}&to={to}&delayed=true
```

`{station}` is a `code` from `stations.json`. `from` and `to` are ISO-8601 date-times with offset (example: `2026-08-28T00:00:00+02:00`). Always send them; omitting both applies no advertised-time filter. Default window: last 24 hours through the next 12 hours, Europe/Stockholm.

Compensation: `POST https://evf-regionsormland.preciocloudapp.net/api/Claims`

## Install

Point your assistant at this `SKILL.md` (Cursor, Claude, or any client that reads [Agent Skills](https://agentskills.io)). Keep `stations.json` next to it.

## License

MIT. See [LICENSE](./LICENSE).
