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
| `test-module.sh` | takes the node address and the image URL, runs the suite, writes `tests/outputs/`. Optional: pass `script: ""` and the shared runner below is used instead |
| `tests/` | the Robot Framework suite |

Nothing in the workflows knows the module's name. They read it from the `images`
output, which is what makes the same file work for every module.

No secret is needed.

## The shared test runner

`scripts/test-module.sh` runs the `tests/` directory of a module inside a
Podman container, against a live NS8 cluster. A module that calls
`test-on-qemu.yml` with `script: ""` uses it and needs no script of its own; the
default keeps the module's own `test-module.sh`, so nothing changes for the
modules that ship one.

Install it once to run the suite from your workstation:

```bash
curl -o /tmp/run-ns8-tests https://raw.githubusercontent.com/stephdl/ns8-ci-actions/v1/scripts/test-module.sh
install -m 0755 -Z /tmp/run-ns8-tests ~/.local/bin
```

Then, from the module directory:

```bash
run-ns8-tests <LEADER_NODE> <IMAGE_URL> [robot options...]
```

| Variable | Default | Description |
|---|---|---|
| `SSH_KEYFILE` | `~/.ssh/id_rsa` | Private key that reaches the leader node |
| `RUN_UI_TESTS` | _(unset)_ | `true` runs the cases tagged `ui`, in the Playwright image. Anything else excludes them and uses a slim Python image |

The workflow sets `RUN_UI_TESTS` from its own `run_ui_tests` input, and
publishes the images found under `tests/outputs/` as a comment on the pull
request of the ref under test.

That comment needs a token allowed to write pull requests, and a called
workflow cannot ask for more than its caller holds. Grant it on the calling
job, otherwise the step is skipped:

```yaml
  test:
    permissions:
      contents: read
      pull-requests: write
    uses: stephdl/ns8-ci-actions/.github/workflows/test-on-qemu.yml@v1
```

## Versioning

Callers pin `@v1`. It is the default branch and the one every change lands on,
through a pull request validated against a real module, so it moves but never
unreviewed.

`main` is left where the repository started and is not maintained. There are no
tags.
