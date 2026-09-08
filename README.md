# ns8-ci-actions

Reusable GitHub Actions workflows for NethServer 8 module CI.

`NethServer/ns8-github-actions` covers the pipelines that run on Nethesis
infrastructure. This repository holds the ones that do not need it, so a fork
can run them too.

## Workflows

| Workflow | Description |
|---|---|
| [`test-on-qemu.yml`](docs/test-on-qemu.md) | Boots a Rocky 9 or Debian guest under KVM on the runner, installs the core, creates a single-node cluster and runs the module's test suite against it. Needs no infrastructure and no secret. |

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
