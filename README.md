# ns8-ci-actions

Reusable GitHub Actions workflows to test a NethServer 8 module on a node the
runner builds for itself.

`NethServer/ns8-github-actions` tests modules on DigitalOcean droplets, which
need the Nethesis account, its `NS8-CI` project and the `ci.nethserver.net`
domain. A fork reaches none of them. `test-on-qemu.yml` boots a cloud image
under KVM on a public GitHub runner instead, so it runs anywhere — including on
a pull request opened from a fork, and with no secret at all.

## Usage

Two modes. Pick one.

### Build from the checkout, on the pull request

The module image is built from the caller's checkout and served from a registry
that lives and dies with the job, so the pull request tests its own code. This
is the mode that works on a fork: it needs no secret and no published image.

```yaml
name: "Test module on QEMU"

on:
  pull_request:
    branches: [main]

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  test:
    strategy:
      fail-fast: false
      matrix:
        distro: [rocky9, debian13]
    uses: stephdl/ns8-ci-actions/.github/workflows/test-on-qemu.yml@v1
    with:
      distro: ${{ matrix.distro }}
```

### Test the published image, chained on the build

Same shape as `NethServer/ns8-github-actions`'s `test-module.yml`: wait for
`Publish images` to succeed, then test what it published. One build instead of
two, and a broken build never reaches the test.

```yaml
on:
  workflow_run:
    workflows: ["Publish images"]
    types: [completed]

concurrency:
  # github.ref is the default branch under workflow_run, so keying the group on
  # it would put every branch in one group and have them cancel each other.
  group: ${{ github.workflow }}-${{ github.event.workflow_run.head_branch || github.ref }}
  cancel-in-progress: true

jobs:
  module:
    if: ${{ github.event.workflow_run.conclusion == 'success' }}
    uses: NethServer/ns8-github-actions/.github/workflows/module-info.yml@main
  test:
    needs: module
    strategy:
      fail-fast: false
      matrix:
        distro: [rocky9, debian13]
    uses: stephdl/ns8-ci-actions/.github/workflows/test-on-qemu.yml@v1
    with:
      distro: ${{ matrix.distro }}
      image_url: ${{ needs.module.outputs.image }}
      repo_ref: ${{ needs.module.outputs.sha }}
      version_tag: ${{ needs.module.outputs.tag }}
```

Under `workflow_run` the default context points at the default branch, not at
the branch being tested, so `repo_ref` and `version_tag` have to be passed. Two
further consequences are inherent to `workflow_run` and apply to the upstream
DigitalOcean workflow just the same: the definition GitHub runs is the one on
the default branch, and the run does not appear as a check on the pull request.

A fork's `Publish images` runs in the fork, so the chain runs there too — but it
raises nothing on the upstream pull request. Use the first mode if you want a
check on incoming pull requests.

`concurrency` belongs to the caller in both modes: a reusable workflow cannot
declare one that covers the calling run.

Pin a tag, not `@main`. On `@main`, a change here silently changes the meaning of
every caller's green tick, with no commit in their repository to point at.

## What the workflow expects from the module

Three things, all of them already conventions in `ns8-*` repositories:

| Path | Used for |
|---|---|
| `build-images.sh` | builds the module image. Honours `REPOBASE`, and under `CI` reports what it built on the `images` step output |
| `test-module.sh` | takes the node address and the image URL, runs the suite, writes `tests/outputs/` |
| `tests/` | the Robot Framework suite |

Nothing in the workflow knows the module's name. It reads it from the `images`
output, which is what makes the same file work for every module.

## How it works

Three environments nested inside one another. Everything else follows from that.

```
┌─ GitHub runner (ubuntu-24.04, throwaway Azure VM) ─────────────────┐
│                                                                     │
│  buildah ──build-images.sh──┐                                       │
│                             ▼                                       │
│                    ci-registry (registry:2)  192.168.77.1:5000      │
│                             ▲                                       │
│  ns8br0 192.168.77.1/24 ────┼──── MASQUERADE ──> internet           │
│    │                        │                                       │
│    │ ns8tap0                │ pull                                  │
│    ▼                        │                                       │
│  ┌─ QEMU/KVM guest   192.168.77.10 ─────────────────────────┐      │
│  │                                                           │      │
│  │  ns8-core ── traefik :80 :443 ──> module pod              │      │
│  │                                                           │      │
│  └───────────────────────────────────────────────────────────┘      │
│    ▲                                                                 │
│    │ ssh root@192.168.77.10                                          │
│  ┌─┴─ test container (netns=host) ─┐                                │
│  │  test-module.sh → robot          │                                │
│  └──────────────────────────────────┘                                │
└─────────────────────────────────────────────────────────────────────┘
```

**The runner** is a fresh VM, destroyed when the job ends. The bridge, the
registry and the built image die with it — which is why the addresses can be
hardcoded.

**The guest** is the real NS8 node. It runs under KVM inside the runner, hence
the `/dev/kvm` check: without hardware acceleration QEMU emulates, and the job
times out instead of failing.

**The test container** runs the suite. It tests nothing itself; it opens an SSH
session to the guest and runs commands there.

### The three phases

**1. Build the image, on the runner.** `build-images.sh` builds from the
caller's checkout with `REPOBASE` pointed at the local registry, and reports
what it built on its `images` output. That output is the entire contract: the
workflow never needs to know the module's name.

Why not ghcr? On a pull request from a fork `GITHUB_TOKEN` has no
`packages: write`. And the point is to test the code of the pull request, not an
image published before it.

**2. Bring the node up.** The cloud-init seed gives the guest its static
address, the runner's SSH key, and a `registries.conf.d` entry marking the
registry `insecure = true` — podman refuses a plain-HTTP registry otherwise.
Then, over SSH: `install.sh` from `ns8-core`, then `create-cluster`. At this
point the guest is a working single-node NS8 cluster with no module on it.

**3. Run the suite.** `test-module.sh` starts the test container and hands it
the node address and the image URL. Robot connects over SSH and drives the node:
`add-module` from the throwaway registry, then whatever the module's own suite
asserts, then `remove-module`.

## Inputs

| Input | Default | |
|---|---|---|
| `distro` | `rocky9` | `rocky9`, `debian12` or `debian13` |
| `cloud_image_url` | | overrides the URL implied by `distro` |
| `corebranch` | `ns8-stable` | branch or tag of `ns8-core` |
| `coremodules` | | extra module URLs passed to `install.sh` |
| `image_url` | | test this image instead of building one |
| `script` | `test-module.sh` | test entry point |
| `path` | | subdirectory holding the module |
| `repo_ref` | `github.sha` | caller ref to check out |
| `runs_on` | `ubuntu-24.04` | must provide `/dev/kvm` |
| `vm_mem` | `8192` | guest memory, MiB. The runner has 15360 and needs some for itself, so 12288 is the practical ceiling |
| `vm_cpus` | `4` | guest vCPUs |
| `disk_size` | `30G` | guest disk after resize |
| `timeout_minutes` | `60` | |
| `version_tag` | branch under test | names the image tag and the artifact. `workflow_run` callers must pass it |
| `debug_shell` | `false` | tmate shell when the suite fails |

`vm_mem` is the input worth setting: the runner has 15 GiB and uses about 1.5 of
them, so 8 leaves room, but a module starting several JVMs wants more.

## Secrets

Both optional.

| Secret | |
|---|---|
| `dockerhub_user` | raises the Docker Hub pull limit above the 100 per 6h that anonymous runners share |
| `dockerhub_token` | |

Pass them only if the module pulls enough Docker Hub images to risk a 429.

## The cloud image cache

Downloading the guest image dominated the Rocky job: 93 s, 263 s then 290 s over
three runs, against 5 s for Debian, whose URL redirects to a CDN mirror. The
image is now cached, keyed on the checksum the distribution publishes next to
it.

That key does the expiry by itself. A new point release changes the checksum,
so the key changes, so the cache misses and the image is refetched. A hit is the
upstream image whatever its age, which means "too old" stops being a state the
cache can be in. GitHub deletes entries unused for 7 days and evicts by
least-recently-used past 10 GB per repository, so nothing has to be pruned by
hand. About 620 MB per distribution.

The key also carries a digest of the image URL, because `cloud_image_url` can
aim two callers with the same `distro` at different images.

The restored file is checked against that same checksum before use: a mismatch
warns, deletes the file and refetches. A fresh download that fails is retried
once against a freshly resolved checksum — a distribution republishing into
`latest/` can leave the sum and the bytes a moment apart — and fatal after that.

Two limits worth knowing. When no checksum is reachable beside the image the key
falls back to the calendar week and **nothing verifies the bytes**; the run
raises a warning saying so. And `actions/cache/save` cannot overwrite a key that
already exists, so an entry that fails its checksum survives until GitHub evicts
it, and every run until then refetches. Only the cache is lost: the image is
verified before it boots either way.

Two details worth knowing if you read the workflow. The save is explicit rather
than left to `actions/cache`, whose post-job step would run after the guest has
written gigabytes into the disk. And the guest boots on a copy, so the cached
base stays exactly what the checksum says.

## What a run produces

The image is tagged with the branch under test, so `add-module` in the Robot log
reads `192.168.77.1:5000/pihole:feat-8678` rather than an anonymous tag, and the
run summary names it. The artifact carries the same slug:

```
test-outputs-debian13-feat-8678
test-outputs-rocky9-feat-8678
```

`diag/module-version.txt` inside it records the image URL, the digests podman
resolved on the node, and `list-installed-modules`.

## When it fails

The artifact holds the Robot `log.html` and `report.html`, plus a `diag/`
directory with the QEMU serial console, the guest journal, its
`/etc/os-release`, listening sockets, and `podman ps` for every module user. That is usually enough to find the cause without opening a shell.
`debug_shell: true` gives you a tmate session when it is not.
