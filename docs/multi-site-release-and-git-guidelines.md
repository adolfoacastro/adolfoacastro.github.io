# Multi-Site Release & Git Guidelines

**Audience:** teams shipping versioned software to manufacturing (or similar) sites that require planned downtime, cannot continuously deploy, and may run on slightly different OpenShift infrastructure per plant.

**Artifact store:** JFrog Artifactory (or equivalent).  
**Runtime:** OpenShift (per-plant clusters or projects).  
**Status:** Hypothetical reference practices — adapt names and tooling to your org.

---

## 1. Goals

1. Ship **approved, immutable** versions during each site’s maintenance window.
2. Keep building the **next** release while sites still run an older version.
3. Apply **hotfixes** (including one-site emergencies) without pulling unfinished work onto production.
4. Express **plant OpenShift differences** without forking the application per site.
5. Make “what is Plant X running?” answerable from **inventory**, not from a branch tip.

---

## 2. Core principles

| Principle | Practice |
|---|---|
| Separate develop from deploy | Develop on `main`; **deploy only immutable Artifactory artifacts** built from git tags. |
| One product line | Avoid long-lived per-plant application branches (`plant-austin`, etc.). |
| Two axes of change | **App version** (git tag → artifact) and **plant shape** (OpenShift overlay/values) are independent. |
| Fix from what’s installed | Hotfixes branch from the **production tag** the affected site runs, not from `main` or the next RC. |
| Config before forks | Prefer plant config / profiles / feature flags over site-specific code forks. |
| Promote, don’t rebuild | Build once from a tag; promote the same artifact through Artifactory; plants pull that digest. |
| Record desired state | Each plant pins `appVersion` / image **digest** + overlay identity in GitOps or a release manifest. |

---

## 3. Branching model

### 3.1 Long-lived

| Branch | Role |
|---|---|
| `main` | Integration trunk for the next version. Not a deployment reference for plants. |

Optional: a `develop` branch only if your org already requires it; prefer a single trunk (`main`) when possible.

### 3.2 Short-lived / release lines

| Branch | When to create | What belongs on it | When to delete |
|---|---|---|---|
| `release/X.Y` | When version X.Y is frozen as an RC | RC bugfixes and cherry-picks only | After GA tag is cut and back-merges are done |
| `hotfix/X.Y.Z-<slug>` | When a site (or all sites) on X.Y need an emergency fix | Minimal fix for the installed line | After patch tag is cut and forward-ported |
| `feature/<slug>` | Normal development | Feature work | After merge to `main` |

### 3.3 Tags (deployment-grade references)

| Tag pattern | Meaning |
|---|---|
| `vX.Y.Z-rc.N` | Release candidate under test/approval |
| `vX.Y.Z` | Generally available patch/minor release |
| `vX.Y.Z+<plant>` | Optional site-qualified build when code *must* differ for one plant (rare; prefer config) |

**Rule:** Plants deploy **tags → Artifacts**, never `main` or an open feature branch.

### 3.4 Flow diagram

```text
main:            ──●──●──●──●──●──●──●──▶   (next work)
                      \           \
release/2.4:           ●──●──●────●── tag v2.4.0
                         ↑
                       RC-only fixes

prod 2.3.x:      tag v2.3.1 ── hotfix branch ── tag v2.3.2
                                      \
                                       └── cherry-pick → main + open release/*
```

---

## 4. Release candidate lifecycle

1. **Freeze:** cut `release/X.Y` from `main` (or from the agreed commit).
2. **Stabilize:** only bugfixes land on `release/X.Y` (PRs + CI).
3. **Package:** CI builds from the release branch / RC tag → publishes to Artifactory (e.g. `rc-local`).
4. **Approve:** QA / plant stakeholders sign off on a specific RC artifact (`vX.Y.Z-rc.N` + digest).
5. **GA:** tag `vX.Y.Z`, rebuild or retag policy as per org rules, promote artifact to `prod-local` (or equivalent).
6. **Rollout:** each plant deploys that artifact in its downtime window (plants may lag versions).
7. **Close:** merge `release/X.Y` back to `main` if needed; delete the release branch when done.

During the RC window, **do not** assume sites run the RC. Sites still on `vX.Y-1.*` must be hotfixed from **their** production tag.

---

## 5. Hotfix process

### 5.1 Shared bug (affects all plants on that version)

1. Identify the exact tag/digest Plant N runs (from inventory).
2. Branch: `hotfix/X.Y.Z-<slug>` from that tag.
3. Fix, PR, CI, test against the production line.
4. Tag `vX.Y.Z` (patch bump).
5. CI publishes immutable artifact to Artifactory; promote through repos.
6. Deploy to the requesting plant(s) in their window; others take the same patch when scheduled.
7. **Forward-port:** merge or cherry-pick into `main` and any open `release/*` so the next RC does not regress.

### 5.2 Site-specific behavior

| Situation | Preferred approach |
|---|---|
| Different OpenShift wiring (routes, StorageClass, quotas, pull secrets, node selectors) | Change **plant overlay / Helm values only**; same app tag |
| Plant parameters (lines, limits, endpoints) | ConfigMaps / plant profile / feature flags; same app tag |
| Code path only one plant needs | Prefer runtime config; if unavoidable, site-qualified tag `vX.Y.Z+plant` from the hotfix branch, pin only that plant’s inventory, and plan absorption into the next general release |

### 5.3 Anti-patterns

- Long-lived `plant-<name>` application branches  
- Hotfixing only on `main` or only on the next `release/*` while sites still run an older tag  
- Rebuilding “from main” per plant for the same logical version  
- Leaving a hotfix off `main` so the next RC reintroduces the bug  

---

## 6. OpenShift & plant differences

### 6.1 Split responsibilities

| Concern | Where it lives |
|---|---|
| Application logic | App repo → git tag → container/package in Artifactory |
| Cluster/plant shape | GitOps overlays or values per plant (`overlays/plants/<name>/`) |
| Desired version per plant | Plant inventory / Argo CD `targetRevision` or pin file |

### 6.2 Suggested layout

```text
app/                          # application source
  ...
  Dockerfile

ops/ or manifests/            # often a GitOps repo
  base/                       # shared OpenShift manifests / chart
  overlays/plants/
    austin/
    houston/
    munich/
  inventory/                  # optional explicit pins
    austin.yaml               # appVersion: v2.3.2  digest: sha256:…
    houston.yaml              # appVersion: v2.3.1  digest: sha256:…
```

### 6.3 Deploy reference per plant

```text
Plant deploy target =
    Artifactory artifact (version + digest)
  + plant OpenShift overlay
```

Not:

```text
git branch plant-austin   ❌
git branch main           ❌
image:latest              ❌
```

### 6.4 Staggered versions are normal

Austin may run `v2.3.2` after a hotfix while Houston remains on `v2.3.1` until its outage. Inventory must show that explicitly. Do not force all plants onto one branch tip.

---

## 7. Artifactory / JFrog practices

### 7.1 Pipeline

```text
git tag vX.Y.Z
  → CI checks out that tag (immutable commit)
  → build once
  → publish to Artifactory (versioned path / image tag)
  → record digest + git tag + build id in metadata
  → promote: dev-local → rc-local → prod-local (names vary)
  → plant pull from promoted repo by version/digest
```

### 7.2 Immutability rules

1. **Do not deploy `main` or `:latest`.**  
2. Prefer **digest pins** in production OpenShift (image tags can be overwritten unless retention/immutability is enforced).  
3. **Build once, promote many times** — same digest across environments and plants for a given version.  
4. RC and GA artifacts are distinct versions (`-rc.N` vs final); promotion policy should make GA provenance clear.  
5. Retain enough history to **rollback** a plant to the previous digest during a failed window.

### 7.3 Traceability checklist (every production deploy)

- [ ] Git tag  
- [ ] Commit SHA  
- [ ] Artifactory path / image reference  
- [ ] Content digest (`sha256:…`)  
- [ ] Plant overlay identity / GitOps revision  
- [ ] Who approved / which change ticket  

---

## 8. What to run and deploy from (quick reference)

| Activity | Reference |
|---|---|
| Day-to-day development | `main`, short feature branches |
| RC integration | `release/X.Y` |
| Emergency fix work | `hotfix/X.Y.Z-<slug>` from production tag |
| **Approve for plant rollout** | Git tag `vX.Y.Z` / `vX.Y.Z-rc.N` |
| **Deploy to a plant** | Artifactory artifact **digest** for that tag + plant overlay |
| Rollback | Previous known-good digest for that plant |

---

## 9. Roles & responsibilities (suggested)

| Role | Responsibility |
|---|---|
| Developers | Land work on `main`; cherry-pick only approved fixes to release/hotfix lines |
| Release manager | Cut release/hotfix branches and tags; gate Artifactory promotion |
| QA / validation | Sign off on specific digests, not “whatever is on the branch” |
| Platform / OpenShift | Own plant overlays, cluster differences, GitOps apps |
| Plant IT / ops | Execute downtime deploy of the pinned artifact; confirm inventory update |

---

## 10. Decision guide

**“Where do I commit?”**

- New feature for next version → `main`  
- Bug in the RC under test → `release/X.Y` (+ ensure `main` gets it)  
- Bug in what’s running at a plant → `hotfix/…` from that plant’s tag → then forward-port  

**“What does the plant install?”**

- Always a **promoted Artifactory artifact** identified by version + digest, with that plant’s overlay.

**“OpenShift differs at one plant — new app branch?”**

- No. Change the plant overlay/values. New app branch only if you are producing a rare site-qualified **tag**, and even then keep it short-lived.

**“Can plants run different versions?”**

- Yes. Expected under downtime scheduling. Track in inventory.

---

## 11. Summary policy (one page)

1. Use **trunk + release branches + hotfix-from-tag** (Git Flow–style release/hotfix), not continuous deploy from `main`.  
2. **Tags** mark releasable commits; **Artifactory digests** are what plants run.  
3. Keep **one application line**; put OpenShift drift in **per-plant overlays**.  
4. Prefer **config** for site-specific behavior; avoid permanent plant forks.  
5. Hotfix the **installed** version, then **forward-port** to `main` and open RCs.  
6. **Build once, promote, digest-pin, inventory.**  

---

## 12. Glossary

| Term | Meaning |
|---|---|
| Trunk / `main` | Primary integration branch; not a plant deploy target |
| RC | Release candidate — tagged build under approval |
| Hotfix | Patch built from a production tag for urgent repair |
| Overlay | Plant-specific OpenShift/Kustomize/Helm customization |
| Promote | Move or copy the same artifact to a higher Artifactory repo without rebuilding |
| Digest | Content-addressed id (`sha256:…`) of an immutable artifact |
| Inventory | Record of which digest + overlay each plant is supposed to run |

---

*Document purpose: capture agreed practices for a multi-site, downtime-bound, OpenShift + Artifactory delivery model. Revise when tooling or org policy changes.*
