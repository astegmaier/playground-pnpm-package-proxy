# pnpm version pinning breaks behind `packagefeedproxy.microsoft.io`

Minimal reproduction of a bug in the npm proxy at `https://packagefeedproxy.microsoft.io/npm/`.

**Goal this blocks:** we want to pin an exact pnpm version in `package.json` (`packageManager`) so every developer and CI job uses the same one, regardless of what they have installed locally. pnpm implements this by downloading and switching to the pinned version automatically. That mechanism does not work behind this proxy.

The proxy serves **authentic, byte-identical package tarballs**, but its packument metadata differs from the public npm registry in two ways, and pnpm depends on both.

---

## 1. Reproduce

Requires a machine using the proxy as its npm registry.

```bash
# 1. Point npm at the proxy (user-level config)
echo 'registry=https://packagefeedproxy.microsoft.io/npm/' >> ~/.npmrc

# 2. Install ANY pnpm 11.x globally EXCEPT the version this repo pins (11.10.0).
#    A different version is required — pnpm only version-switches on a mismatch.
npm install -g pnpm@11.17.0

# 3. Reproduce
git clone <this repo>
cd playground-pnpm-package-proxy
pnpm install
```

Actual result:

```
[ERROR] The packageManager dependency "pnpm@11.10.0" in pnpm-lock.yaml must use a
registry package path and an integrity-only resolution
```

Expected result: pnpm downloads pnpm 11.10.0, verifies it, switches to it, and installs.

The same command succeeds against `https://registry.npmjs.org`.

---

## 2. What the proxy returns vs. the public registry

Compare the metadata pnpm reads, `versions["<version>"].dist`:

```bash
# Proxy
curl -s https://packagefeedproxy.microsoft.io/npm/pnpm | jq '.versions["11.10.0"].dist'
```

```json
{
  "shasum": "43767258be015d25fe0f81e5c07919c002914d90",
  "tarball": "https://ms-feed-12.pkgs.visualstudio.com/…/pnpm/-/pnpm-11.10.0.tgz"
}
```

```bash
# Public npm registry
curl -s https://registry.npmjs.org/pnpm | jq '.versions["11.10.0"].dist'
```

```json
{
  "shasum": "43767258be015d25fe0f81e5c07919c002914d90",
  "tarball": "https://registry.npmjs.org/pnpm/-/pnpm-11.10.0.tgz",
  "fileCount": 450,
  "integrity": "sha512-C3+LmAYAMZBMAX46QesYehbUDuuCm5XE+MsDaBdh/Eq1PdIZEVubRH9NzhoFohR2RGHn03AzkqnzL5URzoyGyA==",
  "signatures": [
    {
      "sig": "MEYCIQC28ef+LZSlyWSRFLk44cwrmdjEMJ7VNytbZfnsN7wSjQIhAPhNkyQC2olHzK5+OfnZVW17tBGT/OXwUD2NwReDg7zi",
      "keyid": "SHA256:DhQ8wR5APBvFHLF/+Tc+AYvPOdTpcIDqOhxsBHRwC7U"
    }
  ],
  "attestations": { "…": "…" },
  "unpackedSize": 18806563
}
```

Two differences:

**(a) `dist.tarball` is not canonical.** The proxy serves packuments from `packagefeedproxy.microsoft.io` but tarballs from a *different* host (`ms-feed-N.pkgs.visualstudio.com`, where `N` varies between requests). The URL is therefore not derivable from the registry URL + package name + version.

**(b) `dist.signatures` and `dist.integrity` are absent** — on every version, not just recent ones:

```bash
curl -s https://packagefeedproxy.microsoft.io/npm/pnpm \
  | jq '{signed: ([.versions[]|select(.dist.signatures)]|length),
         withIntegrity: ([.versions[]|select(.dist.integrity)]|length),
         total: (.versions|length)}'
# proxy => { "signed": 0,    "withIntegrity": 0,    "total": 1277 }
# npm   => { "signed": 1277, "withIntegrity": 1277, "total": 1277 }
```

(Counts as of 2026-07-29; totals grow as pnpm publishes. The point is *none* vs *all* — the oldest releases are stripped too, so this is not a quarantine or freshness effect.)

The tarball bytes themselves are correct — they hash to exactly what npm signed:

```bash
curl -sL https://packagefeedproxy.microsoft.io/npm/pnpm/-/pnpm-11.10.0.tgz -o pnpm.tgz
openssl dgst -sha512 -binary pnpm.tgz | base64
# => C3+LmAYAMZBMAX46QesYehbUDuuCm5XE+MsDaBdh/Eq1PdIZEVubRH9NzhoFohR2RGHn03AzkqnzL5URzoyGyA==
#    ...identical to npm's signed dist.integrity above.
```

So only the metadata is lost, not the content.

---

## 3. Why pnpm needs these fields

When `packageManager` names a version that differs from the running one, pnpm downloads that version and **executes it as a native binary**. A cloned repository controls its own lockfile and can point at any registry, so pnpm treats the pinned version as untrusted input and verifies it before running it. Two checks apply, in order.

### Check 1 — the resolution must be reconstructible (this is what fails here)

pnpm records a registry dependency as a bare `{ integrity }` entry only when the tarball URL can be rebuilt from registry + name + version. Any other URL has to be stored verbatim, or the package could not be re-fetched later. For the package manager itself pnpm requires the *integrity-only* form, so a stored URL is rejected — that is the error above.

Because the proxy's tarball host differs from its registry host, its URLs are never reconstructible.

This also corrupts ordinary lockfiles. `pnpm-lock.yaml` in this repo was generated behind the proxy and shows both defects:

```yaml
'@typescript/typescript-darwin-arm64@7.0.2':
  resolution:
    integrity: sha1-pV/fz6WN9Y0n2yI3zealweNacjU=   # SHA-1, not SHA-512
    tarball: https://ms-feed-2.pkgs.visualstudio.com/…   # host baked in
```

Entries are pinned to specific `ms-feed-N` hosts (not portable, and those hosts vary per request), and integrity falls back to **SHA-1** derived from `shasum` because `dist.integrity` is missing. SHA-1 is collision-broken and unsuitable as an integrity anchor. A lockfile produced against the public registry is `{ integrity: sha512-… }` with no URL.

### Check 2 — the signature must verify

If the resolution check passes, pnpm verifies that the exact bytes it is about to execute carry npm's registry signature for that `name@version`. npm signs the string `<name>@<version>:<integrity>` with an ECDSA P-256 key; pnpm checks it against npm's public keys, which are **embedded in pnpm** rather than fetched — so a registry cannot present its own key pair.

With `dist.signatures` missing, this fails closed:

```
[ERROR] Refusing to run pnpm@<version>: its npm registry signature could not be verified
(pnpm@<version> has no registry signature; …)
```

Failing closed is deliberate: to the verifier, "no signature" is indistinguishable from "unsigned bytes supplied by an attacker". The `shasum` the proxy does provide is not a substitute — it proves the bytes match what *the proxy* says they should be, not that npm ever published them.

A repository whose lockfile was generated against the public registry passes check 1 and then fails on check 2, so **both** fields are required for version pinning to work.

---

## What would fix this

1. Serve `dist.tarball` on the same host and path shape as the registry (`https://packagefeedproxy.microsoft.io/npm/<name>/-/<name>-<version>.tgz`), or redirect from it.
2. Pass `dist.signatures` through verbatim from upstream npm.
3. Pass `dist.integrity` through as well, so lockfiles do not silently downgrade to SHA-1.

Items 1 and 2 are independent — both are needed.

---

## Notes

- Reproduced with pnpm 11.17.0 (global) against a repo pinning pnpm 11.10.0, on macOS arm64.
- If your global pnpm already *is* the pinned version, no switch occurs and no error appears — install a different 11.x to reproduce.
- The missing signatures also disable `npm audit signatures` for any package fetched through the proxy.
