---
name: Mälartåg delay watch
description: >
  Use this when checking cancelled or delayed Mälartåg via a Trafikverket
  announcements API, alerting a commuter on the Stockholm–Uppsala–Gävle
  corridor (or a similar corridor), and filing delay compensation only after
  the user explicitly approves that trip.
---

# Mälartåg delay watch

Watch cancelled and delayed Mälartåg, ping the user only when something new happens on their corridor, and file delay compensation only after they approve that specific trip.

Do not put personal data (name, personnummer, ticket number, phone, address, email) in this skill. Those live in the assistant's memory for the person you are helping.

## When this applies

Swedish regional trains branded Mälartåg. Default corridor is Stockholm C (`cst`), Uppsala C (`u`), Gävle C (`gä`). The same recipe works for another corridor if the user names other station signatures.

## Data sources

Announcements API (reference implementation):

- Base: `https://trafikverket-api-production.up.railway.app`
- Related code: https://github.com/emanuelbodin/trafikverket-api
- Cancelled: `GET /api/announcements/departures/{station}?canceled=true`
- Delayed: `GET /api/announcements/departures/{station}?delayed=true`
- Stations: `u`, `cst`, `gä` (URL-encode `gä`)

Each item looks like Trafikverket `TrainAnnouncement`: `advertisedTrainIdent`, `advertisedTimeAtLocation`, `estimatedTimeAtLocation`, `canceled`, `fromName`, `toName`, `toLocation`, `viaToLocation`, `productInformation`, `operator`, `otherInformation`, `trackAtLocation`.

Mälartåg filter: `productInformation` description contains `Mälartåg`. Operator is typically `TDEV`.

If the proxy is down, fall back to Trafikverket's open `TrainAnnouncement` API and filter the same way. Tell the user about an outage once, not on every poll.

## Corridor filter

Keep a train only if both ends of the relevant hop are in the corridor set (Stockholm C / Uppsala C / Gävle C), using `fromName`, `toName`, via locations, and signatures `Cst`, `U`, `Gä`.

Drop Mälartåg whose destination is outside the corridor (Eskilstuna, Norrköping, Sala, Örebro, Arboga, Tierp, etc.) unless the user is actually travelling that hop.

Deduplicate the same train seen at several stations. Key: `trainNumber|advertisedLocalDate|from|to`.

## Alerting

Poll on a schedule the user asked for (default every 15 minutes). Stay silent when nothing new happened. Do not send "inga förseningar".

If something new:

- Write a short message in Swedish: train number, from → to, advertised time in `Europe/Stockholm`, cancelled vs delayed (minutes if `estimatedTimeAtLocation` exists), track and cause when present.
- Re-alert only if it got materially worse (much more delay, or it became cancelled).
- Persist alerted keys in a local seen-file (for example `/workspace/malartag-seen.json`).

Delay minutes: difference between estimated and advertised time at that station, in the user's timezone.

## Compensation

Swedish regional delay compensation is typically available from about 20 minutes late *to the arrival station*. Do not offer a claim just because the departure was 20 minutes late if arrival is still under the threshold.

If a corridor train is cancelled, or delayed by 20+ minutes at arrival, offer to file and ask for explicit approval. Never POST a claim until the user approves that trip.

Endpoint: `POST https://evf-regionsormland.preciocloudapp.net/api/Claims`

Station UUIDs:

- Uppsala (`u`): `cf09cbb1-fd82-4b83-9c09-87bc8fc2f018`
- Stockholm C (`cst`): `f4d25596-a9f9-41a1-b200-713439d92fc4`
- Gävle C (`gä`): `c1ed2e95-5cc2-4e9d-a89b-fb27f01ad527`

Body shape (fill customer and ticket from assistant memory, not from this file):

```json
{
  "id": "00000000-0000-0000-0000-000000000000",
  "confirmDuplicate": null,
  "payoutOption": "SUS",
  "arrivalStationId": "<to station uuid>",
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
  "departureStationId": "<from station uuid>",
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
- Corridor station signatures
- Period ticket number and expiry
- Customer payload for claims
- Poll cadence
- Path to the seen-file

## Voice

Speak Swedish to the user. Do not mention routines, skills, or internal filenames unless they ask.
