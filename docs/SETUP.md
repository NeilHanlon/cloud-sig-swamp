# SETUP — replicate the Cloud SIG packaging pipeline

This gets you from a fresh checkout to a working `sig-detect` run (and, with the
right CBS permissions, the full build/promote pipeline). Everything is
[swamp](https://github.com/swamp-club/swamp): the four model types are published
to the swamp registry, so setup is *pull + configure your own credentials*.

Assumed clone location below is `~/cloud-sig-swamp` — adjust to taste.

---

## 1. Prerequisites

| Need | Why | Where it comes from |
| --- | --- | --- |
| **swamp CLI** | runs the models/workflows | [swamp install + quickstart](https://swamp-club.com/manual) |
| **A swamp registry account** | to `swamp extension pull` | free account at [swamp-club.com](https://swamp-club.com), then `swamp auth login` |
| **ACO client cert** | mTLS auth to CBS (`cbs.centos.org`) | ACO account → `centos-packager` → `centos-cert` → `~/.centos.cert` (steps below) |
| **A SIG dist-git checkout** | *only for building/updating a package* | clone from `gitlab.com/CentOS/cloud/rpms/<pkg>` (steps below) |
| **CBS group membership** | *only for real builds/tags* — read-only `sig-detect` needs just the cert | ask the Cloud SIG for `cloud` tag ACLs |
| **GitLab PAT** | *only for the fork/MR flow* | fine-grained token, scopes in §4 |

> **Read-only vs write.** `sig-detect` (the monitor) needs only a valid ACO cert.
> Building, tagging, and promoting need your account to hold the corresponding
> CBS permissions — those are the "maintainer-gated" steps in the RUNBOOK.

### Getting the prerequisites

**swamp + a registry account.** Install the CLI per the
[swamp manual](https://swamp-club.com/manual), create a free account at
[swamp-club.com](https://swamp-club.com), then `swamp auth login`. Confirm with
`swamp auth whoami` (it should print your username, not "not authenticated").

**The ACO (CentOS) client certificate** — required for *every* CBS path,
including read-only `sig-detect`:

1. Create an account at [accounts.centos.org](https://accounts.centos.org) (ACO).
2. Install the packager tools: `sudo dnf install centos-packager`.
3. Run `centos-cert` — it authenticates to ACO and writes `~/.centos.cert`
   (valid ~2 years).

> **Toolbox caveat.** The distro `centos-packager` can lag; if `centos-cert`
> misbehaves, run it from a current Fedora `toolbox`/container. The cert file it
> produces is what matters — point the koji model at that path (§5).

**A SIG dist-git checkout** (only needed once you want to *build/update* a
package, not for `sig-detect`). The SIG keeps one repo per package; clone the
one(s) you'll work on and check out the Epoxy branch:

```bash
mkdir -p ~/centos-rpms && cd ~/centos-rpms
git clone https://gitlab.com/CentOS/cloud/rpms/openstack-keystone
cd openstack-keystone && git checkout c9s-sig-cloud-epoxy
```

The RUNBOOK's build/update recipes assume packages live under `~/centos-rpms/<pkg>`.

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
`@kneel/sig-distgit`) come from the registry:

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
this repo; you create your own.** The auto-generated encryption key and the
secrets live under the gitignored `.swamp/` directory (local to your checkout,
never committed) — back that up if you don't want to re-auth after a fresh clone.

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
# CBS Koji, SSL/mTLS auth via your ACO cert.
# Use an absolute path for the cert ("~" is NOT expanded inside a global-arg value).
swamp model create @kneel/koji cbs-koji \
  --global-arg server=https://cbs.centos.org/kojihub \
  --global-arg authtype=ssl \
  --global-arg cert="$HOME/.centos.cert"

# SIG dist-git scanner (defaults to group CentOS/cloud/rpms, branch c9s-sig-cloud-epoxy)
swamp model create @kneel/sig-distgit cloud-sig-epoxy

# Upstream OpenStack release feed
swamp model create @kneel/openstack-releases openstack-epoxy

# GitLab (only for the fork/MR flow)
swamp model create @webframp/gitlab sig-gitlab \
  --global-arg host=gitlab.com \
  --global-arg 'token=${{ vault.get("gitlab", "TOKEN") }}'
```

> The flag is `--global-arg key=value` (repeatable). If a flag or arg name is
> rejected on your swamp version, run `swamp model type describe @kneel/koji --json`
> to see the exact argument names, or set them afterward with `swamp model edit`.

## 6. Verify — run the read-only monitor

```bash
swamp model method run cbs-koji login          # SSL login; should report a session
swamp workflow run sig-detect                  # ~2–4 min; hits CBS read-only
```

`sig-detect` prints the actionable queue and writes the `sig-promote` report
JSON under `.swamp/data/workflow/<workflow-id>/report-kneel-sig-distgit-sig-promote-json/<run>/raw`,
where `<workflow-id>` is the workflow **definition** id (stable across runs) and
`<run>` increments per run. To grab the latest report regardless of ids:

```bash
jq . "$(ls -t .swamp/data/workflow/*/report-kneel-sig-distgit-sig-promote-json/*/raw | head -1)"
```

If that runs green, your setup is correct.

---

## 7. Next steps

- **Manual package work** (build / update / promote a package by hand):
  [RUNBOOK.md](RUNBOOK.md).
- **The five workflows** are in [`../workflows/`](../workflows) — `sig-detect`
  (monitor), `sig-scratch-build` (validate), `sig-propose` (draft build),
  `sig-undraft` (promote on merge), `sig-tag-promote` (human-gated tag step).
- **Current package status** for Epoxy: generate it fresh with `swamp workflow run sig-detect` (a point-in-time report is not committed).

## Constants (Epoxy / c9s)

- Build target: `cloud9s-openstack-epoxy-el9s` (dest tag = `-candidate`)
- Promotion tags: `cloud9s-openstack-epoxy-{candidate,testing,release}`
- SIG lookaside: `https://src.sigs.centos.org` (**not** `sources.stream.centos.org`)
- SIG dist-git: `gitlab.com/CentOS/cloud/rpms`, branch `c9s-sig-cloud-epoxy`
