# cloud-sig-swamp

Tooling to **detect, build, update, and promote** [CentOS Cloud SIG](https://wiki.centos.org/SpecialInterestGroup/Cloud)
(RDO / OpenStack) packages on [CBS](https://cbs.centos.org), built as declarative
[swamp](https://github.com/swamp-club/swamp) workflows over a native Koji client.

It answers two questions a SIG maintainer asks constantly:

1. **What needs my attention?** — `sig-detect` joins the three CBS promotion tags
   (`-candidate` / `-testing` / `-release`), the SIG dist-git spec versions, and
   upstream OpenStack releases into a single actionable queue (unbuilt /
   promote-testing / promote-release / behind-upstream), with the exact command
   per row. Read-only, cron-safe.
2. **How do I move a package through the pipeline?** — a build-once flow: validate
   a revision (`sig-scratch-build`), open a fork→upstream MR, build a **draft**
   (`sig-propose`), promote it on merge (`sig-undraft`), then tag it up the ladder
   at the human gates (`sig-tag-promote`).

## Quickstart

```bash
git clone https://github.com/NeilHanlon/cloud-sig-swamp
cd cloud-sig-swamp
# one-time setup: install swamp, pull extensions, create your vaults + models
#   → see docs/SETUP.md
swamp workflow run sig-detect        # read-only; prints what's actionable
```

## What's here

| Path | What |
| --- | --- |
| [`docs/SETUP.md`](docs/SETUP.md) | Full setup: swamp install, extension pulls, vault creds, model instances |
| [`docs/RUNBOOK.md`](docs/RUNBOOK.md) | No-LLM operator recipes — build / update / promote a package by hand |
| [`docs/SIG-STATUS.md`](docs/SIG-STATUS.md) | Current Epoxy package delta + pipeline status report |
| [`workflows/`](workflows) | The five swamp workflows |

## Extensions

All published to the swamp registry — `docs/SETUP.md` pulls them:

| Extension | Role |
| --- | --- |
| `@kneel/koji` | Native CBS Koji client (SSL/Kerberos); carries the `sig-promote` report |
| `@kneel/sig-distgit` | Scans a SIG dist-git group for spec versions + `sources` state |
| `@kneel/openstack-releases` | Upstream OpenStack release feed |
| `@webframp/gitlab` | GitLab MR operations |
| `@kneel/gitlab-fork` | Adds `fork_project` (fork→upstream MRs) to `@webframp/gitlab` |

## Scope

Configured for **CentOS Stream 9 · OpenStack Epoxy** (`cloud9s-openstack-epoxy-*`).
Other releases are the same shape with different tag/branch constants — see the
inputs on each workflow and the constants block in `docs/SETUP.md`.

## Safety

Every real CBS write (build, tag, promote) is an outward-facing action against
production, and requires your account to hold the corresponding CBS permissions.
The `sig-detect` monitor and all read paths are safe to run anytime.
