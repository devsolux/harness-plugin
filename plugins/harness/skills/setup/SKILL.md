---
name: setup
description: Set up or diagnose Harness for an explicitly selected local software project, including runtime verification and offline preflight.
---

# Set up Harness

Use this workflow when the user asks to set up, configure, or diagnose Harness.
The plugin provides instructions; the separately installed `harness` executable
owns validation and execution. Use established user choices and authorization.
Ask only for information or permission that is still missing. Explicit user
instructions take precedence over workflow preferences.

1. Establish the current host and the absolute project path, if a project was
   selected. Do not infer local execution access from the plugin being visible
   in a desktop, web, or cloud conversation.
2. Locate the executable using the host's normal command lookup. Run
   `harness --version` and `harness doctor --json`. Require version `1.0.0`.
   If it is absent or incompatible, read
   [runtime setup](../../references/runtime-setup.md). Prepare a concrete,
   verified installation command and obtain approval for provisioning if the
   user has not already authorized it. Never install a package solely by its
   unverified public package name or fetch a moving branch.
3. Inspect the doctor's actual results. A detected CLI or credential variable
   is not proof of successful authentication, sandbox confinement, or native
   host compatibility. Do not print credentials, inspect token stores, or copy
   provider secrets into the project, configuration, or chat. If the doctor's
   `sandbox.advice` names `harness-docker build` (macOS, or Linux without system
   Bubblewrap), the supported path is the Linux workspace image described in
   [runtime setup](../../references/runtime-setup.md): once the user has built it
   with `harness-docker build` (from the installed Harness, no checkout needed; again
   after every upgrade, because an image without the installed Harness is refused),
   run every `harness` command of this, the refine and the review workflow through
   `harness-docker` instead, with the same arguments; `diff`, `adopt`, `rollback`
   and `recover` need `--mount PROJECT_PATH` before the command (`:ro` suffices for
   `diff`). Sign-ins for the image come from `harness-docker login claude` or
   `harness-docker login codex`; a sign-in on the host never reaches it. Never pass
   `--privileged`.
4. Run `harness demo --mode parallel --k 1 --json` as the offline smoke test.
   It uses temporary state and makes no model calls or project changes.
   Report failures as failures; a passing demo proves only its offline path.
5. For a selected project, preview configuration with
   `harness init --project PROJECT_PATH --json`. Review the proposed configuration
   with the user. It proposes the project's `README.md` and files below `tests/`;
   every entry of `missing_paths` must be replaced, and the source files the
   checks import must be added. Reuse the project's documented acceptance checks
   and limit editable harness files to the supported, explicitly approved surface.
   Only write configuration when requested or already authorized, using
   `harness init --project PROJECT_PATH --output CONFIG_PATH --json`.
6. For native refinement, ask which authentication the user has, then prepare a
   reviewable native section with
   `harness init --project PROJECT_PATH --host HOST --native-model MODEL --json`
   (`--auth-mode subscription`, the default, for a Claude.ai or ChatGPT sign-in;
   `--auth-mode api_key` for a provider key). MODEL is the provider's full model id,
   which every session of the run uses; Claude Code aliases such as `sonnet` are
   refused, because Claude Code would request the resolved id. To change the model
   later, the user edits `native.model` in the configuration and previews again.
   `init` writes default session, request, token and wall-time limits for the user
   to review against the dry run's `forecast.sizing`; in `api_key` mode the user
   also enters that model's current token prices and a nonzero cost cap (the CLI
   refuses zero prices there), while a subscription is metered by the plan. Native
   execution needs a non-root Linux host with a passing sandbox probe (or the
   workspace image), a root-owned system install of the host CLI below `/usr` or
   `/opt`, and either a sign-in stored by `harness login claude` or
   `harness login codex` (`harness-docker login ...` for the image) or an API-key
   credential exported by the user.
   `harness login` needs the user's own terminal (it shows the vendor's URL or
   device code); never run it from a session without one, read the stored token,
   or paste a token anywhere.
   `harness doctor --config CONFIG_PATH --json` reports the remaining problems
   (`native_readiness.auth_mode`, `billing`, `credential_present`) without reading
   the credential's value, including `check_readiness` for the
   approved checks; the CLI must resolve inside a directory the sandbox mounts
   (`/usr/bin`, `/usr/lib`, `/usr/lib64` or its own directory below `/opt`).
7. Report the runtime version, doctor findings, offline demo result, selected
   project, native readiness, and next supported action. The dry run is always
   available; deterministic `evaluate` requires an enforced sandbox and an
   approved candidate/checks. Never bypass a failed runtime gate or invoke an
   internal runner directly.

Pass paths and task text as separate, safely quoted arguments. Substitute
placeholders with known values; never paste untrusted text into shell syntax.
Keep setup read-only until a particular write has been authorized.
