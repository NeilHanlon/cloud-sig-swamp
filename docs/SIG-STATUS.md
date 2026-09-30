# CentOS Cloud SIG — Epoxy package status & pipeline report

**Snapshot:** 2026-09-29 (generated from a live `sig-detect` run against
`cbs.centos.org`)
**Stream / release:** CentOS Stream 9 · OpenStack **Epoxy** (`cloud9s-openstack-epoxy-*`)
**Author:** Neil Hanlon (`@NeilHanlon`)

> This report is produced mechanically by the swamp `sig-detect` workflow: it
> reads the three promotion tags on CBS (`-candidate` / `-testing` / `-release`),
> the SIG dist-git spec versions, and the upstream OpenStack release metadata,
> then joins them into the actionable queue below. Anyone can reproduce it — see
> **[the replication repo](https://github.com/NeilHanlon/cloud-sig-swamp)** and
> its `docs/SETUP.md`.

---

## 1. Executive summary

The Epoxy package set is **healthy and current**: of **654 packages** tracked,
**581 (89%)** match the latest upstream OpenStack point release and sit correctly
promoted. Only **4 packages are immediately actionable** (1 needs a build, 3 are
built and awaiting a one-command promotion to `-testing`). **68 packages** trail
upstream by a point release — routine maintenance, not breakage. **5 packages**
have stale `sources` (lookaside) metadata that must be corrected before their next
build.

The bigger deliverable this cycle is **tooling**: the promotion pipeline is now
fully automated as a set of swamp workflows, the CBS Koji client is published to
the swamp registry, and the whole thing is packaged into a public repo so other
SIG members can run it. Details in §4–§6.

### Snapshot at a glance

| Bucket | Count | Meaning |
| --- | ---: | --- |
| **Current** | 581 | dist-git == latest upstream, correctly promoted — nothing to do |
| **Behind upstream** | 68 | a newer upstream point release exists (routine bump backlog) |
| **Actionable now** | 4 | needs a build or a promotion **this cycle** (see §2) |
| — unbuilt | 1 | in dist-git / upstream but no build in `-candidate` |
| — promote-testing | 3 | built in `-candidate`, ready to tag into `-testing` |
| — promote-release | 0 | nothing waiting on the release gate |
| **Ahead** | 1 | dist-git newer than the upstream release feed (see note) |
| **Sources stale** | 5 | lookaside `sources` file disagrees with the spec (see §5) |
| **Unknown** | 0 | — |
| **Total tracked** | **654** | every project in `CentOS/cloud/rpms` on the Epoxy branch |

---

## 2. Actionable queue (this cycle)

These are the only packages that need a human decision right now.

> The "Next action" column shows the canonical `cbs`/Koji command the report
> emits, for reference. In this repo you don't run `cbs` by hand — a promotion is
> `swamp workflow run sig-tag-promote` (candidate→testing→release, human-gated)
> and a build is `sig-propose`. See [RUNBOOK.md](RUNBOOK.md) §7.

| Status | Package | Upstream | In candidate | Next action |
| --- | --- | --- | --- | --- |
| **promote-testing** | openstack-keystone | 27.1.0 | `27.0.2-1.el9s` | `cbs tag-build cloud9s-openstack-epoxy-testing openstack-keystone-27.0.2-1.el9s` |
| **promote-testing** | openstack-neutron-fwaas | 22.0.1 | `22.0.1-1.el9s` | `cbs tag-build cloud9s-openstack-epoxy-testing openstack-neutron-fwaas-22.0.1-1.el9s` |
| **promote-testing** | rdo-release | — | `rdo-release-epoxy-1.el9s` | `cbs tag-build cloud9s-openstack-epoxy-testing rdo-release-epoxy-1.el9s` |
| **unbuilt** | openstack-ironic-python-agent-builder | 6.0.1 | *(none)* | build `6.0.1` from dist-git → candidate |

Notes:
- **openstack-keystone** — `27.0.2` is built and sitting in `-candidate`; it is
  ready to promote to `-testing`. Note upstream has since moved to `27.1.0`, so a
  follow-up bump is also owed (it will still be behind after this promotion).
- **openstack-ironic-python-agent-builder** — a `6.0.1` bump scratch-built green
  in August; it just needs a real (draft) build tagged into candidate.

---

## 3. Behind-upstream backlog (68)

A newer upstream point release exists. None are urgent; this is the normal bump
queue. Highlighted larger jumps (major/minor, not just patch): `openstack-kolla`
(19.3→20.6), `python-tooz` (6.3→9.1), `openstack-rally` (4.1→5.1),
`ansible-config_template` (2.1→3.0), `puppet-ceph` (7.0→8.1),
`python-hacking` (7.0→8.1).

<details>
<summary>Full list (dist-git → upstream)</summary>

| Package | dist-git | Upstream |
| --- | --- | --- |
| ansible-config_template | 2.1.1-1 | 3.0.1 |
| openstack-aodh | 20.0.0-1 | 20.0.1 |
| openstack-ceilometer | 1:24.0.0-1 | 24.0.2 |
| openstack-cinder | 1:26.0.0-1 | 26.3.0 |
| openstack-cloudkitty | 22.0.0-1 | 22.1.0 |
| openstack-designate | 1:20.0.0-1 | 20.0.2 |
| openstack-glance | 1:30.0.0-1 | 30.2.0 |
| openstack-heat | 1:24.0.0-1 | 24.1.1 |
| openstack-ironic | 1:29.0.1-1 | 29.1.0 |
| openstack-ironic-python-agent | 10.2.0-1 | 10.2.3 |
| openstack-kolla | 19.3.0-1 | 20.6.0 |
| openstack-magnum | 20.0.0-1 | 20.0.2 |
| openstack-magnum-ui | 16.0.0-1 | 16.1.0 |
| openstack-manila | 1:20.0.0-1 | 20.0.2 |
| openstack-mistral | 20.0.0-1 | 20.1.0 |
| openstack-neutron | 1:26.0.0-1 | 26.0.6 |
| openstack-nova | 1:31.0.0-1 | 31.3.1 |
| openstack-octavia | 16.0.0-1 | 16.1.0 |
| openstack-rally | 4.1.0-1 | 5.1.1 |
| openstack-swift | 2.35.0-1 | 2.35.4 |
| openstack-trove | 1:23.0.0-1 | 23.1.0 |
| openstack-vitrage | 14.0.0-1 | 14.0.1 |
| openstack-watcher | 14.0.0-1 | 14.1.2 |
| openstack-zaqar | 1:20.0.0-1 | 20.1.1 |
| os-apply-config | 14.0.0-1 | 14.0.1 |
| os-collect-config | 14.0.0-1 | 14.0.1 |
| os-refresh-config | 14.0.0-1 | 14.0.1 |
| ovn-bgp-agent | 1:4.0.0-1 | 4.0.1 |
| puppet-ceph | 7.0.0-1 | 8.1.0 |
| python-automaton | 3.2.0-1 | 3.5.0 |
| python-cursive | 0.2.3-2 | 0.3.1 |
| python-debtcollector | 3.0.0-1 | 3.1.0 |
| python-django-horizon | 1:25.3.0-1 | 25.3.2 |
| python-futurist | 3.1.0-1 | 3.5.0 |
| python-hacking | 7.0.0-1 | 8.1.0 |
| python-ironic-inspector-client | 5.3.0-1 | 5.3.1 |
| python-ironicclient | 5.10.0-1 | 5.10.3 |
| python-keystonemiddleware | 10.9.0-1 | 10.9.1 |
| python-manilaclient | 5.4.0-1 | 5.4.1 |
| python-microversion-parse | 2.0.0-1 | 2.1.0 |
| python-networking-generic-switch | 7.5.0-1 | 7.6.0 |
| python-neutron-lib | 3.18.2-1 | 3.18.3 |
| python-neutronclient | 11.4.0-1 | 11.4.1 |
| python-openstackclient | 7.4.0-1 | 7.5.1 |
| python-openstacksdk | 4.4.0-1 | 4.4.1 |
| python-os-brick | 6.11.0-1 | 6.11.1 |
| python-os-client-config | 2.1.0-3 | 2.3.0 |
| python-os-ken | 3.0.1-1 | 3.0.2 |
| python-os-service-types | 1.7.0-4 | 1.9.0 |
| python-os-traits | 3.3.0-1 | 3.9.0 |
| python-oslo-cache | 3.10.1-1 | 3.10.2 |
| python-oslo-config | 2:9.7.1-1 | 9.7.2 |
| python-oslo-db | 17.2.1-1 | 17.2.2 |
| python-oslo-messaging | 16.1.0-1 | 16.1.2 |
| python-oslo-service | 4.1.1-1 | 4.1.2 |
| python-oslotest | 5.0.0-1 | 6.1.1 |
| python-osprofiler | 4.2.0-1 | 4.4.0 |
| python-ovn-octavia-provider | 8.0.0-1 | 8.1.0 |
| python-pycadf | 4.0.1-1 | 4.1.0 |
| python-sphinx-feature-classification | 2.0.0-1 | 2.1.0 |
| python-sushy | 5.5.0-1 | 5.5.1 |
| python-sushy-tools | 2.0.0-1 | 2.2.0 |
| python-swiftclient | 4.7.0-1 | 4.7.1 |
| python-tap-as-a-service | 15.0.0-1 | 15.0.1 |
| python-tooz | 6.3.0-1 | 9.1.0 |
| python-virtualbmc | 3.1.0-1 | 3.3.0 |
| python-whitebox-tests-tempest | 0.0.3-1 | 0.1.0 |
| python-zaqarclient | 3.0.0-1 | 3.0.1 |

</details>

---

## 4. Sources-stale (5) — fix before next build

These packages' lookaside `sources` file references a tarball version that no
longer matches the spec's `Version:`. A build will fail (or silently build the
wrong source) until the `sources` file is regenerated. These are real bugs, not
noise:

| Package | dist-git spec | Upstream | Note |
| --- | --- | --- | --- |
| python-openstackclient | 7.4.0-1 | 7.5.1 | sources pinned well behind spec |
| python-zaqarclient | 3.0.0-1 | 3.0.1 | |
| python-ceilometermiddleware | 3.6.1-1 | 3.6.1 | spec current but sources stale |
| python-gnocchiclient | 7.1.0-1 | 3.1.1 | large sources/version skew |
| python-magnumclient | 4.8.1-1 | 4.8.1 | spec current but sources stale |

### Ahead (1)

- **gnocchi** — dist-git `4.6.0-1` is *newer* than the upstream release feed
  reports (`3.1.4`). Gnocchi is community-released off the OpenStack cadence, so
  this is expected/benign — flagged only so it isn't mistaken for a mis-tag.

---

## 5. Known blocker

**openstack-ironic 29.0.6** is blocked in `%check` by a **buildroot dependency
skew**, not a packaging error in ironic itself:

- The buildroot pairs base `python3-pyasn1-0.6.4` with SIG
  `python3-pysnmp-lextudio-5.0.26-2.el9s`, which still calls
  `pyasn1.compat.octets` — an API removed in pyasn1 ≥ 0.5. ironic's `%check`
  imports pysnmp via the irmc/snmp driver → `ModuleNotFoundError`.
- **Blast radius:** `pysnmp-lextudio` is tagged into `-testing` + `-release` for
  caracal, dalmatian, epoxy, and flamingo — this is a cross-release shared dep.
  Its source lives on `git.centos.org/rpms/pysnmp-lextudio` (the older dist-git,
  not the `CentOS/cloud` GitLab group).
- **Chosen fix:** a downstream compatibility patch to `5.0.26` (`-3.el9s`)
  replacing the removed pyasn1 calls, scratch-validated, then a Neil-gated rebuild
  tagged into the same 4×2 tags. (A same-version rebuild cannot fix it; a version
  bump to a pyasn1-0.6-compatible pysnmp was judged riskier for a shared dep.)
  This is parked pending the patch.

The scratch build **correctly caught** this regression before it could reach a
real build — which is exactly the pre-merge gate the pipeline is meant to provide.

---

## 6. Pipeline & tooling (what was built this cycle)

The promotion process is now automated end-to-end as declarative
[swamp](https://github.com/swamp-club/swamp) workflows over a native Koji client.
No hand-run `koji`/`cbs` commands are required for the common paths.

**Native CBS Koji client** — `@kneel/koji` (published to the swamp registry). A
Swamp-native Koji client that speaks Koji's XML-RPC + header-session protocol and
authenticates with an SSL client cert (the ACO cert) or Kerberos. Reads builds,
tagged builds, and tasks; writes tag/untag, draft promotion, and build submission.
The promotion-matrix report ships inside it.

**Five workflows** (the whole lifecycle, split at the two human gates):

| Workflow | Purpose | Writes? |
| --- | --- | --- |
| `sig-detect` | Read-only monitor → produces the actionable queue in this report | no (cron-safe) |
| `sig-scratch-build` | Validate a revision builds (throwaway) — the pre-merge gate | scratch only |
| `sig-propose` | Build-once **draft** build from a fork branch | draft build |
| `sig-undraft` | On MR merge, promote the draft to a canonical build | promote |
| `sig-tag-promote` | Human-gated tag promotion (candidate→testing→release) | tag |

**Build-once model:** we use Koji **draft** builds (≥1.34) rather than
scratch+rebuild — build one artifact, promote it in place on merge, never build
twice. A scratch build is only used as the cheap pre-merge "does it build?" gate.

**Fork → upstream MR flow:** contributors work on a personal fork
(`fork_project`, published as `@kneel/gitlab-fork` on top of `@webframp/gitlab`)
and open MRs fork→upstream against `c9s-sig-cloud-epoxy`; the branch push is over
SSH (gated on user identity, not a service token).

**Upstream fixes filed** (SIG lookaside is `src.sigs.centos.org`, not the old
`git.centos.org` default):
- `centos-git-common` MR — `get_sources.sh` honors `LOOKASIDE_BASEURL`.
- `sig-guide` docs MR — documents `src.sigs.centos.org` as the SIG lookaside.

---

## 7. Reproduce this

Everything above is reproducible by any SIG member with an ACO cert:

```bash
git clone https://github.com/NeilHanlon/cloud-sig-swamp
cd cloud-sig-swamp
# follow docs/SETUP.md: install swamp, pull extensions, create your koji vault,
# point it at ~/.centos.cert, then:
swamp workflow run sig-detect
```

See **`docs/SETUP.md`** for the full setup (swamp install, vault creds, model
instances) and **`docs/RUNBOOK.md`** for the no-tooling manual recipe
(build / update / promote a package by hand).
