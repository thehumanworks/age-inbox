# age inbox

The URL is the secret. No server stores ciphertext.

- **Invite** (owner): publish your `age1` public key as a URL. Writers can encrypt to you.
- **Seal** (writer): zstd-compress a named secret, then age-encrypt it. The result is a longer URL with `c=`.
- **Append**: seal again against a URL that already has `c=`. Writers do not need the private key. Each new secret is another age file in an AIP1 pack.
- **Open** (owner): decrypt with the matching `AGE-SECRET-KEY-1`. Lists every secret by name.
- **Compact** (owner): decrypt every packed file, merge into one zstd JSON blob, one age file, shorter URL.

Web UI (static, ciphertext in the hash so the host never sees it):

https://thehumanworks.github.io/age-inbox/

Crypto in the browser is [age-encryption 0.3.0](https://github.com/FiloSottile/typage), vendored in `docs/age.js`, plus [zstd-wasm 0.0.27](https://github.com/bokuweb/zstd-wasm) vendored in `docs/zstd/`. Interoperable with the `age` and `zstd` CLIs.

Payload is **zstd (level 19)** of `{"secrets":[{"n":"NAME","v":"…"}]}` then age-encrypted. Raw age (plaintext, no zstd) is rejected.

## CLI

Needs `age`, `age-keygen`, and `zstd` on PATH.

```
./inbox init
./inbox invite --name STRIPE
echo 'sk_live_…' | ./inbox seal --url "$INVITE" --name STRIPE
echo 'sk-…' | ./inbox seal --url "$SEALED" --name OPENAI
./inbox open "$SEALED"
./inbox compact "$SEALED"
```

By default the CLI prints `https://thehumanworks.github.io/age-inbox/#to=…` links. Set `AGE_INBOX_BASE=` (empty) for the `ageinbox:1?…` scheme, or point `AGE_INBOX_BASE` at another origin.

Identity lives at `.age-inbox/identity` in the current directory, or `$AGE_INBOX_IDENTITY`.

## URL

```
https://thehumanworks.github.io/age-inbox/#to=<age1>&n=<comma-names>&c=<urlsafe-base64>
```

- `to` is the recipient public key.
- `n` is comma-separated visible names (not secret).
- `c` is either one age file, or an AIP1 pack: magic `AIP1` + repeated (`u32be` length + age file).

Invite has `to` only. Files do not fit; tokens do. Compact after a few appends if the URL grows past ~2000 characters.
