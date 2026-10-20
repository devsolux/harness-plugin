# Harness runtime setup

The plugin and Python runtime are separate packages. Both must be version
**1.0.0**. A cached plugin can be installed without its source repository, so
locate `harness` on the executable path. Do not derive runtime imports or source
paths by walking out of this plugin directory.

## Use a verified wheel

The primary runtime target is Linux and Python 3.12. Use an isolated tool
environment, not the user's application environment. Prepare an absolute path
to a reviewed `harness-1.0.0-py3-none-any.whl` and verify its SHA-256 against the
matching, trusted release checksum before asking for installation approval.

The published wheel and its checksum file are assets of the `v1.0.0` release of
the public plugin repository, https://github.com/devsolux/harness-plugin. With the
user's approval for the download, fetch both into a new, empty directory and
verify the wheel there (on macOS use `shasum -a 256 -c SHA256SUMS`):

```bash
curl -fLO https://github.com/devsolux/harness-plugin/releases/download/v1.0.0/harness-1.0.0-py3-none-any.whl
curl -fLO https://github.com/devsolux/harness-plugin/releases/download/v1.0.0/SHA256SUMS
sha256sum -c SHA256SUMS
```

A failed or missing checksum line stops the setup. Then install the verified wheel:

```bash
uv tool install --python 3.12 /absolute/path/to/harness-1.0.0-py3-none-any.whl
harness --version
harness doctor --json
harness demo --mode parallel --k 1 --json
```

The wheel path is a placeholder. Substitute the actual reviewed artifact.
Do not imply this command was executed, or that a release exists, merely
because it appears here. Installing `uv`, downloading a Python interpreter,
or replacing an existing runtime also needs the user's authorization if it
was not already given. Do not install an unrelated package named `harness`
from a public package index.

Use the wheel's hash for release reproducibility. The planned release `v1.0.0`
is not evidence of publication: a missing release or a download error means the
runtime is unavailable. The source repository is private, so there is no Git
installation alternative. Never replace a missing release with a branch, a
package index lookup, or an unverified wheel. If version `1.0.0` is
unavailable, report that prerequisite.

## Hosts without a Linux sandbox

On macOS, or on Linux without system Bubblewrap, `harness doctor` reports the sandbox
as unavailable and its `sandbox.advice` names `harness-docker build`; `evaluate` and
native `refine` refuse with the same advice. The supported path is the Linux workspace
image: the user builds it with `harness-docker build` from the installed Harness, which
checks the installed files against their RECORD (Docker Desktop or Docker Engine 20.10
or later; the command requests BuildKit; the build downloads the digest-pinned base
image, Ubuntu's own packages and the two pinned host CLIs and publishes nothing;
`make docker-build` builds the same image in a checkout). It must be rebuilt after every upgrade: an image
that does not contain the installed Harness is refused. `harness-docker` then runs the
unchanged `harness` command inside it:

```bash
harness-docker build
harness-docker doctor --json
harness-docker login claude
harness-docker --mount PROJECT_PATH:ro diff RUN_ID --candidate CANDIDATE_ID
harness-docker --mount PROJECT_PATH adopt RUN_ID --candidate CANDIDATE_ID
```

Prefix every command with `harness-docker`; `diff`, `adopt`, `rollback` and `recover`
need `--mount PROJECT_PATH` first (`:ro` suffices for `diff`), because their record,
not the command line, names the project. The container is a Linux host substitute whose daemon is root-equivalent, not
an extra sandbox; Bubblewrap inside it remains the boundary. Never pass
`--privileged`, and report a failed probe inside the image (`sandbox.advice`) as a
condition to fix, never one to bypass.

## Read the result accurately

- Runtime absent or wrong version: stop execution and prepare the exact
  supported installation or replacement for review.
- Missing native CLI, credentials, or sandbox: report the named condition.
  A plugin installation does not provision them or prove authentication.
- Passing offline demo: proves deterministic runtime operation only.
- Model-backed refinement: runs only through `harness refine --allow-model-calls`
  with a reviewed native configuration section, on non-root Linux with a passing
  sandbox probe and a root-owned system install of the host CLI. The provider
  credential is the user's Claude.ai or ChatGPT sign-in stored by `harness login`
  (default) or an API key they export; the Harness gateway keeps it out of the
  sandbox and enforces the request cap. Live native acceptance for this build
  is still pending, so treat real runs as supervised. The dry run makes zero
  model calls and runs no candidate command.
- Existing-candidate evaluation: uses approved deterministic checks and must
  refuse execution if its required isolation cannot be enforced.

Use provider-managed authentication where supported. Never ask the user to
paste a credential into chat, copy a token store into an experiment, or write
secrets into a task file, configuration, plugin, trace, or experience file.

## Upgrade and remove

Check the installed version first. Do not auto-update during an experiment.
Prepare the verified replacement wheel and obtain authorization for the exact
replacement. Keep the previous wheel and trusted checksum for runtime rollback.
Do not assume state written by a newer runtime is readable by an older one.

`uv tool uninstall harness` removes the tool installation when requested.
Plugin removal is a separate host action. Neither operation is a request to
delete project files, adopted instructions, experiment evidence, or backups.
