# PRD — age inbox (store-backed vault)

Status: approved direction (Phase 1 = immutable encrypted blobs + short links).
Owner: Tomas / Eureka.
Audience: implementing agents.
Live UI today: https://thehumanworks.github.io/age-inbox/
Repo: https://github.com/thehumanworks/age-inbox

## 1. Problem

The current product is correct and too small.

- Ciphertext in the URL hash does not scale. A real `.env` blows past chat and browser URL limits.
- Sharing is painful (800–2000+ character links).
- There is no persistence beyond “I still have the link.”
- Compact/append history confused users. Seal already writes one vault; the remaining pain is *where the bytes live*.

Users need to hand a writer an invite, get a vault back, open it from any device, and keep using the same mental model: **age is the lock, not the host.**

## 2. Product one-liner

A write-only invite (public age key) plus a sealed vault (encrypted blob in an object store) plus a short open link. The host never sees plaintext. The age identity is the only capability that opens.

## 3. Who

| Role | They have | They do |
| --- | --- | --- |
| Owner | `AGE-SECRET-KEY-1` | `init`, share invite, `open` |
| Writer | invite URL only | paste a `.env` / vault text, `seal`, send back the short link |
| Bystander | short open link | can fetch ciphertext, cannot decrypt |

Owner and writer may be the same person.

## 4. Goals (Phase 1)

1. Persist ciphertext in an object store. Plaintext never leaves the client unencrypted.
2. Share links short enough for SMS / Slack / GitHub comments (`#v=<id>`, ~30–50 chars of id).
3. One seal = one immutable blob = one link. No packing extra age files.
4. Works in the browser and the CLI, interoperable (same bytes, same URLs).
5. Accessible from any device with the link + the identity. No account, no password database.
6. UI is something you would show a person, not a debug form.
7. Tested and documented so another agent can change it without rereading this chat.

## 5. Non-goals (do not implement)

- Phase 2: stable vault id, in-place overwrite, write tokens, listing an owner’s vaults.
- Accounts, email magic links, OAuth, “forgot my key.”
- Server-side decrypt, custodial recovery, key escrow.
- Sharing the age identity through this app.
- Search, tags, or secret names in the URL / store metadata.
- Multi-recipient age files (one `age1` recipient per vault).
- File attachments larger than the store max (1 MiB). This is a vault, not Drive.
- AIP1 packing of multiple age files on *new* seals (still *read* old packed URLs).
- Putting `n=` (names) in any new URL.
- A general-purpose password manager.

## 6. User journeys

### 6.1 Owner creates an inbox

1. Open the site or `inbox init`.
2. Get an identity file (downloadable) and an invite URL:
   `https://<ui>/#to=age1…`
3. Invite contains *only* `to`. No names.

### 6.2 Writer seals a vault

1. Open invite (or `inbox seal --url INVITE`).
2. Paste vault text (`.env` or any UTF-8). Names stay *inside* the text.
3. Client: UTF-8 → zstd -19 → age to `to=` → POST ciphertext to store.
4. Client shows/copies a short URL: `https://<ui>/#v=<id>`
5. Writer sends that to the owner.

### 6.3 Owner opens

1. Open `#v=<id>` (or `inbox open URL`).
2. Paste / load identity.
3. Client GET blob, age-decrypt, zstd-decompress, show vault.
4. Failures that are about the blob or the key: UI and CLI say `404 Not Found` and nothing else.

### 6.4 Owner edits

Edit the vault in the UI (or edit `.env`) and seal again. That uploads a *new* blob and mints a *new* `#v=`. The old link still opens the old snapshot. Explain this in the UI. Do not silently overwrite.

### 6.5 Offline / no store (compat)

CLI and UI MUST still open:

- Current hash URLs with `c=` (inline ciphertext).
- Legacy JSON `{secrets:[{n,v}]}` inside zstd, rendered as `NAME=value` lines.
- AIP1 packs of age files (read only).

New seals MUST NOT produce `c=` inline except as a documented `--inline` CLI escape hatch for air-gapped use.

## 7. Information in URLs

| Param | Where | Contents |
| --- | --- | --- |
| `to` | invite only | `age1…` recipient |
| `v` | sealed link | store object id |
| `c` | legacy / `--inline` only | urlsafe-base64 of age file |

MUST NOT appear in new sealed URLs: `n`, secret names, recipient (optional: you MAY omit `to` on `#v=` because the age file already names the recipient). Prefer `#v=` only so the open link does not advertise who can decrypt.

Invite: `#to=age1…` only.

## 8. Threat model

Assume: store operator, GitHub Pages, Cloudflare, writers, anyone with the link.

| Asset | Protection |
| --- | --- |
| Vault plaintext | zstd then age X25519 to owner recipient. Never sent to store or UI host. |
| Age identity | Local only (file / textarea). Never uploaded. |
| Secret names | Inside vault plaintext, hence encrypted. |
| Ciphertext | Public-read by id. Knowing `v` is like holding the URL today. |
| Integrity | Age file authenticated. Content-addressed id (see architecture). |
| Overwrite | Impossible in Phase 1: new content → new id. |
| Store abuse | POST must look like an age v1 file; size cap; rate limit. |
| Oracle | Open/decrypt/wrong-key/missing-blob/garbage all return `404 Not Found`. |

The store is **honest but curious**. A compromised store can delete or withhold blobs. It cannot decrypt. Document that availability ≠ confidentiality.

Identity loss is unrecoverable. UI MUST say that once, clearly, at generate time.

## 9. UX / visual bar

The current page is a working prototype. Phase 1 replaces it. “Beautiful” is a requirement, not polish-if-time.

MUST:

- Feel like a small instrument, not an admin dashboard. No card grids of metrics, no sidebar chrome, no generic shadcn template look.
- One column, max ~40rem, generous type, lots of air.
- Pair a literary serif (or distinctive grotesque) for titles with a monospace for vault and keys. Do not use system-ui as the display face.
- Vault editor is the hero: large, lined or softly ruled, placeholder is a real `.env` fragment.
- Invite and sealed link are copyable chips, not raw overflowing hash dumps as the primary UI.
- Motion: one short easing on reveal/copy. No page-load animation festival.
- Light and dark both designed (not “invert”). Default follows `prefers-color-scheme`.
- Mobile: 320px wide usable. Buttons ≥ 44px. Hash links open and work.
- Empty, invite, sealed, opened, and 404 states are designed, not leftover form errors.
- Copy is short and human. No “payload”, “AIP1”, “zstd-wasm” in the primary UI. Footer may credit crypto.
- Generate identity works even if the store is down. Seal/open that need the store say the store is unreachable (that is not a 404-of-the-vault).
- Never echo the vault into `document.title` or the URL.

Accessibility MUST: keyboard through generate → invite → vault → seal → copy; visible focus; `label`s; contrast ≥ WCAG AA; identity field `autocomplete=off`.

## 10. CLI bar

Ship `inbox` as the human and agent interface.

```
inbox init
inbox invite
inbox seal [--url INVITE] [--store URL] [--inline]
inbox open URL
inbox to URL
inbox compact URL    # read-only migration of old packed c= URLs; do not use for store blobs
```

MUST:

- stdlib Python 3.11+ plus `age`, `age-keygen`, `zstd` on PATH. No pip crypto deps.
- Store HTTP via stdlib `urllib`.
- Identity: `./.age-inbox/identity` or `$AGE_INBOX_IDENTITY`.
- Default UI base: `https://thehumanworks.github.io/age-inbox/` (override `AGE_INBOX_BASE`, empty → `ageinbox:` scheme).
- Default store: documented production worker URL (override `AGE_INBOX_STORE`).
- `seal` reads vault on stdin, prints exactly one URL on stdout, warnings on stderr.
- `open` writes vault to stdout, nothing else on stdout.
- Exit 1 + `404 Not Found` on stderr for blob/crypto failure.
- Interop: a vault sealed in the CLI opens in the browser; the reverse too.

## 11. Acceptance (Phase 1 done)

An agent may merge only when all of these are true:

1. Invite URL has only `to=age1…`. Sealed URL has only `v=<id>` (or documented `--inline` `c=`).
2. A 2–20 line `.env` sealed via UI and via CLI opens in the other, values exact.
3. Sealed URL length for a 5-line `.env` is under 200 characters on the production UI host.
4. Store object is a well-formed age v1 file. `age -d` + `zstd -d` recover the vault with the owner identity.
5. GET of a missing id, wrong identity, raw-age (no zstd), and truncated garbage all surface as `404 Not Found` with no “zstd” / “raw age” / “no identity matched” text.
6. POST of non-age bytes is rejected by the store (4xx). POST above 1 MiB rejected.
7. Re-POST of the same ciphertext returns the same id (content-addressed).
8. Identity never appears in network logs of the app (no identity in query, hash, or POST body).
9. Playwright (or equivalent) covers generate, invite copy, seal, open, 404; CLI prove script in CI.
10. README + this spec stay true. UI matches §9 (screenshot in the PR).
11. Old `#to=&c=` links still open.

## 12. Open questions (do not block Phase 1)

- Custom domain vs `*.workers.dev` + GitHub Pages. Default: Pages for UI, worker for store.
- Phase 2 write-capability format. Do not invent it now.
