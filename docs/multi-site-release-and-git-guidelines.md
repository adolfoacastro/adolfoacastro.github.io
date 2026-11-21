# Multi-Site Release & Git Guidelines

**Audience:** teams shipping versioned software to manufacturing (or similar) sites that require planned downtime, cannot continuously deploy, and may run on slightly different OpenShift infrastructure per plant.

**Branching style:** GitFlow-oriented (`develop` + `main` + `release/*` + `hotfix/*`) to stay close to existing team habits and CI/CD that already key off those branches.

**Environments:** `dev` → `test` → `qa` → `prod`  
**Artifact store:** JFrog Artifactory (or equivalent).  
**Runtime:** OpenShift (per-plant clusters or projects).  
**Status:** Hypothetical reference practices — adapt names and tooling to your org.

---

## 1. Goals

1. Ship **approved, immutable** versions during each site’s maintenance window.
2. Keep building the **next** release on `develop` while sites still run an older version on `prod`.
3. Apply **hotfixes** (including one-site emergencies) without pulling unfinished work onto production.
4. Express **plant OpenShift differences** without forking the application per site.
5. Make “what is Plant X running?” answerable from **inventory**, not from a branch tip.
6. Preserve **`develop` / `main`** as first-class branches so existing pipelines need minimal refactor.

---

## 2. Core principles

| Principle | Practice |
|---|---|
| GitFlow-shaped integration | Features → `develop`; releases stabilize on `release/*`; production history lives on `main` + tags. |
| Separate branch tips from what plants run | CI may **build** from `develop` / `main` / `release/*`; **prod (and usually qa) deploy** only **immutable Artifactory artifacts** tied to git tags. |
| One product line | Avoid long-lived per-plant application branches (`plant-austin`, etc.). |
| Two axes of change | **App version** (git tag → artifact) and **plant shape** (OpenShift overlay/values) are independent. |
| Fix from what’s installed | Hotfixes branch from **`main` / the production tag** the affected site runs, not from `develop`. |
| Config before forks | Prefer plant config / profiles / feature flags over site-specific code forks. |
| Promote, don’t rebuild | Build once for a release line; promote the same artifact digest through Artifactory env repos. |
| Record desired state | Each plant (or the single central target) pins digest + overlay in GitOps / inventory. |
| Same playbook for 1 or N plants | Multi-plant vs single-plant changes **overlay/inventory count**, not the branch model. |

---

## 3. Branching model (GitFlow-oriented)

### 3.1 Long-lived

| Branch | Role |
|---|---|
| `develop` | Integration branch for the **next** version. Default target for feature PRs. Typical source for **dev** (and often early **test**) builds. |
| `main` | Production-ready history. Receives merges from `release/*` at GA and from `hotfix/*`. Tags for shipped versions are cut here (or on the release/hotfix branch and merged back). **Not** a “deploy whatever is tip” signal for plants—tags/artifacts are. |

### 3.2 Short-lived / release lines

| Branch | When to create | What belongs on it | When to delete |
|---|---|---|---|
| `feature/<slug>` | Normal development | Feature work | After merge to `develop` |
| `release/X.Y` | When version X.Y is frozen from `develop` | RC bugfixes only (no new features) | After merge to `main` **and** back-merge to `develop` |
| `hotfix/X.Y.Z-<slug>` | Production emergency on a shipped line | Minimal fix | After merge to `main` **and** back-merge to `develop` (and any open `release/*`) |

### 3.3 Tags (deployment-grade references)

| Tag pattern | Meaning |
|---|---|
| `vX.Y.Z-rc.N` | Release candidate under test / QA approval |
| `vX.Y.Z` | Generally available release (on `main` history) |
| `vX.Y.Z+<plant>` | Rare site-qualified build when code *must* differ for one plant (prefer config) |

**Rule:** Plant **prod** deploys **tags → Artifactory digests**, never floating `develop` or untagged `main`.

### 3.4 Flow diagram

```text
develop:     ──●──●──●──●──●──●──●──●──▶   (next version)
                  \              ^
release/2.4:       ●──●──●──●────│──▶ merge back
                        \        │
                         tag v2.4.0-rc.* / v2.4.0
                          \      │
main:        ──●───────────●─────●──▶   (production history + tags)
               ^           ^
               │           │
hotfix:        └──●── tag v2.3.2
                    \
                     └── back-merge → develop (+ open release/*)
```

---

## 4. Environments: `dev` → `test` → `qa` → `prod`

Environments are a **promotion ladder for artifacts**, not four long-lived app forks.

### 4.1 Recommended mapping

| Environment | Typical git source for **builds** | What gets **deployed** | Purpose |
|---|---|---|---|
| **dev** | `develop` (or latest successful build from `develop`) | Mutable / frequently refreshed artifact from `develop` is OK | Integration, developer validation |
| **test** | `develop` or early `release/X.Y` | Prefer a **named build id**; start treating digests as sticky | Automated / system test |
| **qa** | `release/X.Y` or RC tag `vX.Y.Z-rc.N` | **Immutable** RC digest only | Formal acceptance, plant-like validation |
| **prod** | GA tag `vX.Y.Z` on `main` history | **Immutable** GA digest only; plant downtime windows | Live manufacturing / central production |

### 4.2 Artifactory promotion (illustrative)

```text
build from develop     →  libs-dev  / docker-dev      →  deploy to env:dev
build from release/RC  →  libs-test → libs-qa         →  env:test then env:qa
                         (same digest promoted)
tag vX.Y.Z + promote   →  libs-prod / docker-prod     →  env:prod (per plant window)
```

Exact repo names vary (`*-dev`, `*-test`, `*-qa`, `*-prod`). Keep the idea: **same bytes move up**; do not rebuild “v2.4.0” differently for qa vs prod.

### 4.3 CI/CD compatibility note

Pipelines that already trigger on `develop` and `main` can stay:

- **`develop` push/PR** → build → publish to **dev** (optionally auto-deploy **dev**).  
- **`release/*` push/PR** → build → publish → deploy/promote toward **test** / **qa**.  
- **`main` merge + tag** → ensure GA artifact is in **prod** repo; plant deploy jobs consume the **tag/digest**, not “latest main”.  
- **`hotfix/*`** → same path as a fast release: build → qa as required → tag → prod repo → plant(s).

Slight redundancy of `develop` + `main` is accepted here to avoid a large CI redesign.

### 4.4 Multi-plant vs single-plant (central) services

| | Multi-plant service | Single-plant / central service |
|---|---|---|
| Branches | Same GitFlow | Same GitFlow |
| Env ladder | Same `dev/test/qa/prod` | Same |
| Prod inventory | N plant pins (may stagger versions) | One pin |
| OpenShift overlays | N plant overlays | One overlay (or `base` only) |

---

## 5. Release candidate lifecycle

1. **Integrate** on `develop` (features merge here; **dev** / early **test** stay current).  
2. **Freeze:** cut `release/X.Y` from `develop`.  
3. **Stabilize:** only bugfixes on `release/X.Y`; back-port important fixes to `develop` continuously.  
4. **Package:** CI builds from `release/X.Y` / RC tags → Artifactory (**test** / **qa** repos).  
5. **Approve:** QA signs off on a **specific** RC digest (`vX.Y.Z-rc.N`), not “whatever is on the branch tip tomorrow.”  
6. **GA:** merge `release/X.Y` → `main`, tag `vX.Y.Z`, merge `release/X.Y` → `develop`, promote digest to **prod** Artifactory.  
7. **Rollout:** each plant (or the single central target) deploys that digest in its window.  
8. **Close:** delete `release/X.Y` when merges are done.

During the RC window, plants still on an older prod tag must be hotfixed from **`main` / that tag**, not only from the new `release/*`.

---

## 6. Hotfix process

### 6.1 Shared bug (affects prod on the current shipped line)

1. Read inventory: which tag/digest is installed.  
2. Branch `hotfix/X.Y.Z-<slug>` from `main` (or from that tag).  
3. Fix, PR, CI; promote through **test** / **qa** as policy requires.  
4. Merge hotfix → `main`, tag `vX.Y.Z`, publish/promote to **prod** Artifactory.  
5. Deploy to requesting plant(s) in their window.  
6. **Back-merge** hotfix → `develop` and any open `release/*` so the next train does not regress.

### 6.2 Site-specific behavior

| Situation | Preferred approach |
|---|---|
| Different OpenShift wiring | Plant **overlay / Helm values** only; same app tag |
| Plant parameters | ConfigMaps / profiles / flags; same app tag |
| Code path only one plant needs | Prefer config; if unavoidable, `vX.Y.Z+plant` from hotfix, pin only that plant, absorb into next release on `develop` |

### 6.3 Anti-patterns

- Long-lived `plant-<name>` application branches  
- Hotfixing only on `develop` while prod still runs an older tag  
- Deploying **prod** from floating `main` without a tag/digest pin  
- Rebuilding the same version differently per environment or plant  
- Skipping back-merge of hotfix into `develop`  

---

## 7. OpenShift & plant differences

### 7.1 Split responsibilities

| Concern | Where it lives |
|---|---|
| Application logic | App repo → GitFlow branches → tag → Artifactory |
| Env + plant shape | GitOps overlays (env and/or plant) |
| Desired version | Inventory / Argo pins per plant (or one pin for central) |

### 7.2 Suggested layout

```text
app/                              # application source (GitFlow in git)
  ...

ops/ or manifests/
  base/
  overlays/
    envs/
      dev/
      test/
      qa/
      prod/                       # shared prod base
    plants/                       # prod plant deltas (omit or single entry if central-only)
      austin/
      houston/
  inventory/
    austin.yaml                   # appVersion + digest
    houston.yaml
    # or central.yaml for a single-target service
```

Env overlays can hold non-prod differences; **plant** overlays hold manufacturing-site OpenShift drift on top of **prod**.

### 7.3 Deploy reference

```text
Target =
    Artifactory artifact (version + digest)
  + env overlay (dev|test|qa|prod)
  + plant overlay (if multi-plant prod)
```

Not: `develop` tip, untagged `main`, or `:latest` for **qa**/**prod**.

### 7.4 Staggered prod versions

Multi-plant: Austin on `v2.3.2`, Houston still on `v2.3.1` is normal. Inventory must show it.  
Single-plant central service: one pin; still use tags for rollback clarity.

---

## 8. Artifactory / JFrog practices

### 8.1 Pipeline sketch

```text
develop commit     → build → Artifactory *-dev            → env:dev
release / RC tag   → build → promote *-test → *-qa        → env:test / env:qa
main + tag vX.Y.Z  → promote same digest → *-prod         → env:prod (plants)
hotfix tag         → build → qa as required → *-prod      → env:prod
```

### 8.2 Immutability rules

1. **qa** and **prod** use **digest-pinned** artifacts.  
2. **dev** may track `develop` more loosely; still prefer unique build ids.  
3. **Build once per version/RC**, promote across `test` → `qa` → `prod`.  
4. Retain prior prod digests for **rollback** during a failed downtime window.  
5. Record git branch/tag, SHA, Artifactory path, digest, and env/plant in metadata.

### 8.3 Traceability checklist (every **qa** / **prod** deploy)

- [ ] Git tag (RC or GA)  
- [ ] Commit SHA  
- [ ] Source branch (`release/*`, `hotfix/*`, or `main` at tag)  
- [ ] Artifactory path / image reference  
- [ ] Digest (`sha256:…`)  
- [ ] Env + plant overlay identity  
- [ ] Approver / change ticket  

---

## 9. What to run and deploy from (quick reference)

| Activity | Git reference | Typical env |
|---|---|---|
| Feature work | `feature/*` → `develop` | — |
| Day-to-day integration builds | `develop` | **dev** (→ early **test**) |
| RC stabilization | `release/X.Y` | **test** → **qa** |
| Production history / GA | merge to `main` + tag `vX.Y.Z` | **prod** |
| Emergency fix | `hotfix/*` from `main`/tag → back-merge `develop` | **qa** → **prod** |
| **Approve for plant rollout** | Tag + Artifactory digest | **qa** sign-off → **prod** |
| Rollback | Previous known-good digest | **prod** |

---

## 10. Roles & responsibilities (suggested)

| Role | Responsibility |
|---|---|
| Developers | PR to `develop`; no direct feature commits to `main` |
| Release manager | Cut `release/*` / `hotfix/*`, tags, Artifactory promotion gates |
| QA | Sign off **digests** in **qa**; gate promotion to **prod** |
| Platform / OpenShift | Env + plant overlays, GitOps, secrets/config per env |
| Plant IT / ops | Downtime deploy of pinned prod digest; update inventory |

---

## 11. Decision guide

**“Where do I commit?”**

- New feature → `feature/*` → **`develop`**  
- Bug in the RC → **`release/X.Y`** (+ merge back to `develop`)  
- Bug in prod → **`hotfix/*` from `main`/tag** → `main` + tag → back-merge **`develop`**  

**“What does each env run?”**

- **dev:** builds from `develop` (mutable OK)  
- **test:** builds from `develop` or `release/*` (prefer pinned build ids)  
- **qa / prod:** immutable RC/GA digests only  

**“OpenShift differs at one plant?”**

- Overlay/values — not a plant app branch.

**“Central service with one plant?”**

- Same GitFlow and env ladder; one overlay and one inventory pin.

**“Why keep both `develop` and `main` if tags are the real prod pin?”**

- Slight redundancy is intentional: matches team mental model and existing CI triggers, while tags/digests remain the **authoritative** prod deploy reference.

---

## 12. Summary policy (one page)

1. Use **GitFlow:** `develop` + `main` + `release/*` + `hotfix/*` + tags.  
2. Map **dev → test → qa → prod** to increasing immutability; qa/prod are digest-pinned.  
3. **Prod** (plants or central) installs Artifactory digests from **GA tags on `main` history**, not floating branches.  
4. One application line; OpenShift drift in **env/plant overlays**.  
5. Hotfix from **`main`/prod tag**, then back-merge to **`develop`** (and open releases).  
6. **Build once, promote across env repos, inventory per target.**  
7. Multi-plant and single-plant services share this playbook; only pin/overlay cardinality changes.

---

## 13. Glossary

| Term | Meaning |
|---|---|
| `develop` | Integration branch for the next version; feeds **dev** / early **test** |
| `main` | Production-ready history; receives release/hotfix merges; home of GA tags |
| RC | Release candidate — tagged build under **qa** approval |
| Hotfix | Patch from `main`/prod tag for urgent repair |
| Env ladder | `dev` → `test` → `qa` → `prod` promotion path |
| Overlay | Env- or plant-specific OpenShift/Kustomize/Helm customization |
| Promote | Move the same artifact to a higher Artifactory repo without rebuilding |
| Digest | Content-addressed id (`sha256:…`) of an immutable artifact |
| Inventory | Record of which digest + overlay each deploy target should run |

---

*Document purpose: capture GitFlow-compatible practices for a multi-site (or single-target), downtime-bound, OpenShift + Artifactory delivery model with dev/test/qa/prod. Revise when tooling or org policy changes.*
