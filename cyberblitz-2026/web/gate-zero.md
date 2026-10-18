# Gate Zero

| | |
|---|---|
| **Category** | Web |
| **Points** | 50 |
| **Author** | Anonymous |
| **Solves** | 44 |
| **Files** | None (remote only) |

> A passenger was assigned to Terminal Zero, a place the live kiosk denies exists. Find what the kiosk remembers when the network disappears.

Targets: `http://45.77.247.35:22006` / `http://5.223.64.53:22006`

## Recon

The challenge is an "Afterlight Airways" terminal operations kiosk with two pages:

- **Departures** — a live board showing "No active disruptions" / "Schedule clear". The footer notes `Live feed synchronized / service worker enabled`, and a banner reads `OFFLINE-READY FEED`.
- **Dispatch Control** — a "Recovery intake" page that accepts a POST request:

  ```
  POST /dispatch
  Content-Type: application/json

  {"proof": "<archived proof>"}
  ```

  The page states the proof value "must come from the archived record" — i.e. something recoverable from the client-side offline cache, not the live API.

## Finding the service worker

The "offline-ready" wording pointed straight at a Service Worker caching assets for offline use. Fetching `/sw.js` confirmed this:

```javascript
const CACHE = 'gate-zero-v5';

self.addEventListener('install', event => {
  event.waitUntil(
    caches.open(CACHE).then(cache => cache.addAll([
      '/archive/flight-prior.json',
      '/archive/flight.json',
      '/dispatch'
    ]))
  );
});
```

This revealed two precached JSON routes that aren't linked anywhere in the UI.

## Extracting the archive

Visiting both routes directly returned flight records, each carrying its own `proof` field:

**`/archive/flight.json`**
```json
{"flight": "AL2602", "gate": "T0-00", "status": "last_synced", "proof": "offline-cache-remembers-T0-00"}
```

**`/archive/flight-prior.json`**
```json
{"flight": "AL2602", "gate": "T0-08", "status": "retired", "superseded_by": "/archive/flight.json", "proof": "offline-cache-remembers-T0-08"}
```

Two candidate proofs, since two cached records exist. The `flight-prior.json` record (marked `retired`, pointing forward via `superseded_by`) looked like the more literal "archived" record, so it was tried first — but it was rejected:

```
POST /dispatch → 403 Forbidden
{"error": "dispatch proof rejected"}
```

## Solve

Submitting the proof from `flight.json` instead (the record referencing gate `T0-00` — "Terminal Zero") succeeded:

```javascript
fetch('/dispatch', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ proof: 'offline-cache-remembers-T0-00' })
})
  .then(r => r.text())
  .then(console.log);
```

```json
{"flag": "CYBLTZ2026{0ffL1n3_b04rd_n3v3r_f0rg07}"}
```

## Flag

```
CYBLTZ2026{0ffL1n3_b04rd_n3v3r_f0rg07}
```

## Takeaway

Service workers that precache a fixed asset list (`cache.addAll([...])`) are a straightforward way to leak hidden/unlinked routes — `/sw.js` is always worth checking first on any "offline-capable" web challenge. The two competing JSON records here were a red herring in naming: "archived" didn't mean the explicitly `retired` file, but the one still tied to the gate (`T0-00` / Terminal Zero) named in the prompt.
