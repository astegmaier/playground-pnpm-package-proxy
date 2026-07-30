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
npm install -g pnpm@11.15.1

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
3. Pass `dist.integrity` through as well, so lockfiles do not silently downgrade to SHA-1 — and because pnpm 12.x rejects a package manager with no integrity outright.

Items 1 and 2 are independent — both are needed. See [The upstream PRs behind this](#the-upstream-prs-behind-this) for which change enforces which.

---

## Which pnpm versions are affected

Version pinning has existed for far longer than this breakage. The two checks above are recent, so there is a wide band of pnpm versions that self-install correctly behind the proxy.

### When the feature appeared

Automatic version switching landed in **pnpm 9.7.0** ([#8363](https://github.com/pnpm/pnpm/pull/8363), commit `26b065c`, which added `pnpm/src/switchCliVersion.ts`). It was opt-in there, behind `manage-package-manager-versions=true`, and became the default in **10.0.0** (commit `dfcf034`, `feat!: set manage-package-manager-versions to true`). pnpm 9.6.0 and earlier ignore `packageManager` entirely.

### When it started failing

Both checks were added on the same day and shipped in the same release, **11.5.3**:

| Check | Commit / PR | First release |
| --- | --- | --- |
| Registry package path + integrity-only resolution | `822beb5` / [#12296](https://github.com/pnpm/pnpm/pull/12296) | 11.5.3 |
| npm registry signature verification | `5f2bb9f` / [#12292](https://github.com/pnpm/pnpm/pull/12292) | 11.5.3 |

Both PRs are analysed in [The upstream PRs behind this](#the-upstream-prs-behind-this) below.

So **9.7.0 through 11.5.2 self-install fine behind the proxy; 11.5.3 and later do not.**

### Measured results

Global pnpm installed via `npm install -g pnpm@<version>`, run against this repo (which pins `pnpm@11.10.0`), macOS arm64, registry `https://packagefeedproxy.microsoft.io/npm/`:

| Global pnpm | Result |
| --- | --- |
| 9.6.0 | no switch — feature absent, prints `9.6.0` |
| 9.7.0 | switches to 11.10.0 |
| 10.0.0 | switches to 11.10.0 |
| 11.5.2 | switches to 11.10.0 |
| 11.5.3 | fails — `must use a registry package path and an integrity-only resolution` |
| 11.15.1, 11.16.0 | fails — same |
| 12.0.0-alpha.18 | fails — `Cannot resolve pnpm@11.10.0 as a package manager dependency because it has no integrity` |

The 12.x alpha surfaces a third variant of the same root cause: the proxy's missing `dist.integrity` is now itself a hard error, rather than silently degrading to SHA-1.

### Isolating check 2

Behind `packagefeedproxy.microsoft.io` only **check 1** ever fires, because the tarball host never matches the registry host — so the signature check is unreachable there and the two defects cannot be told apart from the error message alone.

They can be separated using the underlying feed directly, `https://ms-feed-12.pkgs.visualstudio.com/1es-public/_packaging/npm-public/npm/registry/`. That registry serves a `dist.tarball` that *is* canonical for itself, but still carries no `dist.signatures`. Check 1 then passes and check 2 fires:

```
[ERROR] Refusing to run pnpm@11.10.0: its npm registry signature could not be verified
(@pnpm/exe@11.10.0: @pnpm/exe@11.10.0 has no registry signature;
 @pnpm/macos-arm64@11.10.0: @pnpm/macos-arm64@11.10.0 has no registry signature;
 pnpm@11.10.0: pnpm@11.10.0 has no registry signature).
The bytes selected by this project's lockfile/registry do not match a published,
signed pnpm release.
```

On that same registry, 9.7.0 / 10.0.0 / 11.5.2 all self-install successfully — confirming both checks are new in 11.5.3, and that fixing only the tarball URL would leave pinning broken.

### Two traps when reproducing this

- **The signature check only runs on a real download.** `installPnpmToStore` returns early if the engine is already in the global virtual store, at `~/Library/pnpm/store/v11/links/@/pnpm/<version>/<hash>/`. Clearing `~/Library/pnpm/.tools`, `~/Library/pnpm/package-manager-store` and `~/Library/Caches/pnpm` is *not* sufficient — that store entry must be removed too, or the switch silently succeeds from cache. The hash includes the registry, so changing registries also produces a cache miss.
- **You cannot redirect the bootstrap from the command line.** Neither `--registry=…` nor `npm_config_registry=…` changed which registry the pinned pnpm was resolved from (tested on 11.5.3 and 11.15.1 with all caches cleared); a repo-local `.npmrc` is ignored as well. Only the user-level `~/.npmrc` steers it. That is deliberate — see [#12296](https://github.com/pnpm/pnpm/pull/12296) below — but it means the usual "just point at npmjs.org for this one command" workaround is unavailable.

---

## The upstream PRs behind this

Both checks come from two pull requests by pnpm's maintainer `zkochan`, merged less than an hour apart on 2026-06-09 and released together in 11.5.3. Neither targets proxies; both are hardening the same threat model, and this proxy happens to be indistinguishable from the attacker they describe.

### [#12292](https://github.com/pnpm/pnpm/pull/12292) — `fix(security): verify npm registry signature before spawning a package-manager binary`

Merge commit [`5f2bb9f`](https://github.com/pnpm/pnpm/commit/5f2bb9f5ba01d498e03eb54a0d72d185fe3d0aca), 21 files changed. This is **check 2**.

The PR's stated problem is that pnpm can be made to download and execute a native binary through two repository-controlled inputs — the pacquet install engine declared in `configDependencies`, and the `packageManager` / `devEngines.packageManager` version switch, which is on by default. As the PR puts it, the repository also controls the lockfile integrity and the registry the bytes come from, so *"matching the lockfile integrity proves nothing — it matches the hash the attacker wrote."*

What it added:

- `deps/security/signatures/src/verifySignatures.ts` — `verifyInstalledPackageSignatures()`, which validates `name@version:integrity` against `dist.signatures`.
- `deps/security/signatures/src/npmSigningKeys.ts` — npm's public signing keys, **embedded in the CLI** rather than fetched, explicitly modelled on corepack. The PR states there is intentionally *"no runtime override or off-switch"* for them, so a registry the user did not vouch for cannot supply its own keypair.
- `engine/pm/commands/src/self-updater/verifyPnpmEngineIdentity.ts` — the pnpm-engine wrapper that emits the `Refusing to run pnpm@…` error. It verifies `pnpm`, `@pnpm/exe`, and the host platform binary.
- `installing/commands/src/verifyPacquetIdentity.ts` — the same treatment for the pacquet engine.
- A `deps/security/signatures/scripts/update-npm-signing-keys.mjs` script wired into the `create-release-pr` workflow, so an npm key rotation blocks the release rather than silently breaking verification.

Two design decisions in this PR are what make it bite here:

1. **It fails closed.** Any outcome other than a valid signature — invalid, absent, or registry unreachable — refuses the switch. "No signature" and "unsigned bytes from an attacker" are the same observation. The proxy's packuments have no `dist.signatures` at all, so they land in exactly that bucket.
2. **It runs only on a store cache miss.** The PR notes this is a deliberate performance trade-off, *"so it adds no network round trip to every command."* That is also why this failure is intermittent-looking in practice: once any pnpm version has populated the global virtual store for a given registry, later invocations skip verification entirely.

The PR explicitly claims mirrors are fine — *"an **npm mirror works transparently** — it proxies the same signed packument, with no configuration."* That assumption is precisely what `packagefeedproxy.microsoft.io` violates: it proxies the tarballs faithfully but drops the signatures from the packument.

### [#12296](https://github.com/pnpm/pnpm/pull/12296) — `fix: harden package-manager bootstrap metadata`

Merge commit [`822beb5`](https://github.com/pnpm/pnpm/commit/822beb5fa0458a041f2833d452f8dc6b59b1f1cd), 12 files changed. This is **check 1**, and it is the error this repo actually reproduces.

Its summary describes two related changes:

- Resolve package-manager bootstrap metadata through **trusted user/CLI registries and trusted network config**, defaulting to the public npm registry instead of project/workspace registry settings, *"so repository `.npmrc` proxy/TLS/configByUri values cannot steer package-manager bootstrap traffic."* This is `pnpm/src/packageManagerRegistries.ts`, whose `DEFAULT_PACKAGE_MANAGER_REGISTRY` is hard-coded to `https://registry.npmjs.org/`.
- Validate the env-lockfile entries before installing or executing: *"dependency paths must be registry package paths and package records must use integrity-only resolutions."* This is `pnpm/src/packageManagerLockfile.ts`.

The validation is strict in a way that matters here. `assertIntegrityOnlyResolution()` requires the resolution object to have **exactly one key**, `integrity`:

```ts
const resolutionKeys = Object.keys(resolution)
if (
  resolutionKeys.length !== 1 ||
  resolutionKeys[0] !== 'integrity' ||
  ...
) {
  throw invalidPackageManagerLockfile(depPath)
}
```

A `tarball` field is not merely distrusted — its presence alone is fatal. And `toLockfileResolution()` only omits `tarball` when `isCanonicalRegistryTarballUrl()` holds, i.e. when the URL equals `getNpmTarballUrl(name, version, { registry })`. Because the proxy serves packuments from `packagefeedproxy.microsoft.io` and tarballs from `ms-feed-N.pkgs.visualstudio.com`, that equality can never hold, so the URL is always retained and the assertion always fires.

The first bullet is also why the ordinary escape hatches are gone. The bootstrap deliberately reads only non-project config, and in practice (see the traps above) not even `--registry` or `npm_config_registry` redirected it in our testing — leaving the user-level `~/.npmrc` as the only lever.

### [#12394](https://github.com/pnpm/pnpm/pull/12394) — `fix: sync pacquet lockfile output with pnpm`

Merge commit [`baf1502`](https://github.com/pnpm/pnpm/commit/baf15021ec134d55a604b1af42552d670957e4fa), merged 2026-06-14. Not one of the two checks, but it is where the third, distinct 12.x error comes from. The Rust env-installer in `pacquet/crates/env-installer/src/resolve_package_manager_integrities.rs` now rejects a package-manager dependency that has no integrity outright:

```
Cannot resolve pnpm@11.10.0 as a package manager dependency because it has no integrity
```

Under the old behaviour the missing `dist.integrity` silently degraded to a SHA-1 derived from `shasum`. In 12.x that degradation is gone, so the proxy's third defect — stripped `dist.integrity` — becomes a hard failure in its own right, independently of the tarball URL and the signatures.

### What this means for the fix list

The two PRs are independent controls over independent properties, which is why fixing one of them is not enough:

| Proxy defect | Trips | Introduced by |
| --- | --- | --- |
| `dist.tarball` on a different host | check 1 | [#12296](https://github.com/pnpm/pnpm/pull/12296) |
| `dist.signatures` stripped | check 2 | [#12292](https://github.com/pnpm/pnpm/pull/12292) |
| `dist.integrity` stripped | 12.x hard error (SHA-1 downgrade before that) | [#12394](https://github.com/pnpm/pnpm/pull/12394) |

Restoring only the canonical tarball URL moves the failure from check 1 to check 2, as demonstrated above with the direct `ms-feed-12` feed. All three fields have to be passed through.

---

## Notes

- Reproduced with pnpm 11.15.1 (global) against a repo pinning pnpm 11.10.0, on macOS arm64.
- If your global pnpm already *is* the pinned version, no switch occurs and no error appears — install a different pnpm ≥ 11.5.3 to reproduce.
- The missing signatures also disable `npm audit signatures` for any package fetched through the proxy.
