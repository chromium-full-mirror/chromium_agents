---
name: utr
description: >-
  UTR (Universal Test Runner) is a tool to locally compile and/or run tests the
  same way it's done on Chromium CI/try builders (aka "bots"). It's particularly
  useful for reproducing bot failures or running tests on platforms you don't
  have access to locally.
---

## Basic Usage

The UTR tool is located at `tools/utr/run.py`. Always run it with `vpython3`.

When operating in remote or non-interactive workspace environments like Cider G,
ensure `depot_tools` is in your `PATH` and authenticate before invoking UTR:

```sh
luci-auth login -scopes https://www.googleapis.com/auth/userinfo.email
vpython3 tools/utr/run.py -B <bucket> -b <builder> -t <test_suite> <action>
```

### Actions

- `compile`: Only compile the targets.
- `test`: Only run the tests (assumes already compiled).
- `compile-and-test`: Compile and then run tests.

*Note: For `test` and `compile-and-test` actions, UTR will automatically track
and wait for all distributed Swarming shards to complete execution on the bot
farm and output the final aggregated results URL before exiting.*

### Common Flags

- `-B <bucket>`: The bucket name (e.g., `ci` or `try`).
- `-b <builder>`: The builder name (e.g., `Linux Tests`, `Win10 Tests x64`).
- `-t <test_suite>`: The test suite to run (e.g., `viz_unittests`,
  `url_unittests`).
- `--build-dir <dir>` or `-o <dir>`: The build directory to use for compiling
  and invoking test targets. Will use a build dir in `//out/` named after the
  builder if not specified: `//out/UTR${{builder_name}}`
- `--force` or `-f`: Takes every config-mismatch prompt's default, silently
  skipping the checks below. Don't use it unless the user explicitly asks.
- `--keep-gn-args` / `--overwrite-gn-args`: If the local `args.gn` differs from
  the builder's, keep it or replace it with the builder's args.
- `-n N`: Runs the build/test command N times without cleaning the build dir,
  and exits on the first failure.
- `--no-rbe`: Disables remote execution (RBE) and forces local compilation.
- `--`: Any args after this will be passed directly to the test executable.

## Examples and Advanced Usage

More information including examples with builder names and advanced usage can be
found at
[tools/utr/README.md](https://chromium.googlesource.com/chromium/src/+/main/tools/utr/README.md).

Information about cross-compiling Windows targets on Linux can be found at
[docs/win_cross.md](https://chromium.googlesource.com/chromium/src/+/main/docs/win_cross.md).

## Troubleshooting in Non-Interactive Environments

When running UTR inside non-interactive remote sessions, you may encounter
BeyondCorp / Context Aware Access (CAA) authentication blockers, missing remote
`.cipd_bin/` packages, `.gclient` toolchain mismatches, or RBE CAS syncing
issues:

1. **Explicit Re-authentication:** If fetching binaries or updating datasets
   stalls or raises an authentication failure, explicitly generate a fresh
   Context Aware Access token in the terminal:
   ```sh
   luci-auth login -scopes https://www.googleapis.com/auth/userinfo.email
   ```
2. **Forcing Narrow Execution Scope:** Avoid broad isolation failures by always
   supplying specific test targets:
   ```sh
   vpython3 tools/utr/run.py -t <test_suite> -p chromium -B try -b <builder> compile
   ```
3. **Missing Target OS Toolchains (e.g., Android/iOS):** If GN generation fails
   with `Missing native Android toolchain support` (or similar `target_os`
   assertions), ensure your workspace's `.gclient` configuration includes the
   necessary platform in `target_os` and run `gclient sync`:
   ```python
   solutions = [
     ...
   ]
   target_os = ["linux", "android"]
   ```
4. **Missing CAS Inputs on Remote Workers (RBE Failures):** If remote
   compilation fails because RBE cloud workers cannot find local Cog virtual
   files (e.g., `build/util/LASTCHANGE.dummy`), pass `--no-rbe` to force local
   execution where the files are successfully present:
   ```sh
   vpython3 tools/utr/run.py --no-rbe -p chromium -B ci -b <builder> -t <test> compile-and-test
   ```

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
