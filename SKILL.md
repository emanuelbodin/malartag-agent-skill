---
name: Mälartåg delay watch
description: >
  Use this when checking cancelled or delayed Mälartåg via a Trafikverket
  announcements API, alerting a commuter on any Mälartåg station (or a subset
  they named), and filing delay compensation only after the user explicitly
  approves that trip.
---

# Mälartåg delay watch

Watch cancelled and delayed Mälartåg anywhere on the network, ping the user only when something new happens on the stations they care about, and file delay compensation only after they approve that specific trip.

Do not put personal data (name, personnummer, ticket number, phone, address, email) in this skill. Those live in the assistant's memory for the person you are helping.

## When this applies

Swedish regional trains branded Mälartåg. Default scope is **all stations** in [`stations.json`](./stations.json) (name, Trafikverket `code`, claim `id`). The user may narrow that to a subset in assistant memory; if they have not, watch the whole catalog.

## Station catalog

`stations.json` is the source of truth. Each entry:

- `name`: display name (e.g. `Uppsala C`)
- `code`: Trafikverket location signature used in the announcements API (e.g. `U`, `Cst`, `Gä`)
- `id`: UUID used in delay-compensation claims

Look up claim `departureStationId` / `arrivalStationId` by `code` or `name`. Treat codes as case-insensitive. URL-encode codes when calling the API (`Gä`, `Söö`, `Äkb`, …).

If a signature is missing from the catalog, still show the alert, but do not file a claim until the station can be mapped to an `id`.

## Data sources

Announcements API (reference implementation):

- Base: `https://trafikverket-api-production.up.railway.app`
- Related code: https://github.com/emanuelbodin/trafikverket-api
- `GET /api/announcements/departures/{station}?from={from}&to={to}&canceled=true`
- `GET /api/announcements/departures/{station}?from={from}&to={to}&delayed=true`
- `{station}` is a catalog `code`
- `from` and `to` are ISO-8601 date-times with offset, e.g. `2026-08-28T00:00:00+02:00` (URL-encode). They filter `AdvertisedTimeAtLocation` (`GT` / `LT`). Always send both. Omitting both means no advertised-time filter. Invalid timestamps or `from` after `to` → 400.
- Default poll window: now−24h to now+12h in `Europe/Stockholm`.

Each item looks like Trafikverket `TrainAnnouncement`: `advertisedTrainIdent`, `advertisedTimeAtLocation`, `estimatedTimeAtLocation`, `canceled`, `fromName`, `toName`, `toLocation`, `viaToLocation`, `productInformation`, `operator`, `otherInformation`, `trackAtLocation`.

Mälartåg filter: `productInformation` description contains `Mälartåg`. Operator is typically `TDEV`.

If the proxy is down, fall back to Trafikverket's open `TrainAnnouncement` API and filter the same way. Tell the user about an outage once, not on every poll.

## Station filter

Watched set = the user's subset if they named one, otherwise every `code` in `stations.json`.

Keep a train if it is Mälartåg and it touches the watched set: `locationSignature`, `fromName`/`toName`, or via locations match a catalog name or code.

Deduplicate the same train seen at several stations. Key: `trainNumber|advertisedLocalDate|from|to`.

When polling, fetch cancelled + delayed for each watched station. If the watched set is the full catalog, you may skip tiny stops that never originate trains and still catch those trains at hubs they pass, but prefer fetching every watched code unless rate limits force a hub subset (then say so).

## Alerting

Poll on a schedule the user asked for (default every 15 minutes). Stay silent when nothing new happened. Do not send "inga förseningar".

If something new:

- Write a short message in Swedish: train number, from → to, advertised time in `Europe/Stockholm`, cancelled vs delayed (minutes if `estimatedTimeAtLocation` exists), track and cause when present.
- Re-alert only if it got materially worse (much more delay, or it became cancelled).
- Persist alerted keys in a local seen-file (for example `/workspace/malartag-seen.json`).

Delay minutes: difference between estimated and advertised time at that station, in the user's timezone.

## Compensation

Swedish regional delay compensation is typically available from about 20 minutes late *to the arrival station*. Do not offer a claim just because the departure was 20 minutes late if arrival is still under the threshold.

If a watched train is cancelled, or delayed by 20+ minutes at arrival, offer to file and ask for explicit approval. Never POST a claim until the user approves that trip.

Endpoint: `POST https://evf-regionsormland.preciocloudapp.net/api/Claims`

Resolve station UUIDs from `stations.json`. Example: Uppsala C `U` → `cf09cbb1-fd82-4b83-9c09-87bc8fc2f018`.

Body shape (fill customer and ticket from assistant memory, not from this file):

```json
{
  "id": "00000000-0000-0000-0000-000000000000",
  "confirmDuplicate": null,
  "payoutOption": "SUS",
  "arrivalStationId": "<to station uuid from stations.json>",
  "claimReceipts": [],
  "comment": "",
  "customer": {
    "BIC": "",
    "IBAN": "",
    "bankAccountNumber": "",
    "city": "",
    "clearingNumber": "",
    "email": "",
    "firstName": "",
    "hasIdentityNumber": true,
    "id": "00000000-0000-0000-0000-000000000000",
    "identityNumber": "",
    "mobileNumber": "",
    "postalCode": "",
    "streetNameAndNumber": "",
    "surName": ""
  },
  "departureDate": "<ISO-8601 UTC, e.g. 2026-02-10T07:00:00.000Z>",
  "departureStationId": "<from station uuid from stations.json>",
  "refundType": {
    "id": "00000000-0000-0000-0000-000000000000",
    "name": "Payment via Swedbank SUS"
  },
  "status": 0,
  "ticketNumber": "",
  "ticketType": 1,
  "trainNumber": 0
}
```

`ticketType` 1 is a period ticket. `departureDate` is the advertised departure in UTC.

## Period ticket reminder

If the user has given a period ticket number and an expiry date, remind them the day before it expires (morning local time). Do not put the ticket number in this skill.

## Configuration (assistant memory)

Store per-user, never in the skill:

- API base URL (default: the reference implementation above)
- Optional watched-station codes (default: all of `stations.json`)
- Period ticket number and expiry
- Customer payload for claims
- Poll cadence
- Path to the seen-file

## Voice

Speak Swedish to the user. Do not mention routines, skills, or internal filenames unless they ask.
