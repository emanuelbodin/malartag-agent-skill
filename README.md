# Mälartåg agent skill

Agent skill för att bevaka **inställda och försenade Mälartåg** på en stationkorridor (standard: Stockholm C, Uppsala C, Gävle C), varna när något nytt händer, och söka förseningersättning **först efter uttryckligt godkännande**.

Inga personuppgifter hör hemma här. Biljettnummer, namn, personnummer och liknande ligger i assistentens minne.

## Vad den gör

1. Hämtar inställda och försenade avgångar från ett Trafikverket-announcements-API.
2. Filtrerar till Mälartåg (`productInformation` = Mälartåg, operator oftast `TDEV`) på den valda korridoren.
3. Tystnar om inget nytt hänt. Pingar bara vid ny inställelse eller försening, eller om läget blivit märkbart sämre.
4. Föreslår ersättningsansökan vid inställt tåg eller ca 20+ min sen *ankomst*. Postar aldrig utan godkännande.

Receptet ligger i [`SKILL.md`](./SKILL.md) (Agent Skills-format).

## API

Referensimplementation: [trafikverket-api](https://github.com/emanuelbodin/trafikverket-api)  
Bas-URL: `https://trafikverket-api-production.up.railway.app`

```
GET /api/announcements/departures/{station}?canceled=true
GET /api/announcements/departures/{station}?delayed=true
```

`station`: `u` (Uppsala C), `cst` (Stockholm C), `gä` (Gävle C, URL-encoda).

Ersättning: `POST https://evf-regionsormland.preciocloudapp.net/api/Claims`

## Installera

Klona eller peka din assistent på den här `SKILL.md` (Cursor / Claude / andra klienter som läser [Agent Skills](https://agentskills.io)).

I Grok Bot / Cursor kan du också spara innehållet som en skill i assistentens bibliotek.

## Licens

MIT. Se [LICENSE](./LICENSE).
