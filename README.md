# age inbox

The URL is the secret. No server stores ciphertext.

- **Invite** (owner): publish your `age1` public key as a URL. Writers can encrypt to you.
- **Seal** (writer): encrypt a secret to that key. The result is a longer URL with `c=`.
- **Open** (owner): decrypt with the matching `AGE-SECRET-KEY-1`.

Web UI (static, ciphertext in the hash so the host never sees it):

https://thehumanworks.github.io/age-inbox/

Crypto in the browser is [age-encryption 0.3.0](https://github.com/FiloSottile/typage), vendored in `docs/age.js`. Interoperable with the `age` CLI.

## CLI

Needs `age` and `age-keygen` on PATH.

```
./inbox init
./inbox invite --name STRIPE
echo 'sk_live_…' | ./inbox seal --url "$INVITE"
./inbox open "$SEALED"
```

By default the CLI prints `https://thehumanworks.github.io/age-inbox/#to=…` links. Set `AGE_INBOX_BASE=` (empty) for the `ageinbox:1?…` scheme, or point `AGE_INBOX_BASE` at another origin.

## URL

```
https://thehumanworks.github.io/age-inbox/#to=<age1>&n=<name>&c=<urlsafe-base64 ciphertext>
```

Invite has `to` only. Files do not fit; tokens do.
