# Harness plugin

This repository publishes the Harness plugin for Claude Code and Codex. Harness
measures whether a change to a coding agent's instructions (`CLAUDE.md`, `AGENTS.md`
and Skills) makes the agent reliably pass a project's own checks, and changes nothing
until you adopt the exact diff you reviewed.

The plugin, its documentation and its license are in `plugins/harness`. Its README
there describes everything the plugin runs, sends and fetches. The Harness source
repository is private: this repository holds only the plugin, its marketplace
catalogs and the release that carries the runtime wheel.

## Install the plugin

```bash
# Claude Code
claude plugin marketplace add https://github.com/devsolux/harness-plugin
claude plugin install harness@devsolux-harness

# Codex CLI
codex plugin marketplace add devsolux/harness-plugin
codex plugin add harness@devsolux-harness
```

## Install the runtime

The Skills need the `harness` runtime 1.0.0 on Python 3.12. Download
`harness-1.0.0-py3-none-any.whl` and `SHA256SUMS` from the
[v1.0.0 release](https://github.com/devsolux/harness-plugin/releases/tag/v1.0.0) into an
empty directory, then verify and install the wheel (on macOS use `shasum -a 256 -c`):

```bash
sha256sum -c SHA256SUMS
uv tool install --python 3.12 ./harness-1.0.0-py3-none-any.whl
harness --version
harness doctor --json
```

Then start the setup workflow: `/harness:setup` in Claude Code, or the Harness setup
workflow in Codex.

## Support and license

Report problems in this repository's issues. Harness is licensed under Apache-2.0; see
`LICENSE`.
