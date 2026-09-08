# ns8-ci-actions

Reusable GitHub Actions workflows to test a NethServer 8 module on a node the
runner builds for itself.

`NethServer/ns8-github-actions` tests modules on DigitalOcean droplets, which
need the Nethesis account, its `NS8-CI` project and the `ci.nethserver.net`
domain. A fork reaches none of them. `test-on-qemu.yml` boots a cloud image
under KVM on a public GitHub runner instead, so it runs anywhere — including on
a pull request opened from a fork, and with no secret at all.

## Usage

Add this to a module repository:

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

`concurrency` belongs to the caller: a reusable workflow cannot declare one that
covers the calling run.

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
| `vm_mem` | `6144` | guest memory, MiB |
| `vm_cpus` | `4` | guest vCPUs |
| `disk_size` | `30G` | guest disk after resize |
| `timeout_minutes` | `60` | |
| `debug_shell` | `false` | tmate shell when the suite fails |

`vm_mem` is the input worth setting: a DNS cache is happy with 6 GB, a module
starting several JVMs is not.

## Secrets

Both optional.

| Secret | |
|---|---|
| `dockerhub_user` | raises the Docker Hub pull limit above the 100 per 6h that anonymous runners share |
| `dockerhub_token` | |

Pass them only if the module pulls enough Docker Hub images to risk a 429.

## When it fails

Every run uploads `test-outputs-<distro>`: the Robot `log.html` and
`report.html`, plus a `diag/` directory with the QEMU serial console, the guest
journal, its `/etc/os-release`, listening sockets, and `podman ps` for every
module user. That is usually enough to find the cause without opening a shell.
`debug_shell: true` gives you a tmate session when it is not.
