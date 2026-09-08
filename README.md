# ns8-ci-actions

Reusable GitHub Actions workflows for NethServer 8 module CI.

`NethServer/ns8-github-actions` covers the pipelines that run on Nethesis
infrastructure. This repository holds the ones that do not need it, so a fork
can run them too.

## Workflows

| Workflow | Description |
|---|---|
| `test-on-qemu.yml` | Boots a Rocky 9 or Debian guest under KVM on the runner, installs the core, creates a single-node cluster and runs the module's test suite against it. Needs no infrastructure and no secret. [Details](docs/test-on-qemu.md) |

## Quick start

Two ways to call `test-on-qemu.yml`. Pick one.

**Build from the checkout, on the pull request.** The image is built from the
caller's checkout, so the pull request tests its own code. This is the mode that
works on a fork: no secret, no published image.

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

**Test the published image, chained on the build.** Same shape as
`test-module.yml` upstream: wait for the publish workflow to succeed, then test
what it published. One build instead of two, and a broken build never reaches
the test.

```yaml
on:
  workflow_run:
    workflows: ["Publish images"]
    types: [completed]

concurrency:
  # github.ref is the default branch under workflow_run, so it cannot key this.
  group: ${{ github.workflow }}-${{ github.event.workflow_run.head_branch || github.ref }}
  cancel-in-progress: true

jobs:
  module:
    if: ${{ github.event.workflow_run.conclusion == 'success' }}
    uses: NethServer/ns8-github-actions/.github/workflows/module-info.yml@v1
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

`concurrency` belongs to the caller in both modes: a reusable workflow cannot
declare one that covers the calling run.

Under `workflow_run` the default context points at the default branch, not at
the branch being tested, hence `repo_ref` and `version_tag`. Two further
consequences are inherent to that trigger and apply to the upstream DigitalOcean
workflow just the same: GitHub runs the definition from the default branch, and
the run raises no check on a pull request. A fork's publish workflow runs in the
fork, so the chain runs and reports there.

## What a module must provide

Three things, all of them already conventions in `ns8-*` repositories:

| Path | Used for |
|---|---|
| `build-images.sh` | builds the module image. Honours `REPOBASE`, and under `CI` reports what it built on the `images` step output |
| `test-module.sh` | takes the node address and the image URL, runs the suite, writes `tests/outputs/` |
| `tests/` | the Robot Framework suite |

Nothing in the workflows knows the module's name. They read it from the `images`
output, which is what makes the same file work for every module.

## Versioning

Callers pin `@v1`, a branch. It moves, but only when a change has been validated
against a real module — `@main` changes on every commit, including one written
mid-debugging. This repository keeps no tags.
