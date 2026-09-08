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

```
runner (ubuntu-24.04, throwaway VM)
├── buildah ──build-images.sh──> 192.168.77.1:5000/<module>:ci  (registry:2)
├── bridge ns8br0 192.168.77.1/24 + tap + MASQUERADE
├── qemu -accel kvm
│   └── guest @ 192.168.77.10 → install.sh → create-cluster
│       └── add-module 192.168.77.1:5000/<module>:ci
└── podman (host netns) → test-module.sh --ssh--> 192.168.77.10:22
```

The module image is built from the caller's checkout and served from a registry
that lives and dies with the job, so a pull request tests its own code rather
than an image published earlier. Pushing to ghcr instead would not work from a
fork: `GITHUB_TOKEN` has no `packages: write` there.

The addresses are hardcoded on purpose. Every job gets its own runner VM, so the
bridge, the subnet and the registry port are private to it. On a *self-hosted*
runner two concurrent jobs would collide, and nothing here tears the bridge down.

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
