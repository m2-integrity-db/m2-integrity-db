# Security Policy

This repository holds reference data used to detect tampering with Magento
installations. Two things follow from that, and they shape everything below.

The database is itself a target. Anyone who can alter it undetected can make a
compromised installation look clean, which is worse than having no reference data
at all: a false clean bill of health is trusted, an absent one is not. That is why
the manifest is signed and why a consumer should refuse to use data it cannot
verify.

The data contains no code. Only file paths, SHA-256 digests and byte sizes. No
Magento or Adobe source is present and none can be reconstructed from it.

## Reporting a vulnerability

Email **scott@dor.ky** directly. Please do not open a public issue for anything
security-relevant.

Useful things to include, as far as you have them:

- What you found, and what an attacker gains from it
- How to reproduce it — a package name and version, a digest, a signature, or the
  commands you ran
- Whether it is already public anywhere

You should get an acknowledgement within a few days. If a report is confirmed you
will be told what the fix is and when it lands, and credited when it ships unless
you would rather not be.

This is a small project maintained by one person. There is no bounty programme and
no formal service level; what there is, is a direct line to someone who will read
it.

## What is in scope

- Reference data that does not match the artefact Composer actually installs, in a
  way that could hide a modification or manufacture a false one
- Anything that lets the signature be bypassed, forged, or made to verify against
  data it does not cover
- Weaknesses in how the build fetches or verifies upstream archives — for example a
  path that would accept an archive whose published digest did not match
- Anything that could cause a consumer to report an installation as clean when it
  is not

## What is not in scope

- Vulnerabilities in Magento, Adobe Commerce, or any package this database
  describes. Report those to Adobe. This project records what those packages
  shipped; it does not assess whether what they shipped is safe.
- The absence of reference data for a package or release. Gaps are expected and
  documented in the README, and a consumer is expected to report them rather than
  treat them as clean.
- Availability of the upstream sources this database is built from.

## Verifying this database

`manifest.json` carries the SHA-256 of every package CSV, and is signed with
Ed25519 in [minisign](https://jedisct1.github.io/minisign/) format. One signature
therefore covers the whole database: verify the manifest, then check any CSV
against it.

| | |
|---|---|
| Signature | `manifest.json.minisig` |
| Public key | `minisign.pub` |
| Key ID | `B72F78714F9831AA` |
| Algorithm | Ed25519, prehashed with BLAKE2b-512 |

Install minisign — a single small binary, packaged nearly everywhere:

```bash
brew install minisign          # macOS
apt install minisign           # Debian, Ubuntu
dnf install minisign           # Fedora, RHEL
apk add minisign               # Alpine
# or a static build from https://jedisct1.github.io/minisign/
```

Then, from a clone or download of this repository:

```bash
minisign -Vm manifest.json -p minisign.pub
# Signature and comment signature verified
# Trusted comment: timestamp:... file:manifest.json hashed
```

It exits `0` when the signature is good and `1` when it is not:

```bash
minisign -Vm manifest.json -p minisign.pub || {
    echo 'integrity database failed verification — do not use it' >&2
    exit 1
}
```

Both output lines matter. "Signature ... verified" covers the manifest;
"comment signature verified" covers the trusted comment, which is what stops it
being lifted from another file's signature.

### Verifying without minisign

Ed25519 has been in PHP core via `ext-sodium` since 7.2, so a consumer can verify
with no external binary and no network — which matters if you are checking an
installation that will not boot:

```php
$sig = file_get_contents('manifest.json.minisig');
$key = file_get_contents('minisign.pub');

// public key file:  comment line, then base64( "Ed" | key_id[8] | public_key[32] )
// signature file:   comment line, then base64( "ED" | key_id[8] | signature[64] ),
//                   then a trusted comment line, then base64( global_signature[64] )
$publicKey = substr(base64_decode(explode("\n", trim($key))[1]), 10);
$signature = substr(base64_decode(explode("\n", trim($sig))[1]), 10);

$verified = sodium_crypto_sign_verify_detached(
    $signature,
    sodium_crypto_generichash(file_get_contents('manifest.json'), '', 64),
    $publicKey,
);
```

`ED` means prehashed: the signature covers a BLAKE2b-512 digest of the file rather
than its bytes, so use `sodium_crypto_generichash` with a 64-byte output.

Then check any CSV against the verified manifest:

```bash
grep '"db/packages/magento/module-backend/102.0.7.csv"' manifest.json
shasum -a 256 db/packages/magento/module-backend/102.0.7.csv
```

### If you are writing a consumer

Three things are worth doing that a minimal implementation skips:

- **Compare the key ID**, bytes 2 to 10 of both files, not just whether the
  signature verifies. A signature made by a different key is not a lesser failure
  than a bad one — both mean the data was not signed by the key you trust — but
  reporting it as "invalid signature" sends an operator hunting for corruption when
  the answer is a different signer.
- **Verify the global signature** on line 4, over `signature | trusted_comment`,
  against the same public key.
- **Pin the public key.** Do not fetch `minisign.pub` from the same place as the
  data. Anyone able to tamper with one can tamper with the other, and you would
  cheerfully verify a forgery against its own key. Embed it, and compare any fetched
  copy against the embedded one.

Treat both a failed signature and a missing one as hard failures. Refuse to use the
data either way, but report them differently: a missing signature usually means a
stale cache that needs refreshing, while a failed one means the data should not be
trusted.

## What signing does and does not cover

It proves the manifest was produced by whoever holds the secret key. That covers
tampering with the published data: a stolen push token, a compromised repository, an
intercepted download.

It does **not** cover a compromised builder. The key lives in the build workflow's
secrets, so anyone able to change that workflow can sign whatever they like.
Closing that would need an offline key or a workflow-bound keyless scheme, and both
cost the ability to publish unattended.

This is stated plainly because a signature invites more trust than it always earns.
A verified manifest means the data is what the builder published. It does not mean
the builder was honest.

## The signing key

The key is held offline and in the builder's CI secrets. There is no third copy.

If it were ever lost or exposed, recovery means a new keypair, a republished
`minisign.pub`, and a new key ID. **A change of key ID is an event, not a routine
update.** If you see one, and it was not announced here, treat the database as
untrusted and ask before using it.

## Supported versions

This is a continuously published data set rather than released software. Only the
current state of the `main` branch is supported; older commits are history, not
maintained releases.

Reference data is added as Magento publishes releases. Coverage gaps are documented
in the [README](README.md) and are expected — a consumer should report them rather
than treat an unknown package as clean.
