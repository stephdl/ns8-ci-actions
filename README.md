# ns8-ci-actions

Reusable GitHub Actions workflows for NethServer 8 module CI.

`NethServer/ns8-github-actions` covers the pipelines that run on Nethesis
infrastructure. This repository holds the ones that do not need it, so a fork
can run them too.

## Workflows

| Workflow | Description |
|---|---|
| [`test-module.yml`](docs/test-module.md) | What a module should call. Drop-in replacement for the wrapper of the same name in `ns8-github-actions`: gathers the module information, decides whether the UI tests are worth running, and fans out over distributions and scenarios. |
| [`test-on-qemu.yml`](docs/test-on-qemu.md) | One leg. Boots a Rocky 9 or Debian guest under KVM on the runner, installs the core, creates a single-node cluster and runs the module's test suite against it. Needs no infrastructure and no secret. |

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
Podman container, against a live NS8 cluster.

It is vendored verbatim from `NethServer/ns8-github-actions@v1`, so a suite sees
the same runner whether it is tested on DigitalOcean or on the QEMU node.
Re-sync it rather than patching it here.

A module calling `test-on-qemu.yml` with `script: ""` uses this copy and can
**delete its own `test-module.sh`**: one less file to keep in step with the
others. The default keeps the module's script, so nothing moves under the
modules that still ship one.

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
| `SSH_KEYFILE` | `~/.ssh/id_ecdsa` | Private key that reaches the leader node |
| `RUN_UI_TESTS` | _(unset)_ | `true` runs the cases tagged `ui`, in the Playwright image. Anything else excludes them and uses a slim Python image |

The workflow sets `RUN_UI_TESTS` from its own `run_ui_tests` input, and
publishes the images found under `tests/outputs/` as a comment on the pull
request of the ref under test.

That comment needs a token allowed to write pull requests, and a called
workflow cannot ask for more than its caller holds, so the calling job grants
it. [The wrapper documentation](docs/test-module.md#the-pull-request-comment-and-its-token)
says why the grant sits on the job rather than on the repository, and
[the workflow documentation](docs/test-on-qemu.md#interface-screenshots) carries
the caller file to copy, with and without the screenshots.

## Versioning

Callers pin `@v1`. It is the default branch and the one every change lands on,
through a pull request validated against a real module, so it moves but never
unreviewed.

`main` is left where the repository started and is not maintained. There are no
tags.
