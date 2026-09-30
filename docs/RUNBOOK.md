# RUNBOOK — CentOS Cloud SIG packaging with our swamp tooling

Operator runbook for **detecting**, building, updating, and promoting Cloud SIG
(RDO) packages on **CBS** (`cbs.centos.org`). Everything lives in this repo —
models, all five workflows, and the report.
Copy-paste recipes; no LLM required. First-time setup (install swamp, pull the
extensions, add your ACO cert, create your two vaults — the model instances are
already committed) is in [SETUP.md](SETUP.md); this runbook assumes that is done. Every command below was run (or, for the
build/promote steps that are maintainer-gated, `swamp workflow validate`-clean) as
written.

> Every real CBS write (build, tag, promote) is an outward-facing action against
> production. Read the command before you run it. The **detect** and **read**
> sections are 100% read-only and safe to run anytime.

---

## 0. The pieces (all in your clone, referred to as `$SIG`)

| Thing | What | Run by |
| --- | --- | --- |
| `cbs-koji` | `@kneel/koji` instance; SSL auth via `~/.centos.cert` | `swamp model method run cbs-koji …` |
| `cloud-sig-epoxy` | `@kneel/sig-distgit` — scans specs + `sources` state | `swamp model method run cloud-sig-epoxy scan` |
| `openstack-epoxy` | `@kneel/openstack-releases` — upstream release snapshot | `swamp model method run openstack-epoxy snapshot` |
| `sig-promote` report | joins the 3 tags + specs + upstream → actionable queue | attached to `sig-detect` |
| **`sig-detect`** | monitor: latest_builds ×3 + scan + snapshot → report. **Read-only.** | `swamp workflow run sig-detect` |
| **`sig-scratch-build`** | validate-only: login → submit_build(scratch) → wait → assert CLOSED | `swamp workflow run sig-scratch-build` |
| **`sig-propose`** | build-once: login → submit_build(**draft**) → wait → assert CLOSED | `swamp workflow run sig-propose` |
| **`sig-undraft`** | on merge: assert MR merged → promote_build (draft→canonical) | `swamp workflow run sig-undraft` |
| **`sig-tag-promote`** | human-gated tag step: login → approve → tag_build → wait | `swamp workflow run sig-tag-promote` |
| local dist-git mirror | all SIG rpms, `c9s-sig-cloud-epoxy` branch | `~/centos-rpms/<pkg>` |

**Constants for the Epoxy / c9s target:**
- build target: `cloud9s-openstack-epoxy-el9s` (dest tag = `-candidate`)
- promotion tags: `cloud9s-openstack-epoxy-{candidate,testing,release}`
- SIG lookaside: **`https://src.sigs.centos.org`** (NOT `sources.stream.centos.org`)

Run all swamp commands from the root of your clone. The recipes below refer to
that root as `$SIG` — set it once for wherever you cloned the repo:

```bash
export SIG=~/path/to/your/cloud-sig-swamp   # wherever you cloned it
```

All four model types (`@kneel/koji`, `@kneel/sig-distgit`,
`@kneel/openstack-releases`, `@webframp/gitlab`) are published to the swamp
registry and pulled during setup (see [SETUP.md](SETUP.md)) — no local extension
sources required. Auth is your own `~/.centos.cert` (ACO cert), same CBS hub.

---

## 1. Log in to CBS

```bash
cd "$SIG"
swamp model method run cbs-koji login                          # SSL, uses ~/.centos.cert
swamp data get cbs-koji session --json | jq .content.callnum   # sanity: a number
```
Session lasts ~24h. Re-run `login` if a call faults with `session … has expired`.
The workflows all `login` as their first step, so you rarely call this directly.

## 2. Read state (all read-only)

```bash
# latest build of a package in a tag
swamp model method run cbs-koji latest_builds \
  --input tag=cloud9s-openstack-epoxy-candidate --input package=openstack-keystone
swamp data get cbs-koji tagged --json | jq '.content.builds | map(.nvr)'

# one build + the tags it's in
swamp model method run cbs-koji get_build --input build=openstack-keystone-27.0.2-1.el9s
swamp data get cbs-koji build --json | jq '{nvr:.content.build.nvr, tags:(.content.tags|map(.name))}'

# a build/tag task (state 2 == CLOSED, 5 == FAILED)
swamp model method run cbs-koji task_info --input task_id=5839190
swamp data get cbs-koji task --json | jq '.content.task | {id, method, state}'
# hub UI: https://cbs.centos.org/koji/taskinfo?taskID=5839190
```
Note: `swamp data get <model> <spec> --json` puts the payload under **`.content`**.

## 3. Detect — "what should I work on?"  ← start here

`sig-detect` gathers the three inputs (latest builds in each promotion tag, SIG
dist-git spec versions, upstream OpenStack releases) and runs the `sig-promote`
report over them. **Read-only — no builds, tags, or MRs.** ~2 min (the dist-git
scan is the slow part).

```bash
cd "$SIG"
swamp workflow run sig-detect        # defaults = the c9s epoxy tag triplet
```
The console prints the full matrix (actionable rows sorted first). To re-read the
report the run produced, ask swamp for it — don't dig through `.swamp/`:

```bash
# Human-readable: renders the Action queue + sources-staging tables directly (no jq)
swamp report get "@kneel/sig-distgit/sig-promote" --workflow sig-detect --markdown

# Machine-readable: payload is under .json — jq on --json output is fine (rule 3), just
# never read the .swamp/ files by hand.
swamp report get "@kneel/sig-distgit/sig-promote" --workflow sig-detect --json \
  | jq '.json.summary'                                # counts: actionable / unbuilt / promoteTesting / …
swamp report get "@kneel/sig-distgit/sig-promote" --workflow sig-detect --json \
  | jq -r '.json.rows[] | select(.status=="unbuilt" or .status=="promote-testing" or .status=="promote-release")
           | "\(.status)\t\(.package)\t\(.candidate // "-")"'
```
Statuses: **unbuilt** (spec/upstream newer than candidate → build owed),
**promote-testing** / **promote-release** (a built NVR ready to tag up),
**behind-upstream** (needs a dist-git bump first — see §5), **ahead**, **current**.
`summary.sourcesStale`/`sourcesPartial` flag bump bugs (a spec bumped without
re-staging `sources` — would fail `buildSRPMFromSCM`; see §7).

Override the tags for another target (e.g. el10s):
```bash
swamp workflow run sig-detect \
  --input candidateTag=cloud10s-openstack-epoxy-candidate \
  --input testingTag=cloud10s-openstack-epoxy-testing \
  --input releaseTag=cloud10s-openstack-epoxy-release
```

## 4. Scratch-build a package (net-zero dry run)

A scratch build leaves no retained artifact and does not auto-tag — it just
answers "does this revision build?". Use it to validate a bump/SRPM before you
commit to the real (draft) build. The workflow asserts the task finished CLOSED,
so a red build fails the workflow (the PR-gate: don't propose what won't build).

```bash
cd "$SIG"
# from a local SRPM (proves it builds; also the RDO-native upload path):
swamp workflow run sig-scratch-build \
  --input source=/path/to/openstack-foo-1.2.3-1.el9s.src.rpm \
  --input target=cloud9s-openstack-epoxy-el9s

# or from an SCM ref:
swamp workflow run sig-scratch-build \
  --input source='git+https://gitlab.com/CentOS/cloud/rpms/openstack-foo.git#<sha>' \
  --input target=cloud9s-openstack-epoxy-el9s
```
Blocks on `wait_task` (may exceed a 2-min shell timeout — the build continues
server-side; re-check with `task_info`, §2). `timeout`/`interval` inputs default
to 1800/20s.

## 5. Update a package to a new upstream version  ← the bump recipe

Worked example: `python-keystonemiddleware` 10.9.0 → 10.9.1. Substitute names.

```bash
K=~/centos-rpms/python-keystonemiddleware
SPEC=$K/python-keystonemiddleware.spec
V=10.9.1
cd "$K"
git checkout c9s-sig-cloud-epoxy && git pull
git checkout -b bump-keystonemiddleware-$V
```

**5a. Bump the spec** (Version + a %changelog entry). `Source0` usually uses
`%{version}`, so the tarball name follows automatically:
```bash
sed -i "s/^Version:\(\s*\).*/Version:\1$V/" "$SPEC"
sed -i "/^%changelog/a\\
* $(date +'%a %b %d %Y') Your Name <you@example.com> - $V-1\\
- Update to $V\\
" "$SPEC"
```

**5b. Fetch the binary sources.** RDO specs carry three URL `Source`s — the
tarball, its `.asc`, and the OpenStack gpg key:
```bash
SVC=keystonemiddleware   # the upstream project (%{sname}/%{service})
curl -sSLO "https://tarballs.openstack.org/$SVC/$SVC-$V.tar.gz"
curl -sSLO "https://tarballs.openstack.org/$SVC/$SVC-$V.tar.gz.asc"
```

**5c. Resolve the CORRECT gpg key** — see §6. A bump almost always needs
`%{sources_gpg_sign}` updated to the OpenStack release key active *when the
tarball was signed*. Set it and fetch that key file:
```bash
KEY=0x<lowercase-primary-fpr>          # from §6
sed -i "s/^%global sources_gpg_sign .*/%global sources_gpg_sign $KEY/" "$SPEC"
curl -sSLO "https://releases.openstack.org/_static/$KEY.txt"
gpg --verify "$SVC-$V.tar.gz.asc" "$SVC-$V.tar.gz"    # must say: Good signature
```

**5d. Build the SRPM locally:**
```bash
rpmbuild -bs --define "_topdir /tmp/rpmbuild" --define "_sourcedir $K" \
  --define "_srcrpmdir /tmp" --define "dist .el9s" "$SPEC"
```

**5e. Scratch-build it** (§4). Fails at `%prep gpgverify` → key is wrong (§6).
Fails in `buildArch` → a real packaging problem; fix the spec.

**5f. Upload the three sources to the lookaside** (idempotent; §7), then write
the `sources` metadata + gitignore the blobs:
```bash
rm -f sources
sha512sum --tag "$SVC-$V.tar.gz" "$SVC-$V.tar.gz.asc" "$KEY.txt" > sources
for f in "$SVC-$V.tar.gz" "$SVC-$V.tar.gz.asc" "$KEY.txt"; do
  grep -qxF "$f" .gitignore || echo "$f" >> .gitignore
done
```
> The tarball/.asc/key are **binary sources — they go in the lookaside, NEVER in
> git.** Only `sources` (the SHA512 metadata) and `.gitignore` are committed.

**5g. Commit + push to your FORK, open the MR fork→upstream** (git push-options,
SSH remote, no token — the GitLab GraphQL MR API can't do cross-project MRs, so
the push-option is how the fork→upstream MR is opened). **Policy: push bump
branches to your fork, never to the upstream ref list.** Only after a green
scratch build:
```bash
# one-time: add your fork as a remote (replace <you> with your gitlab namespace)
git remote add fork git@gitlab.com:<you>/rpms/python-keystonemiddleware.git 2>/dev/null || true
git add python-keystonemiddleware.spec sources .gitignore
git commit -m "Update to keystonemiddleware $V"
git push fork bump-keystonemiddleware-$V \
  -o merge_request.create \
  -o merge_request.target=c9s-sig-cloud-epoxy \
  -o merge_request.title="Update to keystonemiddleware $V" \
  -o merge_request.description="Scratch build: https://cbs.centos.org/koji/taskinfo?taskID=<id>"
# → prints the MR URL
```

## 6. Resolve the OpenStack signing key (the tricky bit)

OpenStack signs every release with the key **active at signing time**, regardless
of series. A bump to a newer point release is usually signed by a newer key than
the spec pins → `%prep gpgverify` fails. **Never disable verification**
(`sources_gpg 0`); fix the key.

```bash
# 1. who signed the tarball? (note the EDDSA key id)
gpg --verify keystonemiddleware-10.9.1.tar.gz.asc keystonemiddleware-10.9.1.tar.gz
#   → "using EDDSA key C268329D...486FD056" ; "Can't check signature: No public key"

# 2. get that key from the keyserver and find its PRIMARY fingerprint
curl -sSL "https://keys.openpgp.org/vks/v1/by-fingerprint/<SIGNING-SUBKEY-FPR>" -o /tmp/signer.asc
gpg --show-keys --with-subkey-fingerprints /tmp/signer.asc
#   pub = the PRIMARY (what the spec references); uid = e.g. "OpenStack Infra (2026.1/Gazpacho Cycle)"

# 3. the published key file is the PRIMARY fingerprint, LOWERCASE, 0x-prefixed:
KEY=0x$(echo <PRIMARY-FPR> | tr 'A-Z' 'a-z')
curl -s -o /dev/null -w '%{http_code}\n' -I "https://releases.openstack.org/_static/$KEY.txt"   # want 200
```
Gotcha: `_static/` filenames are **lowercase** — an uppercase fingerprint 404s.
Set `%{sources_gpg_sign} = $KEY` and re-verify (`Good signature`).

## 7. Lookaside upload (SIG)

The SIG lookaside is `src.sigs.centos.org`; the tool defaults to `git.centos.org`
(which mirrors it — do **not** override to `sources.stream.centos.org`, that's the
Stream distro and 403s). Auth is the ACO `~/.centos.cert`. The tool isn't always
current on the host, so run it in the Fedora toolbox:

```bash
K=~/centos-rpms/<pkg>
podman run --rm --security-opt label=disable \
  -v "$HOME/.centos.cert":/root/.centos.cert:ro -v "$K":/work \
  registry.fedoraproject.org/fedora-toolbox:42 bash -lc '
    dnf install -y -q centos-packager
    cd /work
    for f in <tarball> <asc> <key.txt>; do centos-lookaside-upload-sig -f "$f" -n <pkg>; done'
# verify (200): https://src.sigs.centos.org/sources/<pkg>/<file>/sha512/<sha512>/<file>
```
Uploads are content-addressed + idempotent (additive, never overwrite).

## 8. Build-once promotion pipeline (propose → merge → undraft → promote)

The build-once model: build a **draft** ONCE, then promote that same artifact on
merge (no rebuild). A draft is retained and tagged into candidate as a draft NVR;
on merge it's undrafted to the canonical build. (Scratch = throwaway validation;
draft = the keeper. Don't scratch-then-real — that builds twice.)

**8a. `sig-propose` — the kept draft build.** Run it after §5's branch is pushed
+ MR opened. Builds once and keeps it.
```bash
cd "$SIG"
swamp workflow run sig-propose \
  --input source='git+https://gitlab.com/<you>/rpms/<pkg>.git#<sha>' \
  --input target=cloud9s-openstack-epoxy-el9s
```
Asserts the build finished CLOSED. The kept build's NVR carries a draft marker
(`…-1,draft_<buildid>.el9s`) — find it for the next step:
```bash
swamp model method run cbs-koji list_builds --input prefix=<pkg> --input state=1
swamp data get cbs-koji builds --json | jq -r '.content.builds[] | select(.draft==true) | .nvr'
```

**8b. (human) The SIG reviews and merges the MR.** No tooling — that's their gate.

**8c. `sig-undraft` — promote the kept draft to canonical on merge.** Verifies the
MR actually merged first (fails fast if not, so you can retry), then undrafts. The
canonical build lands in candidate; no rebuild.
```bash
swamp workflow run sig-undraft \
  --input project=CentOS/cloud/rpms/<pkg> \
  --input iid=<MR-iid> \
  --input draftBuild='<pkg>-<v>-1,draft_<buildid>.el9s'
```

**8d. `sig-tag-promote` — human-gated tag up the ladder** (candidate→testing, then
testing→release). Reusable; pick `toTag`. It pauses at a `manual_approval` step —
**this is your review gate.**
```bash
# candidate → testing
swamp workflow run sig-tag-promote \
  --input build=<nvr> \
  --input toTag=cloud9s-openstack-epoxy-testing
# it pauses; see what's pending, then approve (step name is 'approve'):
swamp workflow approvals
swamp workflow approve sig-tag-promote approve --reason "reviewed, promoting to testing"
#   (use --run <run-id> if more than one run is pending)

# later, testing → release (release triggers hub-side sign-and-push to mirrors):
swamp workflow run sig-tag-promote --input build=<nvr> --input toTag=cloud9s-openstack-epoxy-release
swamp workflow approve sig-tag-promote approve --reason "sign-off, releasing"
```
Reject instead with `swamp workflow reject sig-tag-promote approve`. Tagging into
`-release` is the delivery step — the hub signs with the SIG key and pushes to
mirrors. Verify before/after with `get_build` (§2).

### Low-level manual promotion (when you don't want the gate)
```bash
swamp model method run cbs-koji tag_build \
  --input tag=cloud9s-openstack-epoxy-testing --input build=<nvr>
swamp model method run cbs-koji wait_task --input task_id=<tag-task-id>   # tag_build is async
# untag_build is synchronous (no wait). A net-zero test: tag then untag.
```

---

## Gotchas cheat-sheet

- **`.content`** — where `swamp data get --json` payloads live (workflow CEL uses `.attributes`).
- **Run swamp commands from your clone root (`$SIG`)** — all models + workflows are native here.
- **`sig-detect` is read-only** — run it anytime; it's the "what's actionable" entry point.
- **Draft, not scratch, for keeps** — `sig-propose` (draft) builds once and is promotable; scratch is throwaway validation. You can't tag a scratch build.
- **Push bumps to your FORK**, MR fork→upstream via `git push -o merge_request.create` over the **SSH** remote — never push branches to the upstream repo. (GitLab's GraphQL MR API can't do cross-project MRs; the push-option is the mechanism.)
- **Lookaside = `src.sigs.centos.org`**; tool default `git.centos.org` mirrors it; `sources.stream.centos.org` 403s (Stream distro).
- **`_static/` key URLs are lowercase**; gpg key = the OpenStack cycle active at *signing time* — a bump usually needs a new `%{sources_gpg_sign}`. Don't skip verification.
- **Binary sources never go in git** — lookaside + the `sources` SHA512 file only.
- **task state**: `2` = CLOSED (good), `5` = FAILED. `wait_task` snapshots add `stateName`/`finished`.
- **Workflow `dependsOn`** entries are objects: `- step: <name>\n  condition: {type: succeeded}`, not bare strings.
- **`zsh` noclobber** blocks `> file` when it exists — use `>|` or `rm -f` first.

## Status of the tooling

- **Live-proven, read-only:** `sig-detect` (654 pkgs, actionable queue), and every §1–§3 read.
- **Validated (`swamp workflow validate` clean); first live CBS write is yours:**
  `sig-propose`, `sig-undraft`, `sig-tag-promote`. A draft build is the first
  thing to confirm live (that CBS's draft policy authorizes your user — the hub
  capability is already proven).
- **Only automation left:** a cron wrapper around `swamp workflow run sig-detect`
  (via the `schedule` skill) so it tells you unprompted.

Design note: there is intentionally **no** `koji.promotion_matrix` method — a
promotion ladder isn't a koji hub resource, so the 3-tag join lives in the
`sig-promote` report (which self-joins the `latest_builds` snapshots), keeping
`@kneel/koji` a generic build-system model.
