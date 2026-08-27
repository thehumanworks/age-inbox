# Task breakdown — age inbox Phase 1

Audience: AI agents. Execute in order. Do not start N+1 until N’s done-when is true. Each task is one PR unless marked “same PR”.

Shared constraints (every task):

- Read `spec/PRD.md` + `spec/ARCHITECTURE.md` first.
- Never put plaintext, identity, or secret names on the store or in new URLs.
- Never invent Phase 2 write tokens.
- Prove with real `age` and `zstd`, not mocks.
- ESM imports in `docs/zstd/` must include `.js` extensions (browser 404 otherwise).
- GitHub Pages publishes `docs/`. Do not put the worker or spec into `docs/` except if you intentionally want them public (spec may stay in `spec/`).

---

## T0 — Vectors and prove harness

**Do:** Create `test/vectors/` with a dedicated `age-keygen` identity used only in tests (label it TEST, not Tomas’s key). Create `vault.env`. Script `test/gen-vector.py` that seals inline (zstd -19 + age) and writes `cipher.age`, `id.txt` (algorithm from architecture). Create `test/prove.sh` that currently tests inline seal/open/404/no-`n=` against `./inbox` (today’s CLI).

**Done-when:** `test/prove.sh` exits 0 on this repo’s current CLI (inline `c=`). Vector files committed. README says how to regenerate.

---

## T1 — Content-addressed id helper

**Do:** Implement id function once (Python + TypeScript identical). Add a fixture test: known 16-byte digest → known 26-char Crockford id. Use it from T0’s `id.txt`.

**Done-when:** Python and TS tests pass on the same vector. Function is imported by CLI and worker later, not copy-pasted a third time.

---

## T2 — Worker + R2 stub

**Do:** `worker/` with wrangler, POST/GET as specified, in-memory or miniflare map if R2 binding is awkward in tests. Reject non-age, 413 over 1 MiB, CORS, immutable GET cache header, idempotent POST.

**Done-when:** `wrangler` tests (or vitest + miniflare) cover the failure table for the store. `curl` against `wrangler dev` can POST a fixture `cipher.age` and GET the same bytes back. No deploy required yet.

---

## T3 — CLI talks to the store

**Do:** Extend `inbox seal` to POST and print `#v=` (default). `--inline` keeps `#c=`. `inbox open` GETs `#v=`. Env `AGE_INBOX_STORE`. Keep invite/init. Keep opening legacy `#c=`. Wrong key / missing id → `404 Not Found`. Store down → exit 2, not 404.

**Done-when:** `AGE_INBOX_STORE=http://127.0.0.1:8787 test/prove.sh` (updated) seals a `.env`, prints a URL under 200 chars with `v=` and no `n=` / no `c=`, opens it, CLI↔inline interop still works. `py_compile` clean.

---

## T4 — Browser uses the store

**Do:** Split UI JS into `docs/app.js` + `docs/config.js` (`STORE`, `UI_BASE`). Seal POSTs, sets `location.hash` to `#v=`. Open on load if hash has `v=`. Invite remains `#to=`. Keep opening `#c=`. Identity never sent. Store down ≠ vault 404.

**Done-when:** Against `wrangler dev` + `python -m http.server` in `docs/`, a manual or Playwright path: generate → invite → paste vault → seal → hash is `#v=` only → open in a fresh load with the same identity. Generate still works if STORE is unreachable.

---

## T5 — Beautiful UI

**Do:** Replace the prototype look. Follow PRD §9. CSS in `docs/styles.css`. Designed empty / invite / sealed / opened / 404 / store-down. Copyable chips for invite and `#v=` link. Vault editor is the hero. Light + dark. Mobile 320px. Do not add a component framework unless it stays tiny (no Next, no React).

**Done-when:** Screenshots in the PR: desktop dark, desktop light, mobile sealed state. Another human (or UI Lead) would not call it a bootstrap form. Lighthouse a11y not embarrassing (labels, contrast). No secret in `document.title`.

---

## T6 — Playwright e2e

**Do:** `test/e2e` against UI + local worker. Cases: generate+invite; seal+open roundtrip; wrong key → “404 Not Found”; missing `v` → 404; store down message; no `n=` in sealed hash.

**Done-when:** `npx playwright test` green in CI (GitHub Actions) on PR.

---

## T7 — Deploy

**Do:** Document wrangler deploy + R2 bind + set `docs/config.js` production STORE. Deploy worker. Point Pages at `docs/` (already). Smoke: seal from production UI, open the short link in a private window with the identity.

**Done-when:** Production invite and `#v=` link work from a phone-sized viewport. README has the real STORE URL. Tomas has been told the worker URL and that identity loss is forever.

---

## T8 — Docs for humans

**Do:** Rewrite root `README.md`: what it is, invite/seal/open, CLI, threat model in 8 lines, link `spec/`. Remove stale “URL is the ciphertext” as the default story. Keep “host never sees plaintext.”

**Done-when:** A stranger can use UI + CLI from the README alone. Spec still matches the code (agent must re-read and patch spec if they drifted).

---

## Parallelism

```
T0 → T1 → T2 → T3 → T4 → T5 → T6 → T7 → T8
           └───── T1 is needed by T2 and T3
```

T5 may start after T4’s behavior works (even if ugly). Do not restyle before store seal works.

T7 needs Tomas for `wrangler login` / Cloudflare account. Stop and say so; do not fake a store.

## Definition of done (whole Phase 1)

PRD §11 acceptance list, plus CLI help matches commands, plus `spec/` updated if the id alphabet or routes changed.
