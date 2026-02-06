# Phase R: References

Extract patterns from reference repositories to inform the build plan.

## Phase Objective

Acquire reference repositories (local or remote), extract only the specific aspects the user requested, and write a curated `REFERENCES.md` file. This file becomes the sole interface between reference repos and downstream phases — creating a content air gap that prevents unintended pattern leakage.

## Execution Context

This phase runs as a subagent spawned by the orchestrator (SKILL.md). You have full access to:
- `Read` / `Glob` / `Grep` tools for exploring reference repositories
- `Write` / `Edit` tools for creating REFERENCES.md
- `Bash` tool for cloning repos and cleanup
- `AskUserQuestion` tool for clarifying vague extraction requests and error recovery

Your job: Acquire repos, resolve vague requests, extract targeted patterns, write REFERENCES.md, cleanup, and return a summary.

## Prerequisites

- Phase 2 must be completed
- `.appgen/PHASE.md` contains reference list with locations and extraction descriptions

## Steps

### Step 1: Read Reference List

Read `{app-directory}/.appgen/PHASE.md` and extract the References section:
- Number of references
- For each reference: location, type (local/github), and what to extract

### Step 2: Resolve Ambiguity (Before Any Extraction)

For **each** reference, evaluate the user's extraction request. If it is vague or broad, use `AskUserQuestion` to narrow scope **before reading any code**.

Examples of vague requests that need clarification:
- "how they do config" → Ask: "Do you mean the config schema field definitions, the config loading/validation logic, or the UI for editing config?"
- "the structure" → Ask: "Do you mean the file/directory layout, the class hierarchy, or the data flow pattern?"
- "how they handle data" → Ask: "Do you mean data ingestion, data transformation/processing, data storage, or data output/display?"
- "the auth stuff" → Ask: "Do you mean the authentication flow, the token management, the permission model, or the API key handling?"

A request is specific enough if it names a concrete aspect — e.g., "the MQTT connection retry logic", "their config schema field definitions", "the webhook payload parsing function". When in doubt, ask.

After clarification, update the extraction description in your working notes so the extraction is well-scoped.

### Step 3: Acquire Sources

For each reference, resolve the source:

**Local path:**
1. Verify the path exists using `Bash` (`test -d {path}`)
2. If it doesn't exist, use `AskUserQuestion`:
   - Options: "Correct the path" / "Skip this reference"
3. If it exists, proceed to extraction

**GitHub repository (`org/repo` or full URL):**
1. Clone to a temp directory:
   ```bash
   gh repo clone {url} {scratchpad}/ref-{index} -- --depth 1 --single-branch
   ```
2. If clone fails, use `AskUserQuestion` with the exact error message:
   - Options: "Retry" / "Provide alternative URL" / "Skip this reference"
   - On second failure for the same reference, include the suggestion: "You may want to check `gh auth status` to verify GitHub authentication"
3. Track the temp directory path for cleanup

### Step 4: Targeted Extraction

For each reference + each extraction aspect:

1. **Locate relevant files:** Use `Glob` and `Grep` to find files related to the requested aspect
   - Search for keywords from the extraction description
   - Look at file names, class names, function names

2. **Read only relevant files:** Do not read the entire repository. Only read files that directly relate to the extraction request.

3. **Distill into pattern + snippets:**
   - Write a prose description of the pattern/approach (2-5 paragraphs)
   - Include focused code snippets (10-40 lines each) that demonstrate the pattern
   - **Scope snippets tightly:** Only include the specific functions, classes, or blocks that implement the pattern
   - Strip unrelated methods from the same class
   - Strip unrelated imports
   - Strip surrounding code that doesn't contribute to understanding the pattern
   - Do NOT include entire files

4. **Write applicability notes:** How this pattern could apply to the app being built

If no relevant code is found for an aspect, note "No relevant code found for: {aspect}" in that section.

### Step 5: Write REFERENCES.md

Create `{app-directory}/.appgen/REFERENCES.md` with the following format:

```markdown
# References

Curated patterns extracted from reference repositories.

## Reference 1: {identifier}

### Aspect: {user's description}

#### Pattern Summary
{2-5 paragraphs describing the approach as a reusable pattern}

#### Key Implementation Details
- {detail 1}
- {detail 2}

#### Representative Code
```{language}
{focused snippet, 10-40 lines, tightly scoped to the pattern}
```

#### Applicability Notes
{How this pattern applies to the app being built}

---

## Extraction Metadata
- Extracted at: {timestamp}
- References processed: {count}
- References skipped: {count, with reasons}
```

Repeat the Reference/Aspect sections for each reference and each aspect extracted.

### Step 6: User Review Checkpoint

After writing REFERENCES.md, present a summary to the user:
- List each reference and what was extracted
- Note any references or aspects that were skipped
- Use `AskUserQuestion` to ask: "Does this extraction look correct? You can review `.appgen/REFERENCES.md` for full details."
  - Options: "Looks good, proceed" / "I want to review and modify first"
  - If user wants to review: pause and tell them to re-invoke the skill after reviewing

### Step 7: Cleanup

Remove all temporary clone directories:
```bash
rm -rf {scratchpad}/ref-*
```

### Step 8: Update State

Update `.appgen/PHASE.md`:
- Set current phase to "Phase R - References"
- Set status to "completed"
- Add Phase R to completed phases list
- Note that REFERENCES.md was created

## Error Handling

- **Clone fails:** AskUserQuestion with exact error → "Retry" / "Provide alternative URL" / "Skip this reference"
- **Second clone failure (same ref):** Same options, plus suggest checking `gh auth status`
- **Local path not found:** AskUserQuestion → "Correct the path" / "Skip this reference"
- **No relevant code found:** Note it in REFERENCES.md, continue with other aspects
- **All references skipped:** Set `has_references: false` in PHASE.md, proceed to Phase 3 without REFERENCES.md

## Completion Criteria

Phase R is complete when:
- [ ] All reference extraction requests clarified (no vague requests remain)
- [ ] All reachable references acquired and processed
- [ ] REFERENCES.md created with extracted patterns
- [ ] User reviewed the extraction summary
- [ ] All temp directories cleaned up
- [ ] `.appgen/PHASE.md` updated with status "completed"

## Summary for Orchestrator

When returning to the orchestrator, include:
- **Status**: success or failure
- **References processed**: Count and identifiers
- **References skipped**: Count with reasons
- **Aspects extracted**: List of what was captured
- **Errors**: Any issues encountered
