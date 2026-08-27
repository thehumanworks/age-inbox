# age inbox

The URL is the secret. No server stores ciphertext.

The vault is whatever you encrypt, typically a `.env`. Names live inside that text, so they are encrypted with the values. The URL is only `to=` (public key) and `c=` (ciphertext).

- **Invite** (owner): publish your `age1` public key as a URL.
- **Seal** (writer): zstd-compress the vault, then age-encrypt it.
- **Append**: seal again against a URL that already has `c=`. Writers do not need the private key.
- **Open** (owner): decrypt with the matching `AGE-SECRET-KEY-1`.
- **Compact** (owner): merge packed vault chunks into one zstd+age file.

https://thehumanworks.github.io/age-inbox/

Crypto is [age-encryption 0.3.0](https://github.com/FiloSottile/typage) and [zstd-wasm 0.0.27](https://github.com/bokuweb/zstd-wasm), both vendored. Interoperable with `age` and `zstd`.

Payload is **zstd (level 19)** of the vault bytes, then age. Older JSON `{secrets:[{n,v}]}` blobs still open, rendered as `NAME=value` lines.

## CLI

Needs `age`, `age-keygen`, and `zstd` on PATH.

```
./inbox init
./inbox invite
cat .env | ./inbox seal --url "$INVITE"
./inbox open "$SEALED"
./inbox compact "$SEALED"
```

By default the CLI prints `https://thehumanworks.github.io/age-inbox/#to=…` links. Set `AGE_INBOX_BASE=` (empty) for the `ageinbox:1?…` scheme.

Identity lives at `.age-inbox/identity`, or `$AGE_INBOX_IDENTITY`.

## URL

```
https://thehumanworks.github.io/age-inbox/#to=<age1>&c=<urlsafe-base64>
```

`c` is one age file, or an AIP1 pack of them. Invite has `to` only.
