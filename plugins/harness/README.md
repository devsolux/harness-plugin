# Harness plugin for Claude Code and Codex

![Harness](assets/logo.svg)

Harness measures whether a change to a coding agent's instructions (`CLAUDE.md`,
`AGENTS.md` and Skills) makes Claude Code or Codex reliably pass a project's own checks
that it failed before. The unchanged baseline and each candidate run the same task in a
Linux sandbox, followed by the same deterministic checks. The result is either a
demonstrated improvement with an exact diff for you to review, or "No demonstrated
improvement". Nothing in your project changes until you adopt a reviewed candidate by
typing its digest, and every adoption can be rolled back.

## What the plugin contains

Three Skills, one reference document and two icon images for the OpenAI plugin
directory listing. There are no hooks, MCP servers, commands, agents or executable
files.

- `setup`: checks the installed runtime, runs its doctor and an offline demo, and
  previews a project configuration.
- `refine`: previews a refinement with a dry run that makes no model calls, runs one only
  after you authorize the spending, or evaluates an existing candidate.
- `review`: shows a run's evidence and the exact candidate diff, explains whether it is
  eligible, and guides an authorized adoption, rollback or recovery.
- `references/runtime-setup.md`: how to install, verify, upgrade and remove the runtime.

## What it runs, sends and fetches

The Skills ask the assistant to run the separately installed `harness` command-line
program (Python 3.12, Apache-2.0) on your machine, for example:

```bash
harness --version
harness doctor --json
harness demo --mode parallel --k 1 --json
harness init --project PROJECT_PATH --json
```

- The plugin itself contains no code, sends no data and contacts no service.
- Installing the runtime downloads `harness-1.0.0-py3-none-any.whl` and `SHA256SUMS`
  from the `v1.0.0` release of https://github.com/devsolux/harness-plugin. The setup
  Skill verifies the checksum and asks before it installs anything.
- `harness refine --allow-model-calls` runs the selected host CLI (Claude Code or Codex)
  inside the sandbox on a copy of the approved project files. That CLI sends the task and
  what it reads to its provider (Anthropic or OpenAI) through a local Harness gateway,
  which holds the credential outside the sandbox and enforces request, token and cost
  limits. Without that flag Harness makes no model calls.
- `harness login claude` and `harness login codex` start the vendor's own sign-in in your
  terminal and store the resulting token for the gateway.
- On macOS, or on Linux without Bubblewrap, `harness-docker build` builds a local Linux
  image. It downloads a digest-pinned Ubuntu 24.04 base image, Ubuntu packages and pinned
  Claude Code and Codex builds. The image is never pushed.
- Runs, evidence and backups stay in a local state directory. Harness has no telemetry;
  its only network client is the gateway that forwards the host CLI's requests.

## Requirements

- Python 3.12 and an isolated tool installer such as `uv`.
- Non-root Linux with Bubblewrap for sandboxed evaluation and refinement, or Docker for
  the Linux workspace image on macOS.
- For model-backed refinement: Claude Code or Codex installed system-wide, and your own
  Claude.ai or ChatGPT sign-in, or an API key that you export.

## Status

Version 1.0.0. The offline runtime, both real host CLIs against a loopback provider, and
a live Claude Code refinement with adoption and rollback have been verified. Acceptance
on a supported non-root Linux host outside CI is still pending, so supervise real runs.

## License

Apache-2.0. The full text is in `LICENSE`.
