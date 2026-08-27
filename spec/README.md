# Age inbox — implementation spec

Audience: AI coding agents. Humans can read it. Do not improvise around this contract.

Read in this order, then implement in `TASKS.md` order:

1. [PRD.md](PRD.md) — product, threat model, acceptance
2. [ARCHITECTURE.md](ARCHITECTURE.md) — components, bytes, APIs, CLI
3. [TASKS.md](TASKS.md) — ordered work, done-when, tests

Phase 1 only. Phase 2 (stable vault id, write capability) is specified as out of scope so you do not invent it.

Current shipped app (client-only, ciphertext in the URL hash) lives in `docs/` + `inbox`. Phase 1 replaces URL-as-blob with store-as-blob. Keep crypto identical.
