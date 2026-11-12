---
name: utr
description: >-
  Compiles local changes and/or runs Chromium tests in the bot fleet using
  CI/try builder configurations via the Universal Test Runner (UTR). Use
  proactively to reproduce or investigate bot build/test failures, or as a
  much quicker way to check local changes against a single builder than a
  full CQ run.
---

## When to Use UTR (Proactive Workflows)

UTR compiles local changes (using the builder's exact `gn_args` and `.gclient`
validation) and/or executes tests in the bot fleet (via Swarming) matching
Chromium's CI and Try builders. Proactively use UTR when:

- **Reproducing or Investigating Bot Build/Test Failures:** A compile or test
  failure occurred on a CQ run, trybot, or CI builder that does not reproduce in
  standard local `out/Default` builds (e.g. due to bot-specific `gn_args`,
  sanitizers, or platform/OS differences). Pass `-n {count}` to repeat the run
  until the first failure when reproducing flaky tests.
- **Quick Single-Builder Verification:** Checking local changes (either
  compilation via `compile` or Swarming test execution via `compile-and-test`)
  against a specific builder (prefer `-B try`) much faster than running a full
  CQ dry run.

## Basic Usage

Run `tools/utr/run.py` with `vpython3` from the Chromium `src/` checkout root:

```sh
vpython3 tools/utr/run.py -B {bucket} -b {builder} -t {test_suite} {action}
```

Read `tools/utr/README.md` locally (or run `vpython3 tools/utr/run.py -h` and
`vpython3 tools/utr/run.py compile -h`) for:

- [Command-line Usage](https://chromium.googlesource.com/chromium/src/+/main/tools/utr/README.md#command-line-usage):
  common `try` builders, host cross-compilation support, and `.gclient` setup.
- [Actions and Common Flags](https://chromium.googlesource.com/chromium/src/+/main/tools/utr/README.md#actions-and-common-flags):
  `compile` / `test` / `compile-and-test` actions, `-n` flake repetition,
  compile subparser flags (such as `--no-rbe`, placed after the action), and
  `--` test arguments.
- [Custom Builder Configs](https://chromium.googlesource.com/chromium/src/+/main/tools/utr/README.md#custom-builder-configs)
  and
  [Git Bisect](https://chromium.googlesource.com/chromium/src/+/main/tools/utr/README.md#git-bisect).
- [Troubleshooting](https://chromium.googlesource.com/chromium/src/+/main/tools/utr/README.md#troubleshooting):
  `luci-auth` login and `--no-rbe` for missing CAS inputs in Cog workspaces.

## Guidelines for AI Agents (User Interaction & Prompts)

When invoking UTR as an AI agent, you must follow these guidelines to ensure a
safe and smooth user experience:

### 1. Config-Mismatch Warnings

UTR doesn't prompt AI agents. When it finds a config mismatch, it prints the
warning and aborts, saying how to proceed:

- `args.gn` differs from the builder's: re-run with `--keep-gn-args` if you
  believe the differences in the current `args.gn` are intentional. Otherwise,
  ask the user before passing `--overwrite-gn-args` (replaces local `args.gn`).
- Coverage branch check: re-run with `--no-coverage-instrumentation`.
- `.gclient` mismatch (e.g. missing `target_os = ["win"]`): the checkout lacks
  what the builder needs. Show the warning to the user and ask how to proceed.
- Builder config may not fit this machine: show the warning to the user. Only if
  they confirm it's expected (e.g. Windows cross-compile from Linux), re-run
  with `--skip-config-validation`, and keep passing it on later runs.

**DO NOT** simply ask the user "Should I proceed?". Show the warning first, so
they know what they are agreeing to.

### 2. Don't Use `--force`

`--force` (or `-f`) silently takes every prompt's default, including the checks
above. Don't use it unless the user explicitly asks. In your final message,
quote every warning UTR printed and the flag that got past it.

### 3. Wait for Swarming Completion

`test` and `compile-and-test` block until all Swarming shards finish and print
the final results URL before exiting. Wait for the command to complete rather
than backgrounding or killing the run early.
