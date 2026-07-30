# pnpm version pinning breaks behind `packagefeedproxy.microsoft.io`

Minimal reproduction of a bug in the npm proxy at `https://packagefeedproxy.microsoft.io/npm/`.

**Goal this blocks:** pinning an exact pnpm version in `package.json` (`packageManager`) so every developer and CI job uses the same one, regardless of what they have installed locally. pnpm implements this by downloading and switching to the pinned version automatically. That mechanism does not work behind this proxy.

The proxy serves **authentic, byte-identical package tarballs**, but its packument metadata differs from the public npm registry, and pnpm depends on the parts that are missing. There are two distinct problems, and they do not have the same answer:

- **[Issue A](#4-issue-a--the-non-canonical-disttarball-url) — `dist.tarball` points at a different origin from the packument.** No specification requires otherwise, and pnpm accepts such URLs elsewhere. The more appropriate fix is for **pnpm** to relax this check; a change on the feed side would also resolve it.
- **[Issue B](#5-issue-b--the-stripped-distsignatures-and-distintegrity) — `dist.signatures` and `dist.integrity` are stripped from every version.** This departs from a published npm convention for third-party registries and breaks the npm CLI and Corepack as well as pnpm. The fix belongs with **Azure DevOps**.

[Section 6](#6-summary-where-each-fix-belongs) summarises both.

## Contents

1. [Reproduce](#1-reproduce)
2. [What the proxy returns vs. the public registry](#2-what-the-proxy-returns-vs-the-public-registry)
3. [How pnpm validates a pinned version](#3-how-pnpm-validates-a-pinned-version)
4. [Issue A — the non-canonical `dist.tarball` URL](#4-issue-a--the-non-canonical-disttarball-url)
5. [Issue B — the stripped `dist.signatures` and `dist.integrity`](#5-issue-b--the-stripped-distsignatures-and-distintegrity)
6. [Summary: where each fix belongs](#6-summary-where-each-fix-belongs)
7. [Which pnpm versions are affected](#7-which-pnpm-versions-are-affected)
8. [The upstream PRs behind this](#8-the-upstream-prs-behind-this)
9. [Notes](#9-notes)

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

This is the failure the repository reproduces today, and it is the one where the case for changing pnpm is stronger than the case for changing the feed.

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

The presence of a `tarball` field is fatal on its own. `toLockfileResolution()` omits `tarball` only when `isCanonicalRegistryTarballUrl()` holds — that is, when the URL equals `getNpmTarballUrl(name, version, { registry })`. Because this proxy serves packuments from `packagefeedproxy.microsoft.io` and tarballs from `ms-feed-N.pkgs.visualstudio.com`, that equality can never hold, the URL is always retained, and the assertion always fires.

### 4.2 No standard requires a canonical tarball URL

npm's own documentation describes `tarball` only as "the url of the tarball containing the payload for this package" ([`package-metadata.md`](https://github.com/npm/registry/blob/main/docs/responses/package-metadata.md)); [`REGISTRY-API.md`](https://github.com/npm/registry/blob/main/docs/REGISTRY-API.md) says it is "usually in the form of `https://registry.npmjs.org/<name>/-/<name>-<version>.tgz`" — *usually*, not necessarily. No specification requires the tarball to be reachable at a path derivable from the registry URL, and none requires it to be on the registry's own host.

Rewriting `dist.tarball` is normal and universal: Verdaccio, JFrog Artifactory and Azure Artifacts all do it. What is unusual about this proxy is only that the rewritten URL points at a *different origin* from the one that served the packument, and that the `ms-feed-N` host varies between requests.

### 4.3 pnpm already accepts non-canonical tarball URLs elsewhere

pnpm's ordinary lockfile logic deliberately preserves tarball URLs that cannot be reconstructed, precisely because legitimate registries serve them. From `@pnpm/lockfile.utils` 1100.0.6:

> Restored the heuristic that preserves tarball URLs in `pnpm-lock.yaml` when they cannot be derived from name+version+registry, even with the default `lockfileIncludeTarballUrl: false`. Without this, `pnpm install --frozen-lockfile` from an empty store fails with `ERR_PNPM_FETCH_404` for packages on registries that serve tarballs from a non-standard path — most notably GitHub Packages (`https://npm.pkg.github.com/download/<scope>/<name>/<version>/<hash>`) and JSR.

So pnpm already recognises that a non-derivable tarball URL is a normal registry behaviour rather than evidence of tampering. The package-manager bootstrap path applies the opposite rule to the same data.

### 4.4 What the strict rule does and does not buy

The stated purpose of the check is that a cloned repository controls its own lockfile, so a stored URL could point anywhere. That concern is real, but it is already addressed by the checks that follow:

- The bytes are pinned by `integrity`, and — since [#12292](https://github.com/pnpm/pnpm/pull/12292) — must additionally carry npm's ECDSA signature over `name@version:<integrity>`, verified against keys embedded in pnpm. A substituted URL cannot yield bytes that satisfy both.
- In the failing case here, the URL is not repository-supplied at all. It is resolved fresh from the *trusted* bootstrap registry (user/global `.npmrc`), then rejected — even though pnpm itself just fetched it from the registry it was told to trust.

What the rule adds beyond that is control over *which host is contacted*, which is a fetch-destination concern rather than a content-integrity one.

### 4.5 Where the fix belongs

**Primarily with pnpm.** A minimal relaxation would be to accept a stored `tarball` alongside `integrity` for package-manager dependencies, keeping signature verification unchanged. If restricting the fetch destination is considered essential, a narrower rule — accept a stored URL when it was resolved from a trusted bootstrap registry, or when its origin is among the configured registries — would preserve the intent without rejecting conformant registries. Note that an origin-matching rule alone would *not* unblock this proxy, since its tarball origin differs from its packument origin; only accepting a stored URL does.

**Secondarily with Azure DevOps.** Serving `dist.tarball` from the same endpoint that served the packument (`https://packagefeedproxy.microsoft.io/npm/<name>/-/<name>-<version>.tgz`), or redirecting from it, would resolve this for pnpm and for any other client that assumes a registry-relative layout — Corepack, for instance, rewrites tarball URLs by string-replacing the registry prefix. This is what Verdaccio does, and it is good hygiene regardless of Issue B, because the current URLs pin lockfiles to a specific `ms-feed-N` host that varies between requests and is therefore not portable between machines.

Because Issue A is a stricter-than-ecosystem rule rather than a conformance failure, it is reasonable for pnpm to relax it independently of whether the feed changes.

---

## 5. Issue B — the stripped `dist.signatures` and `dist.integrity`

This issue is the reverse: pnpm's behaviour follows a published npm convention, and the feed does not.

### 5.1 What fails

Once a resolution is accepted, pnpm verifies that the bytes it is about to execute carry npm's registry signature for that exact `name@version`, and fails closed when no signature is present:

```
[ERROR] Refusing to run pnpm@11.10.0: its npm registry signature could not be verified
(@pnpm/exe@11.10.0: has no registry signature; @pnpm/macos-arm64@11.10.0: has no
registry signature; pnpm@11.10.0: has no registry signature).
```

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

### 5.2 Standards status

No standards body has specified the npm registry API. There is no IETF RFC, W3C Recommendation, ECMA standard, or Node.js-official specification for the packument format, `dist.integrity`, or `dist.signatures`. The de facto specification is npm's own documentation together with the behaviour of the live registry.

W3C [Subresource Integrity](https://www.w3.org/TR/SRI/) is a Recommendation, but it governs the `integrity` attribute on HTML `<script>` and `<link>` elements. npm's `dist.integrity` borrows SRI's *string format* (`<algorithm>-<base64>`) and nothing more; the specification does not extend to npm packuments.

What does exist is an explicit, currently maintained npm convention addressed specifically to third-party registries.

### 5.3 npm's published convention for third-party registries

From [docs.npmjs.com/about-registry-signatures](https://docs.npmjs.com/about-registry-signatures); the identical text also appears under "Audit Signatures" in the [`npm audit` documentation](https://docs.npmjs.com/cli/v11/commands/npm-audit):

> The npm CLI supports registry signatures and signing keys provided by any registry if the following conventions are followed:
>
> **1. Signatures are provided in the package's `packument` in each published version within the `dist` object** […]
>
> **2. Public signing keys are provided at `registry-host.tld/-/npm/v1/keys`** […]

Both halves are unmet:

| Convention | Expected | Actual |
| --- | --- | --- |
| 1. `dist.signatures` in each version's `dist` object | present | **absent on every version of every package checked** |
| 2. Signing keys at `<registry>/-/npm/v1/keys` | served | **HTTP 404** |

```bash
curl -s -o /dev/null -w '%{http_code}\n' https://packagefeedproxy.microsoft.io/npm/-/npm/v1/keys
# => 404
```

The wording is "supports … *if* the following conventions are followed", which describes an opt-in capability rather than an obligation. The practical consequence of not opting in is set out in section 5.5: three widely deployed clients stop working.

### 5.4 `dist.integrity` is what the signature is computed over

`dist.integrity` has been documented since April 2017 in [`package-metadata.md`](https://github.com/npm/registry/blob/main/docs/responses/package-metadata.md):

> `integrity`: since Apr 2017, string in the format `<hashAlgorithm>-<base64-hash>`

Two qualifications on that source, so it is not relied on for more than it says: the repository describes itself as "A collection of archived documentation about registry endpoints/API" (it is not formally GitHub-archived, but has issues disabled and was last pushed 2024-06-02), and it does not mention ECDSA `dist.signatures` at all — the only signature field it documents is the legacy PGP `npm-signature`. It is cited here for `dist.integrity` alone; the signatures convention comes from `docs.npmjs.com`.

The argument does not depend on that document. npm's signing formula is `${package.name}@${package.version}:${package.dist.integrity}`. Without `dist.integrity` there is no value for a signature to be computed over, so signature verification is impossible by construction — passing signatures through without integrity would not help. This also means the two fields must be restored together.

### 5.5 Three independent clients break

**npm CLI** — `npm audit signatures` against a project installed through the proxy:

```
npm warn Fetching verification keys using TUF failed.  Fetching directly from https://packagefeedproxy.microsoft.io/npm/.
npm error found no dependencies to audit that were installed from a supported registry
```

npm's own CLI classifies the feed as not "a supported registry".

**Corepack** — bundled with Node.js from 14.19.0 up to 25.0.0, and the original implementation of the `packageManager` field. Version 0.35.0 with `COREPACK_NPM_REGISTRY` pointed at the proxy:

```
Error: Server answered with HTTP 404 when performing the request to
https://packagefeedproxy.microsoft.io/npm/pnpm/11.10.0
    at fetchTarballURLAndSignature (.../corepack.cjs:12914:27)
```

Corepack fails at the documented `GET /{package}/{version}` endpoint before reaching its signature check. Had that endpoint existed, [`sources/npmRegistryUtils.ts`](https://github.com/nodejs/corepack/blob/main/sources/npmRegistryUtils.ts) would have failed next:

```ts
export function verifySignature({signatures, integrity, packageName, version}) {
  if (!Array.isArray(signatures) || !signatures.length)
    throw new Error(`No compatible signature found in package metadata`);
  …
  verifier.end(`${packageName}@${version}:${integrity}`);
```

and its version identifiers fall back to SHA-1 exactly as pnpm's lockfiles do:

```ts
integrity ? `sha512.${…}` : `sha1.${shasum}`
```

Corepack's trusted key `SHA256:DhQ8wR5APBvFHLF/+Tc+AYvPOdTpcIDqOhxsBHRwC7U`, hard-coded in [`config.json`](https://github.com/nodejs/corepack/blob/main/config.json), is the same `keyid` that appears in npmjs.org's packument for `pnpm@11.10.0` in [section 2](#2-what-the-proxy-returns-vs-the-public-registry).

**pnpm** ≥ 11.5.3 — the failure this repository reproduces.

The breakage therefore spans the npm CLI, a Node.js-distributed tool, and a third package manager, across two independently written verifiers of the same convention. It is not a pnpm-specific expectation.

### 5.6 Two documented endpoints return 404

Independently of signatures, [`REGISTRY-API.md`](https://github.com/npm/registry/blob/main/docs/REGISTRY-API.md) lists `GET /{package}/{version}` as a Package Endpoint. Both the proxy and the underlying feed return 404 for it, which is sufficient on its own to break Corepack:

```bash
curl -s -o /dev/null -w '%{http_code}\n' https://packagefeedproxy.microsoft.io/npm/pnpm/11.10.0
# => 404
```

`GET /-/npm/v1/keys`, required by convention 2 above, also returns 404.

### 5.7 Other registries preserve this metadata

Rewriting tarball URLs while preserving the rest of `dist` is established practice:

| Proxy | `dist.integrity` | `dist.signatures` | Evidence |
| --- | --- | --- | --- |
| **Verdaccio** | preserved | preserved | [`convertDistRemoteToLocalTarballUrls.ts`](https://github.com/verdaccio/verdaccio/blob/master/packages/core/tarball/src/convertDistRemoteToLocalTarballUrls.ts) spreads `...distName` and replaces only `tarball` |
| **JFrog Artifactory** | preserved | preserved on the package endpoint, stripped on the version endpoint | [nodejs/corepack#808](https://github.com/nodejs/corepack/issues/808), [#537](https://github.com/nodejs/corepack/issues/537) |
| **AWS CodeArtifact** | reported absent | reported absent | community reports only — unverified |
| **Azure Artifacts** | stripped | stripped | measured above |

Verdaccio is the closest precedent: a caching proxy that rewrites `dist.tarball` to its own host **and** passes signatures through, and the registry pnpm's own end-to-end tests run against. pnpm's claim that "an npm mirror works transparently" holds for Verdaccio and not for this proxy; the difference lies entirely in the metadata, not in the proxying model.

Azure Artifacts is not alone in stripping signatures — Artifactory strips them on one endpoint, and CodeArtifact reportedly strips them too. Two Corepack issues report the identical failure mode at a different vendor and pre-date this report:

- [#808](https://github.com/nodejs/corepack/issues/808) — *"verifySignature fails when registry returns dist.signatures on package root but not on version endpoint"*
- [#537](https://github.com/nodejs/corepack/issues/537) — *"Corepack breaks when `integrity` key is missing from metadata, from artifactory"*

That the behaviour is not unique does not make it correct, and the reference open-source implementation shows it is straightforward to avoid.

### 5.8 Anticipated objections

| Objection | Response |
| --- | --- |
| "There is no formal standard to violate." | Accurate, and no formal-standards claim is made here. The request is compatibility with the npm CLI, Corepack and pnpm, as measured in 5.5. |
| "The convention is opt-in — 'supports … *if*'." | Also accurate. But the feed is offered as an npm-compatible registry with npmjs.org as an upstream, and not opting in breaks `packageManager` pinning and `npm audit signatures` for every consumer of the feed. |
| "This is a proxy, not a signing registry; it cannot sign with npm's keys." | No signing is requested. The request is to pass npm's existing signatures through, as Verdaccio does. The `keyid` field identifies npm as the signer. |
| "Byte-identical tarballs cannot be guaranteed, so npm's signatures might not validate." | Testable, and tested: the SHA-512 of the proxied tarball equals npm's signed `dist.integrity` exactly ([section 2](#2-what-the-proxy-returns-vs-the-public-registry)). The bytes are identical; only the metadata is dropped. |
| "Feeds mix upstreams, so partial signatures would mislead." | Packages sourced from npmjs.org can carry their npm signatures; packages published directly to the feed simply have none. That is the position of every mixed registry, and it is what `keyid` disambiguates. |

### 5.9 Where the fix belongs

**With Azure DevOps.** pnpm's behaviour here matches npm's published convention and Corepack's independent implementation of it, and failing closed is the correct choice for code that is about to be executed: to a verifier, "no signature" is indistinguishable from "unsigned bytes supplied by an attacker". The `shasum` the proxy does provide is not a substitute, because it attests only that the bytes match what the proxy says they should be, not that npm ever published them.

There is no corresponding relaxation to ask of pnpm. Removing the check would reintroduce the vulnerability [#12292](https://github.com/pnpm/pnpm/pull/12292) was written to close, and would not help users of the npm CLI or Corepack, which apply the same requirement.

---

## 6. Summary: where each fix belongs

| Defect in the packument | Client check it trips | Documented expectation? | Fix belongs with |
| --- | --- | --- | --- |
| `dist.tarball` on a different origin from the packument | pnpm's integrity-only resolution check ([#12296](https://github.com/pnpm/pnpm/pull/12296)) | **No.** No specification requires a canonical or same-host tarball URL, and pnpm accepts non-derivable URLs elsewhere. | **pnpm**, primarily — relax the rule. Azure DevOps secondarily, as portability hygiene. |
| `dist.signatures` absent | pnpm signature verification ([#12292](https://github.com/pnpm/pnpm/pull/12292)), `npm audit signatures`, Corepack `verifySignature` | **Yes.** npm's published convention for third-party registries. | **Azure DevOps** |
| `dist.integrity` absent | the above (nothing to sign over); pnpm 12.x rejects outright ([#12394](https://github.com/pnpm/pnpm/pull/12394)); SHA-1 lockfile downgrade before that | **Yes.** Documented since April 2017 and structurally required by the signing formula. | **Azure DevOps** |
| `GET /{package}/{version}` returns 404 | Corepack, at its first request | **Yes.** Listed as a Package Endpoint in `REGISTRY-API.md`. | **Azure DevOps** |
| `GET /-/npm/v1/keys` returns 404 | convention 2 for signature support | **Yes.** Named explicitly in npm's third-party registry convention. | **Azure DevOps** |

### Requested of pnpm

Accept a stored `tarball` alongside `integrity` for package-manager dependencies, leaving signature verification unchanged (see [4.5](#45-where-the-fix-belongs)). This is the only change that unblocks existing users of registries whose tarball origin differs from their packument origin, without those registries shipping anything.

### Requested of Azure DevOps

For packages whose upstream is npmjs.org, pass the upstream `dist` object through unmodified except for `tarball`:

1. `dist.signatures` — verbatim.
2. `dist.integrity` — verbatim. The signature is computed over this value, so item 1 is inert without it.
3. `dist.tarball` — rewriting is expected; point it at the same endpoint that served the packument.
4. Serve `GET /{package}/{version}` and `GET /-/npm/v1/keys`, both currently 404.

Items 1–2 are what npm's third-party-registry convention describes. Items 3–4 are conformance with documented endpoints and with the behaviour of other npm proxies.

Items 1 and 2 together resolve Issue B; item 3 resolves Issue A from the feed side. Either the pnpm change or item 3 unblocks version pinning, but only items 1–2 restore signature verification, `npm audit signatures`, and Corepack support.

---

## 7. Which pnpm versions are affected

Version pinning has existed for far longer than this breakage. The two checks above are recent, so there is a wide band of pnpm versions that self-install correctly behind the proxy.

### When the feature appeared

Automatic version switching landed in **pnpm 9.7.0** ([#8363](https://github.com/pnpm/pnpm/pull/8363), commit `26b065c`, which added `pnpm/src/switchCliVersion.ts`). It was opt-in there, behind `manage-package-manager-versions=true`, and became the default in **10.0.0** (commit `dfcf034`, `feat!: set manage-package-manager-versions to true`). pnpm 9.6.0 and earlier ignore `packageManager` entirely.

### When it started failing

Both checks were added on the same day and shipped in the same release, **11.5.3**:

| Check | Commit / PR | First release |
| --- | --- | --- |
| Registry package path + integrity-only resolution | `822beb5` / [#12296](https://github.com/pnpm/pnpm/pull/12296) | 11.5.3 |
| npm registry signature verification | `5f2bb9f` / [#12292](https://github.com/pnpm/pnpm/pull/12292) | 11.5.3 |

Both PRs are analysed in [The upstream PRs behind this](#8-the-upstream-prs-behind-this) below.

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
- **The bootstrap cannot be redirected from the command line.** Neither `--registry=…` nor `npm_config_registry=…` changed which registry the pinned pnpm was resolved from (tested on 11.5.3 and 11.15.1 with all caches cleared); a repo-local `.npmrc` is ignored as well. Only the user-level `~/.npmrc` steers it. That is deliberate — see [#12296](https://github.com/pnpm/pnpm/pull/12296) below — but it means the usual "just point at npmjs.org for this one command" workaround is unavailable.

---

## 8. The upstream PRs behind this

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

The first bullet is also why the ordinary escape hatches are gone. The bootstrap deliberately reads only non-project config, and in practice (see the traps above) not even `--registry` or `npm_config_registry` redirected it in testing — leaving the user-level `~/.npmrc` as the only lever.

### [#12394](https://github.com/pnpm/pnpm/pull/12394) — `fix: sync pacquet lockfile output with pnpm`

Merge commit [`baf1502`](https://github.com/pnpm/pnpm/commit/baf15021ec134d55a604b1af42552d670957e4fa), merged 2026-06-14. Not one of the two checks, but it is where the third, distinct 12.x error comes from. The Rust env-installer in `pacquet/crates/env-installer/src/resolve_package_manager_integrities.rs` now rejects a package-manager dependency that has no integrity outright:

```
Cannot resolve pnpm@11.10.0 as a package manager dependency because it has no integrity
```

Under the old behaviour the missing `dist.integrity` silently degraded to a SHA-1 derived from `shasum`. In 12.x that degradation is gone, so the proxy's third defect — stripped `dist.integrity` — becomes a hard failure in its own right, independently of the tarball URL and the signatures.

### Which PR governs which issue

The two PRs are independent controls over independent properties, which is why addressing one of them alone is not enough:

| Packument defect | Trips | Introduced by | Analysed in |
| --- | --- | --- | --- |
| `dist.tarball` on a different origin | check 1 | [#12296](https://github.com/pnpm/pnpm/pull/12296) | [Issue A](#4-issue-a--the-non-canonical-disttarball-url) |
| `dist.signatures` stripped | check 2 | [#12292](https://github.com/pnpm/pnpm/pull/12292) | [Issue B](#5-issue-b--the-stripped-distsignatures-and-distintegrity) |
| `dist.integrity` stripped | 12.x hard error (SHA-1 downgrade before that) | [#12394](https://github.com/pnpm/pnpm/pull/12394) | [Issue B](#5-issue-b--the-stripped-distsignatures-and-distintegrity) |

Restoring only the canonical tarball URL moves the failure from check 1 to check 2, as demonstrated in [section 7](#7-which-pnpm-versions-are-affected) with the direct `ms-feed-12` feed. Section [6](#6-summary-where-each-fix-belongs) sets out which side each remedy belongs on.

---

## 9. Notes

- Reproduced with pnpm 11.15.1 (global) against a repo pinning pnpm 11.10.0, on macOS arm64.
- If the global pnpm already *is* the pinned version, no switch occurs and no error appears — a different pnpm ≥ 11.5.3 is required to reproduce.
- The missing signatures also disable `npm audit signatures` for any package fetched through the proxy.
- Measurements of the proxy in this document were taken on 2026-07-30 against `https://packagefeedproxy.microsoft.io/npm/` and, where noted, the underlying feed at `https://ms-feed-12.pkgs.visualstudio.com/1es-public/_packaging/npm-public/npm/registry/`.
- A Developer Community thread on Azure Artifacts defaulting to SHA-1 is reported at `developercommunity.visualstudio.com/t/npm-packages-from-azure-artifact-defaulting-to-sha/957522`. That host was unreachable from the test environment, so it is **unverified** and is not relied on anywhere above.
