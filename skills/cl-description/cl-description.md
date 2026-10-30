# Instructions for Generating a Chromium Commit Message

As an expert Chromium developer and Gerrit expert, your goal is to analyze the
current session and generate a useful CL description. You understand that a
Chromium commit message is a permanent record of technical rationale and a
trigger for automated infrastructure.

### 1. Pre-flight Investigation (Interactivity)

Before generating the final draft, first try to deduce the required details from
your current context (and inspect the local diff only if the changes are not
already in context). Do not invoke external history search skills (such as
`history-rag`) or search repository commit logs to infer commit message fields.

If any of the following are missing or ambiguous after checking your context,
**STOP and ask the user for clarification** through a series of prompts:

- **The "Why":** If the technical rationale or motivation isn't explicitly clear
  from the current context or diff (not needed if the change is self-
  explanatory from the subject line).
- **Component Tag:** Deduce the bracketed subject prefix from explicit user
  input or clear evidence from the current change. If no single tag is clearly
  supported, first ask the user if having no tag is okay or which tag to use
  before going through the local selection history workflow.
- **Bug ID:** If no bug number was mentioned in the context, first ask the user
  if having no bug is okay or if one should be associated before going through
  the local selection history workflow.
- **Internal vs. Public Bug:** If a bare bug ID is present without a clear
  tracker indicator (such as `b/`, `b:`, or `crbug.com/`), confirm if it's a
  public Chromium bug or an internal Buganizer issue.
- **Manual Testing:** If no test commands were successfully run, ask the user
  how they verified the change to populate the `Test:` footer, or if they want
  to leave it empty.
- **Histogram Obsoletion:** If non-expired UMA histograms are removed or
  renamed, and the obsoletion reason or replacement is unclear, ask the user.

#### Local Selection History

Only consult local selection history after asking the user whether having no tag
or no bug is okay and the user still wants a suggestion or wants to store a
selection. Use `.agents/cl-description/history.md`, resolved from the Chromium
repository root, as an optional record of tag and bug selections confirmed by
the user. The file is local to the checkout and ignored by Git. Read the file
directly. If it does not exist, continue without history. Do not create or edit
it directly.

Follow these rules when using the history:

- Treat every entry as a suggestion, not as authoritative.
- Use only rows in the `Tags` and `Bugs` tables as selection data.
- Match entries against the current change's terminology, affected paths, and
  purpose.
- If matching entries conflict or the match is uncertain, ask the user.
- Use a remembered bug only under the conditions in its `Applies when` value.
  For example, do not use a general feature bug for a regression or crash.
- Preserve the bug form confirmed by the user: public bug numbers are bare and
  internal Buganizer IDs use the `b:` prefix.

Update the history only when the user directly decides a value by stating it,
correcting a proposed value, or answering a question about that specific value,
**and only after asking the user if they want to store it**. Do not record an
inferred value based only on the user's general acceptance of a draft.

Use `agents/shared/skills/cl-description/scripts/history.py` to initialize and
update the file. Run the script from within the Chromium checkout.

For a confirmed tag that the user agreed to store, run:

```text
vpython3 agents/shared/skills/cl-description/scripts/history.py record-tag \
  --tag '[component]' --signals 'relevant terms and paths'
```

For a confirmed public Chromium bug that the user agreed to store, run:

```text
vpython3 agents/shared/skills/cl-description/scripts/history.py record-bug \
  --type public --id 123456 --signals 'relevant terms and paths' \
  --applies-when 'General feature work'
```

Use concise signals that identify the relevant feature or component. Describe
when a bug applies, such as `General feature work` or
`Follow-up fixes for the same issue`. Do not record unrelated CL content,
private context, or inferred selections. The history stores only the bug
reference. Determine whether to use `Bug:` or `Fixed:` from the current task.

### 2. Formatting Constraints (Mandatory)

- **72-Column Wrap:** Every line (Subject and Body) **MUST** be hard-wrapped at
  72 characters (exception: footer tags like `OBSOLETE_HISTOGRAM` must remain on
  a single line).
- **Subject Line:** A single, concise summary. Prefix it with the relevant
  component in brackets, e.g., `[Omnibox]: ...`. The entire subject line
  **MUST** be under 50 characters if possible, and no more than 72 characters.
- **Subject Spacing:** There **MUST** be exactly one blank line after the
  subject.
- **Footer Spacing:** There should be no blank lines within the footer block.
- **No Markdown-style Hyperlinks:** DO NOT use the markdown-style hyperlinks
  (e.g., `[link](url)`).

### 3. Body Content Requirements

- **Self-Explanatory Changes:** OMIT the description body entirely if the
  subject line is sufficient to understand a trivial change. Do not add filler
  paragraphs that merely rephrase the subject line.
- **Content over Code:** Do not just list what changed. Focus on **why** it was
  necessary.
- **Context:** When a body is necessary, describe the "Before" (the
  problem/baseline) and the "After" (the solution/new behavior). Usually both
  can be described in a single sentence. Avoid creating dedicated "Before" and
  "After" paragraphs if that leads to the same idea repeated twice. Focus on
  high-level ideas and avoid code references, unless absolutely necessary to
  convey the idea.
- **Documentation Links:** Do not include any links (e.g., to design docs or
  other CLs) unless specifically requested. Use the format
  `https://crrev.com/c/NUMBER` for Gerrit CL references.
- **Omit Boilerplate:** Omit tags like `RELNOTES` or `TESTED` unless
  specifically requested.

### 4. Critical Footer Logic

- **Internal Bugs (Buganizer):** MUST use the `b:` prefix. Example:
  `Bug: b:123456`.
  - **WARNING:** Never use the `b/123` format; it triggers OSS lint warnings.
- **Public Bugs:** Use the bare number. Example: `Bug: 123456`.
- **Bug: None:** IF NO BUG is associated with the session, **DO NOT** include a
  `Bug:` line at all. Do not write `Bug: None`.
- **Closing Bugs:** Use the `Fixed:` tag if the bug should be closed
  automatically.
- **Verification:** Populate the `Test:` footer with manual verification steps
  or the specific test suites run. If the user chooses to leave testing empty,
  omit the `Test:` line.
- **Histogram Obsoletion (UMA):** When removing or renaming non-expired
  histograms from `histograms.xml` (including patterned variants or suffixes),
  add obsoletion tags to the footer explaining why and when (YYYY-MM) they were
  removed, or what replaces them:
  - **Specific Histogram:** `OBSOLETE_HISTOGRAM[HistogramName]=message` (e.g.,
    `OBSOLETE_HISTOGRAM[Tab.Count]=Replaced by Tab.Count2`).
  - **All Removed Histograms:** `OBSOLETE_HISTOGRAMS=message` (used when all
    removed histograms share the same message, especially useful for patterned
    histograms; overridden by specific tags).
  - **Single Line:** The full tag **MUST** be on a single line, even if it
    exceeds 72 characters.
  - **Expired Histograms:** Can be omitted if the removed histogram was already
    expired.

______________________________________________________________________

## Final Message Template:

```
[Component] [Short summary of change (< 50 chars)]

[Description (OMIT if subject line is self-explanatory)]

Bug: [b:ID or ID]
Test: [Manual test commands or verification steps]
OBSOLETE_HISTOGRAM[HistogramName]=[Reason or replacement (single line)]
```
