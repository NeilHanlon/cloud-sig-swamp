# SETUP — replicate the Cloud SIG packaging pipeline

This gets you from a fresh checkout to a working `sig-detect` run (and, with the
right CBS permissions, the full build/promote pipeline). Everything is
[swamp](https://github.com/swamp-club/swamp): the four model types are published
to the swamp registry, so setup is *pull + configure your own credentials*.

Assumed clone location below is `~/cloud-sig-swamp` — adjust to taste.

---

## 1. Prerequisites

| Need | Why | How |
| --- | --- | --- |
| **swamp CLI** | runs the models/workflows | see the [swamp install docs](https://swamp-club.com) |
| **A swamp registry account** | to `swamp extension pull` | `swamp auth login` |
| **ACO client cert** | mTLS auth to CBS (`cbs.centos.org`) | `centos-cert` (from `centos-packager`) → writes `~/.centos.cert` |
| **CBS group membership** | *only for real builds/tags* — read-only `sig-detect` needs just the cert | ask the Cloud SIG for `cloud` tag ACLs |
| **GitLab PAT** | *only for the fork/MR flow* | fine-grained token, scopes below |

> **Read-only vs write.** `sig-detect` (the monitor) needs only a valid ACO cert.
> Building, tagging, and promoting need your account to hold the corresponding
> CBS permissions — those are the "maintainer-gated" steps in the RUNBOOK.

---

## 2. Clone + initialize the repo

```bash
git clone https://github.com/NeilHanlon/cloud-sig-swamp ~/cloud-sig-swamp
cd ~/cloud-sig-swamp
swamp repo init          # if .swamp.yaml isn't already present
swamp auth whoami        # confirm you're logged in to the registry
```

## 3. Pull the extensions

All four model types (plus the `sig-promote` report, which ships inside
`@kneel/koji`) come from the registry:

```bash
swamp extension pull @kneel/koji
swamp extension pull @kneel/sig-distgit
swamp extension pull @kneel/openstack-releases
swamp extension pull @webframp/gitlab
swamp extension pull @kneel/gitlab-fork     # only if you'll open fork→upstream MRs
```

Verify they registered:

```bash
swamp model type search koji
swamp model type search sig-distgit
```

## 4. Create your vaults

Two local-encryption vaults hold per-user state — **no secrets are shipped in
this repo; you create your own.**

```bash
# koji session persistence (auto-generates its own encryption key)
swamp vault create local_encryption koji

# GitLab token (only needed for the fork/MR flow)
swamp vault create local_encryption gitlab
swamp vault put gitlab TOKEN            # paste your fine-grained PAT when prompted
```

**GitLab PAT scopes** (fine-grained, scoped to the `CentOS/cloud` group):
Merge Request *Create/Read/Merge/Update*, Code/Repository *Read*, and
Project *Fork* (for `@kneel/gitlab-fork`). Note: a fine-grained token has no
classic `api` scope, so the GitLab extension's user-level GraphQL methods
(`list_my_merge_requests`, `list_todos`) won't work — the REST methods this
pipeline uses do.

## 5. Create the model instances

```bash
# CBS Koji, SSL/mTLS auth via your ACO cert
swamp model create @kneel/koji cbs-koji \
  --arg server=https://cbs.centos.org/kojihub \
  --arg authtype=ssl \
  --arg cert='~/.centos.cert'

# SIG dist-git scanner (defaults to group CentOS/cloud/rpms, branch c9s-sig-cloud-epoxy)
swamp model create @kneel/sig-distgit cloud-sig-epoxy

# Upstream OpenStack release feed
swamp model create @kneel/openstack-releases openstack-epoxy

# GitLab (only for the fork/MR flow)
swamp model create @webframp/gitlab sig-gitlab \
  --arg host=gitlab.com \
  --arg token='${{ vault.get("gitlab", "TOKEN") }}'
```

> Exact `--arg`/global-argument syntax can vary by swamp version; if a flag is
> rejected, run `swamp model type describe @kneel/koji --json` to see the
> argument names and set them via `swamp model edit`.

## 6. Verify — run the read-only monitor

```bash
swamp model method run cbs-koji login          # SSL login; should report a session
swamp workflow run sig-detect                  # ~2–4 min; hits CBS read-only
```

`sig-detect` prints the actionable queue and writes the `sig-promote` report
JSON under `.swamp/data/workflow/<id>/report-kneel-koji-sig-promote-json/`. If
that runs green, your setup is correct.

---

## 7. Next steps

- **Manual package work** (build / update / promote a package by hand):
  [RUNBOOK.md](RUNBOOK.md).
- **The five workflows** are in [`../workflows/`](../workflows) — `sig-detect`
  (monitor), `sig-scratch-build` (validate), `sig-propose` (draft build),
  `sig-undraft` (promote on merge), `sig-tag-promote` (human-gated tag step).
- **Current package status** for Epoxy: [SIG-STATUS.md](SIG-STATUS.md).

## Constants (Epoxy / c9s)

- Build target: `cloud9s-openstack-epoxy-el9s` (dest tag = `-candidate`)
- Promotion tags: `cloud9s-openstack-epoxy-{candidate,testing,release}`
- SIG lookaside: `https://src.sigs.centos.org` (**not** `sources.stream.centos.org`)
- SIG dist-git: `gitlab.com/CentOS/cloud/rpms`, branch `c9s-sig-cloud-epoxy`
