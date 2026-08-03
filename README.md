# pnpm version pinning breaks behind `packagefeedproxy.microsoft.io`

Minimal reproduction of a bug in the npm proxy at `https://packagefeedproxy.microsoft.io/npm/`.

**Goal this blocks:** pinning an exact pnpm version in `package.json` (`packageManager`) so every developer and CI job uses the same one, regardless of what they have installed locally. pnpm implements this by downloading and switching to the pinned version automatically. That mechanism does not work behind this proxy.

The proxy serves **authentic, byte-identical package tarballs**, but its packument metadata differs from the public npm registry, and pnpm depends on the parts that are missing. Two distinct problems are involved:

- **[Issue A](#4-issue-a--the-non-canonical-disttarball-url) — `dist.tarball` points at a different origin from the packument**, which pnpm's package-manager resolution check rejects.
- **[Issue B](#5-issue-b--the-stripped-distsignatures-and-distintegrity) — `dist.signatures` and `dist.integrity` are stripped from every version**, so pnpm's signature check cannot pass. This also affects the npm CLI and Corepack.

Each issue is described below along with what a fix would look like from the pnpm side and from the Azure DevOps side. Both are documented neutrally; which side changes is a decision for the respective maintainers.

## Contents

1. [Reproduce](#1-reproduce)
2. [What the proxy returns vs. the public registry](#2-what-the-proxy-returns-vs-the-public-registry)
3. [How pnpm validates a pinned version](#3-how-pnpm-validates-a-pinned-version)
4. [Issue A — the non-canonical `dist.tarball` URL](#4-issue-a--the-non-canonical-disttarball-url)
5. [Issue B — the stripped `dist.signatures` and `dist.integrity`](#5-issue-b--the-stripped-distsignatures-and-distintegrity)
6. [When this started, and the changes responsible](#6-when-this-started-and-the-changes-responsible)
7. [Notes](#7-notes)

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

## 3. How pnpm validates a pinned version

When `packageManager` names a version that differs from the running one, pnpm downloads that version and **executes it as a native binary**. A cloned repository controls its own lockfile and can point at any registry, so pnpm treats the pinned version as untrusted input and verifies it before running it. Two checks apply, in order. Check 1 is analysed as [Issue A](#4-issue-a--the-non-canonical-disttarball-url) and check 2 as [Issue B](#5-issue-b--the-stripped-distsignatures-and-distintegrity).

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

Entries are pinned to specific `ms-feed-N` hosts (not portable, and those hosts vary per request), and integrity falls back to **SHA-1** derived from `shasum` because `dist.integrity` is missing. The more fundamental problem is not that SHA-1 is weaker than SHA-512: it is that `dist.integrity` is the value npm's signature is computed over (`name@version:<integrity>`), so a packument without it makes signature verification impossible by construction. A lockfile produced against the public registry is `{ integrity: sha512-… }` with no URL.

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

## 4. Issue A — the non-canonical `dist.tarball` URL

### 4.1 What fails

`assertIntegrityOnlyResolution()` in [`pnpm/src/packageManagerLockfile.ts`](https://github.com/pnpm/pnpm/blob/main/pnpm11/pnpm/src/packageManagerLockfile.ts) requires the resolution object for a package-manager dependency to have **exactly one key**, `integrity`:

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

The presence of a `tarball` field is fatal on its own. `toLockfileResolution()` omits `tarball` only when `isCanonicalRegistryTarballUrl()` holds — that is, when the URL equals `getNpmTarballUrl(name, version, { registry })`.

This proxy serves packuments from `packagefeedproxy.microsoft.io` and tarballs from `ms-feed-N.pkgs.visualstudio.com`, so that equality can never hold, the URL is always retained in the resolution, and the assertion always fires:

```
[ERROR] The packageManager dependency "pnpm@11.10.0" in pnpm-lock.yaml must use a
registry package path and an integrity-only resolution
```

The `ms-feed-N` host also varies between requests, so the retained URL is not stable across machines or runs.

### 4.2 How this compares to the rest of the ecosystem

Two facts frame the gap:

- **No specification requires a canonical tarball URL.** npm documents `tarball` only as "the url of the tarball containing the payload for this package" ([`package-metadata.md`](https://github.com/npm/registry/blob/main/docs/responses/package-metadata.md)), and [`REGISTRY-API.md`](https://github.com/npm/registry/blob/main/docs/REGISTRY-API.md) says it is "usually in the form of `https://registry.npmjs.org/<name>/-/<name>-<version>.tgz`" — *usually*, not necessarily. Rewriting `dist.tarball` is normal and universal: Verdaccio, JFrog Artifactory and Azure Artifacts all do it. What is unusual here is that the rewritten URL points at a *different origin* from the one that served the packument.
- **pnpm accepts non-derivable tarball URLs elsewhere.** Its ordinary lockfile logic deliberately preserves them, per `@pnpm/lockfile.utils` 1100.0.6: "Restored the heuristic that preserves tarball URLs in `pnpm-lock.yaml` when they cannot be derived from name+version+registry … most notably GitHub Packages … and JSR." The package-manager bootstrap path applies the opposite rule to the same data.

The stated purpose of the stricter rule is that a cloned repository controls its own lockfile, so a stored URL could point anywhere. In the failing case here the URL is not repository-supplied — it is resolved fresh from the trusted bootstrap registry (user-level `~/.npmrc`) and then rejected. The bytes are separately pinned by `integrity` and, since [#12292](https://github.com/pnpm/pnpm/pull/12292), by signature verification, so what the rule adds beyond those is control over which host is contacted.

### 4.3 What a fix would look like

**From the pnpm side.** Accept a stored `tarball` alongside `integrity` for package-manager dependencies, leaving signature verification unchanged. A narrower variant — accept a stored URL when its origin is among the configured/trusted registries — would preserve the fetch-destination intent, but would *not* unblock this proxy, whose tarball origin differs from its packument origin; only accepting a stored URL does.

**From the Azure DevOps side.** Serve `dist.tarball` from the same endpoint that served the packument (`https://packagefeedproxy.microsoft.io/npm/<name>/-/<name>-<version>.tgz`), or redirect from it. This is what Verdaccio does, and it also satisfies other clients that assume a registry-relative layout — Corepack, for example, rewrites tarball URLs by string-replacing the registry prefix. It additionally removes the varying `ms-feed-N` host from generated lockfiles.

Either change alone resolves Issue A.

---

## 5. Issue B — the stripped `dist.signatures` and `dist.integrity`

### 5.1 What fails

Once a resolution is accepted, pnpm verifies that the bytes it is about to execute carry npm's registry signature for that exact `name@version`, and fails closed when no signature is present:

```
[ERROR] Refusing to run pnpm@11.10.0: its npm registry signature could not be verified
(@pnpm/exe@11.10.0: has no registry signature; @pnpm/macos-arm64@11.10.0: has no
registry signature; pnpm@11.10.0: has no registry signature).
```

Behind `packagefeedproxy.microsoft.io` this error is normally masked, because Issue A fires first and the signature check is never reached. The two can be separated by pointing at the underlying feed directly (`https://ms-feed-12.pkgs.visualstudio.com/1es-public/_packaging/npm-public/npm/registry/`), whose `dist.tarball` *is* canonical for itself but which still carries no signatures — check 1 then passes and the error above is what appears.

The metadata is stripped uniformly, not selectively:

```bash
for p in pnpm is-positive typescript react; do
  curl -s "https://packagefeedproxy.microsoft.io/npm/$p" | jq "{pkg:\"$p\",
    total:(.versions|length),
    integrity:([.versions[]|select(.dist.integrity)]|length),
    signatures:([.versions[]|select(.dist.signatures)]|length)}"
done
# pnpm:        total 1277, integrity 0, signatures 0
# is-positive: total    4, integrity 0, signatures 0
# typescript:  total 3775, integrity 0, signatures 0
# react:       total 2882, integrity 0, signatures 0
```

The two fields are not independent. npm computes the signature over the string `name@version:<integrity>`, so `dist.integrity` is the signed payload: without it there is nothing for a signature to be checked against, and restoring signatures alone would not be sufficient.

The effect is not limited to pnpm. `npm audit signatures` reports `found no dependencies to audit that were installed from a supported registry`, and Corepack — bundled with Node.js and the original implementation of `packageManager` — fails on the same feed, though it stops earlier, at `GET /{package}/{version}`, which returns 404 (see 5.3).

### 5.2 Standards status

No standards body has specified the npm registry API. There is no IETF RFC, W3C Recommendation, ECMA standard, or Node.js-official specification for the packument format, `dist.integrity`, or `dist.signatures`. The de facto specification is npm's own documentation together with the behaviour of the live registry.

W3C [Subresource Integrity](https://www.w3.org/TR/SRI/) is a Recommendation, but it governs the `integrity` attribute on HTML `<script>` and `<link>` elements. npm's `dist.integrity` borrows SRI's *string format* (`<algorithm>-<base64>`) and nothing more; the specification does not extend to npm packuments.

`dist.integrity` itself has been documented since April 2017 in [`package-metadata.md`](https://github.com/npm/registry/blob/main/docs/responses/package-metadata.md): "`integrity`: since Apr 2017, string in the format `<hashAlgorithm>-<base64-hash>`". That document describes itself as archived and does not mention ECDSA `dist.signatures`; the signatures convention is documented separately, in 5.3.

### 5.3 npm's convention for third-party registries

From [docs.npmjs.com/about-registry-signatures](https://docs.npmjs.com/about-registry-signatures); the identical text also appears under "Audit Signatures" in the [`npm audit` documentation](https://docs.npmjs.com/cli/v11/commands/npm-audit):

> The npm CLI supports registry signatures and signing keys provided by any registry if the following conventions are followed:
>
> **1. Signatures are provided in the package's `packument` in each published version within the `dist` object** […]
>
> **2. Public signing keys are provided at `registry-host.tld/-/npm/v1/keys`** […]

Measured against this feed:

| Convention | Expected | Actual |
| --- | --- | --- |
| 1. `dist.signatures` in each version's `dist` object | present | **absent on every version of every package checked** |
| 2. Signing keys at `<registry>/-/npm/v1/keys` | served | **HTTP 404** |

Separately, [`REGISTRY-API.md`](https://github.com/npm/registry/blob/main/docs/REGISTRY-API.md) lists `GET /{package}/{version}` as a Package Endpoint; it also returns 404 here, which is what stops Corepack before it reaches any signature check:

```bash
curl -s -o /dev/null -w '%{http_code}\n' https://packagefeedproxy.microsoft.io/npm/-/npm/v1/keys
# => 404
curl -s -o /dev/null -w '%{http_code}\n' https://packagefeedproxy.microsoft.io/npm/pnpm/11.10.0
# => 404
```

### 5.4 What a fix would look like

**From the Azure DevOps side.** For packages whose upstream is npmjs.org, pass the upstream `dist` object through unmodified apart from `tarball`: `dist.signatures` verbatim, and `dist.integrity` verbatim since it is the value the signature is computed over. Optionally serve `GET /{package}/{version}` and `GET /-/npm/v1/keys`, which affect Corepack and `npm audit signatures` respectively rather than pnpm's version switch. Verdaccio is a worked example: it [rewrites `dist.tarball` to its own host while spreading the remaining `dist` fields through unchanged](https://github.com/verdaccio/verdaccio/blob/master/packages/core/tarball/src/convertDistRemoteToLocalTarballUrls.ts), and npm's signature stays valid across that rewrite because it covers `name@version:integrity` and not the URL.

Note that this is passthrough rather than re-signing. A registry that instead minted its own signatures under its own key and served that key at `/-/npm/v1/keys` would satisfy npm's third-party convention, but would not satisfy pnpm's package-manager check, which verifies against npm public keys embedded in the pnpm binary and does not consult a registry's keys endpoint.

**From the pnpm side.** Make the guarantee independent of registry-supplied signatures — for example by honouring an integrity pin supplied in the project (pnpm already accepts inline integrity for `configDependencies`, in the form `1.0.0+sha512-<base64>`), by using the npm provenance attestations pnpm already publishes with, or by verifying against a key the pnpm maintainers control. Any of these would decouple verification from whether a given registry mirrors npm's metadata.

Unlike Issue A, the two sides here are not interchangeable: only the registry-side change makes the current check pass, and only the client-side change removes the requirement that a registry mirror npm's signature metadata.

---

## 6. When this started, and the changes responsible

Version pinning long predates this breakage, so a wide band of pnpm versions self-installs correctly behind the proxy.

Automatic version switching landed in **pnpm 9.7.0** ([#8363](https://github.com/pnpm/pnpm/pull/8363), commit `26b065c`, which added `pnpm/src/switchCliVersion.ts`), opt-in behind `manage-package-manager-versions=true`, and became the default in **10.0.0** (commit `dfcf034`). pnpm 9.6.0 and earlier ignore `packageManager` entirely.

### Bisection

Global pnpm installed via `npm install -g pnpm@<version>`, run against this repo (which pins `pnpm@11.10.0`), macOS arm64, registry `https://packagefeedproxy.microsoft.io/npm/`:

| Global pnpm | Result |
| --- | --- |
| 9.6.0 | no switch — feature absent, prints `9.6.0` |
| 9.7.0 | switches to 11.10.0 |
| 10.0.0 | switches to 11.10.0 |
| 10.34.1 | switches to 11.10.0 |
| **10.34.2** | **fails — Issue B**: `Refusing to run pnpm@11.10.0: its npm registry signature could not be verified` |
| 10.34.5 | fails — same |
| 11.5.2 | switches to 11.10.0 |
| **11.5.3** | **fails — Issue A**: `must use a registry package path and an integrity-only resolution` |
| 11.15.1, 11.16.0 | fails — same |
| 12.0.0-alpha.18 | fails — `Cannot resolve pnpm@11.10.0 as a package manager dependency because it has no integrity` |

The two release lines fail *differently*, which is useful for telling the issues apart:

- **10.34.2+** fails with **Issue B directly**, because the 10.x line has the signature check but not the Issue A assertion.
- **11.5.3+** has both checks, and Issue A is evaluated first, so it **masks Issue B** — the signature error never appears. (Isolating Issue B on the 11.x line therefore requires a registry whose tarball URL *is* canonical, such as the underlying `ms-feed-12` feed; there, 9.7.0 / 10.0.0 / 11.5.2 still pass and 11.5.3 fails with the signature error.)

The 10.x and 11.x lines are **parallel maintenance lines, not points on one timeline** — `v10.34.2` and `v11.5.3` were both released on 2026-06-10. Nothing was added in 10.x and later removed: the signature check shipped to both lines the same day, via [#12300](https://github.com/pnpm/pnpm/pull/12300) (`fix(security): port the latest security fixes to v10`).

That port deliberately took only *part* of [#12296](https://github.com/pnpm/pnpm/pull/12296). It applied the trusted-registry/network-config half, but explicitly skipped the env-lockfile validation that produces the Issue A error:

> The env-lockfile validation parts of the main PR (`packageManagerLockfile.ts`, `syncEnvLockfile`, peer-suffix handling) do not apply: v10 bootstraps through a staged child install, which already runs outside the repository's config context.

This is confirmed by the release trees: `v10.34.2` contains `tools/plugin-commands-self-updater/src/verifyPnpmEngineIdentity.ts` but **no** `packageManagerLockfile.ts`, whereas `v11.5.3` contains both. (The ports are re-implementations rather than cherry-picks, so `git tag --contains 5f2bb9f` does not list any 10.x tag.)

The 12.x alpha surfaces a third variant of the same root cause: the missing `dist.integrity` is now a hard error in its own right ([#12394](https://github.com/pnpm/pnpm/pull/12394)), rather than silently degrading to a SHA-1 derived from `shasum`.

### Existing upstream reports

**Issue B is reported upstream:** [pnpm/pnpm#13147 — *"Registries without signatures cannot be used"*](https://github.com/pnpm/pnpm/issues/13147) (open, `type: bug` / `state: needs design`, filed 2026-07-19). It reproduces the identical error against a signature-less registry, notes the same regression window (`v11.5.3`+ and `v10.34.2`+), and pnpm's triage confirms the affected code path as `verifyPnpmEngineIdentity.ts`, with parity work also required in the Rust `pacquet` implementation.

**Issue A does not appear to be reported upstream.** No issue in `pnpm/pnpm` (open or closed) matches the error text, and neither of the two nearest issues covers it:

- [#13534](https://github.com/pnpm/pnpm/issues/13534) — `pnpm install` v10 discards `dist.tarball` on installation (GitHub Enterprise serving non-canonical tarball paths). Same underlying theme, but the ordinary install path, not the package-manager bootstrap.
- [#13558](https://github.com/pnpm/pnpm/issues/13558) — wrong lockfile tarball registry for same-host multi-path registries (`ERR_PNPM_TARBALL_URL_MISMATCH`). Again the ordinary install path.
- [#13263](https://github.com/pnpm/pnpm/issues/13263) — `packageManager` resolution ignores the project `.npmrc` registry. This is the other half of [#12296](https://github.com/pnpm/pnpm/pull/12296) (bootstrap reads only trusted config), not the resolution-shape assertion.

It has, however, been hit and worked around in the wild — the error text appears in several unrelated repositories, always in connection with a corporate mirror that rewrites the tarball host. For example [hakula139/nixos-config#120](https://github.com/hakula139/nixos-config/pull/120) diagnoses it precisely:

> pnpm 11 hardened its package-manager self-install: when a repo pins `packageManager` […] pnpm fetches it as `@pnpm/exe` and asserts an **integrity-only** resolution. The artifactory npm mirror rewrites the tarball host to its own domain, so the resolution carries a `tarball` field alongside `integrity` and the assertion throws.

Their workaround was to scope only the `@pnpm` packages to `registry.npmjs.org` while leaving everything else on the mirror. Others resorted to `pmOnFail: ignore` or to pinning a public registry in the project `.npmrc`.

### The two PRs responsible

Both were merged on 2026-06-09 and first shipped in **11.5.3**:

| Issue | PR | Merge commit | What it added |
| --- | --- | --- | --- |
| A | [#12296](https://github.com/pnpm/pnpm/pull/12296) — `fix: harden package-manager bootstrap metadata` | [`822beb5`](https://github.com/pnpm/pnpm/commit/822beb5fa0458a041f2833d452f8dc6b59b1f1cd) | `pnpm/src/packageManagerLockfile.ts` (the integrity-only resolution assertion) and `pnpm/src/packageManagerRegistries.ts`, which restricts bootstrap resolution to trusted (non-project) registry and network config |
| B | [#12292](https://github.com/pnpm/pnpm/pull/12292) — `fix(security): verify npm registry signature before spawning a package-manager binary` | [`5f2bb9f`](https://github.com/pnpm/pnpm/commit/5f2bb9f5ba01d498e03eb54a0d72d185fe3d0aca) | `engine/pm/commands/src/self-updater/verifyPnpmEngineIdentity.ts`, npm's public keys embedded in `deps/security/signatures/src/npmSigningKeys.ts`, and the call site in `installPnpm.ts` |

#12292 addresses a case where a cloned repository controls both the lockfile and the registry the bytes are fetched from, so lockfile integrity alone attests nothing; verification is therefore done against keys the client already holds, and fails closed. #12296 hardens the metadata that feeds that step and prevents a project-level `.npmrc` from steering bootstrap traffic.

---

## 7. Notes

- Reproduced with pnpm 11.15.1 (global) against a repo pinning pnpm 11.10.0, on macOS arm64.
- If the global pnpm already *is* the pinned version, no switch occurs and no error appears — a different pnpm ≥ 11.5.3 is required to reproduce.
- The missing signatures also disable `npm audit signatures` for any package fetched through the proxy.
- **The signature check only runs on a real download.** `installPnpmToStore` returns early if the engine is already in the global virtual store at `~/Library/pnpm/store/v11/links/@/pnpm/<version>/<hash>/`. Clearing `~/Library/pnpm/.tools`, `~/Library/pnpm/package-manager-store` and `~/Library/Caches/pnpm` is *not* sufficient — that store entry must be removed too, or the switch silently succeeds from cache. The hash includes the registry, so changing registries also produces a cache miss.
- **The bootstrap cannot be redirected from the command line.** Neither `--registry=…` nor `npm_config_registry=…` changed which registry the pinned pnpm was resolved from (tested on 11.5.3 and 11.15.1 with all caches cleared); a repo-local `.npmrc` is ignored as well. Only the user-level `~/.npmrc` steers it, by design ([#12296](https://github.com/pnpm/pnpm/pull/12296)).
- Measurements of the proxy in this document were taken on 2026-07-30 against `https://packagefeedproxy.microsoft.io/npm/` and, where noted, the underlying feed at `https://ms-feed-12.pkgs.visualstudio.com/1es-public/_packaging/npm-public/npm/registry/`.
- A Developer Community thread on Azure Artifacts defaulting to SHA-1 is reported at `developercommunity.visualstudio.com/t/npm-packages-from-azure-artifact-defaulting-to-sha/957522`. That host was unreachable from the test environment, so it is **unverified** and is not relied on anywhere above.
