# Mälartåg compensation skill

An agent skill for **filing Mälartåg delay/cancellation compensation** after the user explicitly approves a specific trip.

It does not poll departures or watch live traffic. Ticket numbers and personal data live in the assistant's memory, not here.

## What it does

1. Maps from/to stations to claim UUIDs via [`stations.json`](./stations.json).
2. Builds the Region Sörmland EVF claim body.
3. POSTs only after explicit approval.
4. Typical threshold: cancelled, or about 20+ minutes late *on arrival*.

The recipe is in [`SKILL.md`](./SKILL.md) ([Agent Skills](https://agentskills.io) format).

## Stations

`stations.json` lists Mälartåg stations: `name`, Trafikverket `code`, and claim `id`.

## API

```
POST https://evf-regionsormland.preciocloudapp.net/api/Claims
```

## Install

Point your assistant at this `SKILL.md` (Cursor, Claude, or any client that reads [Agent Skills](https://agentskills.io)). Keep `stations.json` next to it.

## License

MIT. See [LICENSE](./LICENSE).
