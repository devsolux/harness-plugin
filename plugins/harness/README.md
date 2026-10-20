# Harness plugin for Claude Code and Codex

![Harness](assets/logo.svg)

**Languages:** **English** · [Deutsch](README.DE.md) · [Türkçe](README.TR.md)

Harness measures whether a change to a coding agent's instructions (`CLAUDE.md`,
`AGENTS.md` and Skills) makes Claude Code or Codex reliably pass a project's own checks
that it failed before. Nothing in your project changes until you adopt a reviewed
candidate by typing its digest, and every adoption can be rolled back.

## Why Harness

The model is fixed: you cannot retrain Claude or GPT. What you control is the system
around it, the instructions and Skills your agent loads in every session. Most teams
change that system by intuition: add a rule, try it once or twice, keep it if it seemed
to help. That evidence is anecdotal. It is one sample without a baseline, the CLI may
have updated in between, and personal hooks, MCP servers and plugins influence the
outcome. Rules that do nothing stay, and every line is loaded in every session.

Harness turns such a change into a controlled experiment. The unchanged baseline and a
candidate run the same task in the same pinned environment, repeatedly, each followed
by the same deterministic checks. The answer is either a demonstrated improvement with
an exact diff for you to review, or **No demonstrated improvement**.

### With and without Harness

| | Without Harness | With Harness |
| --- | --- | --- |
| Evidence | One or two manual tries | Baseline versus candidate, repeated, same checks |
| Model variance | Unnoticed ("it worked this time") | Repetitions must agree, otherwise no claim |
| Environment | Your daily setup; CLI version, hooks and MCP servers vary | Pinned CLI, empty configuration, no network |
| Safety | The agent works in your real project | A sandboxed private copy; your project is untouched until you adopt |
| Model usage | Unbounded and untracked | Hard request, session, token and time limits |
| When nothing helps | The instruction file grows anyway | Nothing changes, and the result says so |
| Undo | `git revert`, if remembered | `harness rollback`, bound to the displayed diff |
| What remains | A rule of unknown effect | A rule with evidence, and the test that defines it |

### When Harness is worth it

- **Worth it** for recurring agent mistakes that a test can detect, for instruction
  files a team shares, for conventions that must hold, and when many sessions per week
  amortize the cost of a run.
- **Not worth it** when the agent already solves the task (nothing can improve, so the
  result is *No demonstrated improvement*), when no deterministic check can express the
  expectation, when the failure stems from model capability rather than instructions,
  or for one-off tasks.

## How it works

```text
You: project + task + approved files + acceptance checks + limits
                           |
        Preflight: sandbox, trusted host CLI, limits, credential, checks
                           |
   Core: baseline -> Propose -> Probe -> Compose -> Select
                           |
   Native sessions (Claude Code or Codex CLI) in Bubblewrap, no network
                 |                                   |
     loopback forwarder  -->  provider gateway (holds the credential,
                              caps requests, meters tokens and cost)
                           |
   Independent validation: fresh baseline and candidate rollouts + approved checks
                           |
   Eligibility: improved / unchanged / regressed / indeterminate
                           |
   You review the exact diff -> explicit adoption -> safe rollback
```

1. **Preflight** refuses to start without a working sandbox, a trusted host CLI,
   complete limits, a credential and checks that can run in the sandbox.
2. **Core.** The Propose–Probe–Compose core runs the unchanged baseline, lets the host
   propose instruction edits, probes and composes them, and selects a final candidate.
   At `k=1` that selection is trivial.
3. **Independent validation** runs fresh, interleaved rollouts of baseline and
   candidate; after each one the approved checks run in a network-denied sandbox.
4. **Eligibility.** A candidate is eligible only when every approved check passes for
   it, at least one baseline failure is fixed, nothing regresses and the repetitions
   agree. Otherwise Harness reports **No demonstrated improvement**. Selection is never
   evidence of improvement.
5. **Adoption** is yours alone: you review the exact diff and type its digest in your
   terminal. Rollback is bound to the same kind of digest.

Neither the model nor a plugin Skill can authorize spend, widen the scope or change
your live project. Supported instruction surfaces, limited to the files you list:
`CLAUDE.md` and `.claude/skills/<name>/SKILL.md` for Claude Code; `AGENTS.md` and
`.agents/skills/<name>/SKILL.md` for Codex.

## What the plugin contains

Three Skills, one reference document, this README in three languages and two icon
images for the OpenAI plugin directory listing. The Skills and the reference document
are in English. There are no hooks, MCP servers, commands, agents or executable files.

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
