# Gerrit CLI Common Commands & Usage (Public)

Run the tool directly using the wrapper script path with the `--help` flag:

- `vpython3 agents/shared/skills/gerrit-cli/scripts/gerrit_client_wrapper.py --help`
- `vpython3 agents/shared/skills/gerrit-cli/scripts/gerrit_client_wrapper.py changes --help`

## Global Flags

Use the following global flags for all commands:

- `--host <url>` (e.g., `--host https://chromium-review.googlesource.com`).
- `--project <project_name>` (e.g., `--project chromium/src`).
- `--json_file <path>` (saves the structured response as a JSON file).

______________________________________________________________________

## Common Invocations

### 1. Inspecting and Searching Changes

- **Query active changes:**

  ```bash
  vpython3 agents/shared/skills/gerrit-cli/scripts/gerrit_client_wrapper.py \
    changes \
    --host https://chromium-review.googlesource.com \
    --query "owner:self status:open"
  ```

- **Get published file and line comments across patchsets:**

  ```bash
  vpython3 agents/shared/skills/gerrit-cli/scripts/gerrit_client_wrapper.py \
    comments \
    --host https://chromium-review.googlesource.com \
    --change <change_id> \
    --json_file <path>
  ```

  This writes the unchanged structured Gerrit JSON to `--json_file`. Filtering,
  threading, and presentation are the caller's responsibility.

- **Get automated check runs and findings for a patchset:**

  ```bash
  vpython3 agents/shared/skills/gerrit-cli/scripts/gerrit_client_wrapper.py \
    checks \
    --host https://chromium-review.googlesource.com \
    --change <change_id> \
    --revision <revision> \
    --json_file <path>
  ```

  `--change` is required. `--revision` is optional and defaults to `current`.
  This writes the unchanged JSON array of check runs to `--json_file`.

  Requires Gerrit's Findings plugin (`list_checks`), which returns static
  analysis findings only and does not include LUCI Buildbucket tryjobs; empty
  findings do not mean all try jobs passed. To inspect LUCI tryjob or CI builds
  for a change, load the `luci` skill if available; otherwise use the
  `luci-test-results` and `buildbucket` skills (or `bb` and `rdb` from
  `depot_tools`).

- **View the content of a specific file in a change:**

  To inspect the commit message as seen by Gerrit (including header lines for
  line-accurate commenting), use `--path /COMMIT_MSG`.

  **Note:** This command does not work with the `--json_file` flag, instead use
  stdout redirection (`>`).

  ```bash
  vpython3 agents/shared/skills/gerrit-cli/scripts/gerrit_client_wrapper.py \
    content \
    --host https://chromium-review.googlesource.com \
    --project <project> \
    --change <change_id> \
    --revision <revision> \
    --path <file_path> \
    > output_file
  ```

- **Get a formatted patch for a change:**

  `--revision <revision>` is optional and defaults to `current`. Use the
  optional `--path <file_path>`, `--parent <parent_number>`, and
  `--context <lines>` flags to customize the patch.

  **Note:** Decoded formatted patch bytes are written to stdout, so redirect
  them to a file (`>`). The `--json_file` flag is not used for patch bytes.

  ```bash
  vpython3 agents/shared/skills/gerrit-cli/scripts/gerrit_client_wrapper.py \
    patch \
    --host https://chromium-review.googlesource.com \
    --change <change_id> \
    > output.patch
  ```

- **Get related changes:**

  ```bash
  vpython3 agents/shared/skills/gerrit-cli/scripts/gerrit_client_wrapper.py \
    relatedchanges \
    --host https://chromium-review.googlesource.com \
    --change <change_id> --revision <revision>
  ```

### 2. Reviewing and Voting

- **Create a draft comment (`adddraftcomment`):** Create a draft comment on a
  revision. The command's response is the created draft JSON (CommentInfo).
  - **Patchset-level draft:** `--path` defaults to `/PATCHSET_LEVEL` when
    omitted, and `--revision` defaults to `current`.
    ```bash
    vpython3 agents/shared/skills/gerrit-cli/scripts/gerrit_client_wrapper.py \
      adddraftcomment \
      --host https://chromium-review.googlesource.com \
      --change <change_id> \
      --revision current \
      -m "Review findings: <message>"
    ```
  - **File- and line-level draft:** Specify `--path`, `-l`/`--line`, and
    optionally `--side` (`REVISION` or `PARENT`; Gerrit defaults to `REVISION`).
    Set the resolution state with `--unresolved` or `--resolved`.
    ```bash
    vpython3 agents/shared/skills/gerrit-cli/scripts/gerrit_client_wrapper.py \
      adddraftcomment \
      --host https://chromium-review.googlesource.com \
      --change <change_id> \
      --path <file_path> \
      --line <line_number> \
      --side REVISION \
      --unresolved \
      -m "<comment_message>"
    ```
  - **Reply to an existing comment:** Specify `--in-reply-to <uuid>`. If
    `--path` and `--revision` are omitted, `adddraftcomment` automatically
    resolves the parent comment's path, line/range, side, and revision.
    ```bash
    vpython3 agents/shared/skills/gerrit-cli/scripts/gerrit_client_wrapper.py \
      adddraftcomment \
      --host https://chromium-review.googlesource.com \
      --change <change_id> \
      --in-reply-to <comment_uuid> \
      --resolved \
      -m "Done"
    ```
  - **Richer fields via `--body` (`range`, `fix_suggestions`):** Pass a JSON
    object to `--body` to provide fields not covered by dedicated flags, such as
    `range` or `fix_suggestions`. CLI flags take precedence over keys in
    `--body` (e.g., `-m`/`--message` or `--path` will override `message` or
    `path` in `--body`).
    ```bash
    vpython3 agents/shared/skills/gerrit-cli/scripts/gerrit_client_wrapper.py \
      adddraftcomment \
      --host https://chromium-review.googlesource.com \
      --change <change_id> \
      --path <file_path> \
      -m "Consider renaming this variable." \
      --body '{"range": {"start_line": 15, "end_line": 15, "start_character": 4, "end_character": 12}, "fix_suggestions": [{"description": "Rename variable", "replacements": [{"path": "<file_path>", "range": {"start_line": 15, "end_line": 15, "start_character": 4, "end_character": 12}, "replacement": "new_name"}]}]}'
    ```
  - **Special Paths:**
    - `/PATCHSET_LEVEL`: Represents a review comment on the whole patchset
      rather than a file line. This is the default `--path` if omitted (unless
      replying to a comment).
    - `/COMMIT_MSG`: Represents the commit message. Line numbers include
      Gerrit's generated header lines (`Parent:`, `Author:`, `Commit:`, etc.).
      Always take line numbers from `content --path /COMMIT_MSG` rather than
      counting lines in the local commit message.
- **Add a published patchset-level comment (only when no non-author humans are
  involved):**
  ```bash
  vpython3 agents/shared/skills/gerrit-cli/scripts/gerrit_client_wrapper.py \
    addpatchsetcomment \
    --host https://chromium-review.googlesource.com \
    --change <change_id> --revision <revision> \
    --message "Review findings: <message>"
  ```
- **Vote on a review label (e.g., Code-Review +1):**
  ```bash
  vpython3 agents/shared/skills/gerrit-cli/scripts/gerrit_client_wrapper.py \
    setlabel \
    --host https://chromium-review.googlesource.com \
    --change <change_id> \
    --label Code-Review 1
  ```

### 3. Actions and Shepherding

- **Submit/Merge a change:**
  ```bash
  vpython3 agents/shared/skills/gerrit-cli/scripts/gerrit_client_wrapper.py \
    submitchange \
    --host https://chromium-review.googlesource.com \
    --change <change_id>
  ```
- **Abandon a change:**
  ```bash
  vpython3 agents/shared/skills/gerrit-cli/scripts/gerrit_client_wrapper.py \
    abandon \
    --host https://chromium-review.googlesource.com \
    --change <change_id> \
    --message "<reason>"
  ```
- **Restore an abandoned change:**
  ```bash
  vpython3 agents/shared/skills/gerrit-cli/scripts/gerrit_client_wrapper.py \
    restore \
    --host https://chromium-review.googlesource.com \
    --change <change_id> \
    --message "<reason>"
  ```

______________________________________________________________________

## Just-in-Time Help

Explore the built-in CLI help for additional subcommands or advanced syntax:

- Display all top-level commands:
  `vpython3 agents/shared/skills/gerrit-cli/scripts/gerrit_client_wrapper.py --help`
- Display help for a specific subcommand:
  ```bash
  vpython3 agents/shared/skills/gerrit-cli/scripts/gerrit_client_wrapper.py \
    help <command>
  ```

## Advanced / Escape Hatch: Arbitrary REST API Calls (`rawapi`)

If the built-in subcommands do not cover a specific action or metadata query,
you can use the `rawapi` subcommand to execute arbitrary HTTP requests against
the Gerrit REST API.

- **Endpoint Reference**:
  [Gerrit REST API - Changes](https://gerrit-review.googlesource.com/Documentation/rest-api-changes.html)

### Usage

```bash
vpython3 agents/shared/skills/gerrit-cli/scripts/gerrit_client_wrapper.py \
  rawapi \
  --host https://chromium-review.googlesource.com \
  --path "/changes/<change_id>/detail?o=SUBMITTABLE" \
  --json_file output.json
```

- **Options**:
  - `--path`: HTTP path of the API endpoint (e.g.
    `/changes/<change_id>/revisions/current/mergeable`).
  - `--method`: HTTP method (GET, POST, PUT, DELETE). Defaults to GET.
  - `--body`: JSON string body for write requests (e.g., POST/PUT).
  - `--accept_status`: Comma-separated list of successful HTTP status codes.
