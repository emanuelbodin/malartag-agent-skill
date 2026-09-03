---
name: Mälartåg compensation claim
description: >
  Use this when filing a Mälartåg delay or cancellation compensation claim
  after the user explicitly approves that trip. Do not use this to poll
  departures or watch live traffic.
---

# Mälartåg compensation claim

File a delay/cancellation compensation claim for a Mälartåg trip the user already knows about. Never fetch live departures. Never POST until they explicitly approve that specific trip.

Do not put personal data (name, personnummer, ticket number, phone, address, email) in this skill. Those live in the assistant's memory.

## When this applies

The user (or another watcher) has a concrete trip: from station, to station, advertised departure, and whether it was cancelled or how late it arrived. Your job is only the claim.

## Station catalog

[`stations.json`](./stations.json) is the source of truth. Each entry:

- `name`: display name (e.g. `Uppsala C`)
- `code`: Trafikverket location signature (e.g. `U`, `Cst`, `Gä`)
- `id`: UUID for `departureStationId` / `arrivalStationId`

Look up IDs by `code` or `name`. Treat codes as case-insensitive. If a station cannot be mapped to an `id`, do not file; ask which catalog station to use.

## When a claim is worth filing

Swedish regional delay compensation is typically available from about 20 minutes late *to the arrival station*, or when the train was cancelled. Do not file just because the departure was 20 minutes late if arrival was under the threshold.

Always ask for explicit approval of that trip (yes/no) before POSTing.

## Endpoint

`POST https://evf-regionsormland.preciocloudapp.net/api/Claims`

## Body

Fill customer and ticket from assistant memory, not from this file. Resolve station UUIDs from `stations.json`. Example: Uppsala C `U` → `cf09cbb1-fd82-4b83-9c09-87bc8fc2f018`.

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
  "departureDate": "<advertised local clock as …T22:12:00.000Z, e.g. 2026-08-31T22:12:00.000Z>",
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

`ticketType` 1 is a period ticket. `departureDate` is the **advertised local clock time in Europe/Stockholm**, written as an ISO-8601 string ending in `Z` (e.g. a 22:12 departure → `2026-08-31T22:12:00.000Z`). **Do not convert to UTC.** EVF displays the clock fields as-is; converting 22:12+02:00 to `20:12Z` made the claim show the wrong train time. Follow this template (`trainNumber` 0) unless the API requires the advertised train ident.

## After posting

Tell the user the outcome in Swedish: success or the error body. Do not retry a successful claim.

## Configuration (assistant memory)

Store per-user, never in the skill:

- Customer payload for claims
- Period ticket number
- Default from/to stations if they always commute the same hop

## Voice

Speak Swedish to the user. Do not mention skills or internal filenames unless they ask.
