# Phase R: Widget References

Extract JavaScript/UI patterns and process image references to inform the widget build plan.

## Phase Objective

Acquire reference repositories and images, extract JavaScript/UI patterns and visual design cues relevant to building the widget, and write a curated `WIDGET_REFERENCES.md` file. This file becomes the sole interface between references and downstream phases — creating a content air gap that prevents unintended pattern leakage.

For code references: focus exclusively on JavaScript, React, UI layout, data access, and styling patterns. Ignore Python, processor, or backend patterns.

For image references: describe the visual design, layout, color scheme, and component arrangement. Copy image assets into the widget directory for use during the Build phase.

## Execution Context

This phase runs as a subagent spawned by the orchestrator (SKILL.md). You have full access to:
- `Read` / `Glob` / `Grep` tools for exploring reference repositories
- `Write` / `Edit` tools for creating WIDGET_REFERENCES.md
- `Bash` tool for cloning repos and cleanup
- `AskUserQuestion` tool for clarifying vague extraction requests and error recovery

Your job: Acquire repos, resolve vague requests, extract targeted JS/UI patterns, write WIDGET_REFERENCES.md, cleanup, and return a summary.

## Prerequisites

- Phase 2 must be completed (widget is scaffolded)
- `.appgen/WIDGET_PHASE.md` contains reference list with locations and extraction descriptions

## Steps

### Step 1: Read Reference List

Read `{app-dir}/.appgen/WIDGET_PHASE.md` and extract the References section:
- Number of references
- For each reference: location, type (`repo`, `image`, or `asset`), and what to extract/how to use

### Step 2: Resolve Ambiguity (Before Any Extraction)

For **each** reference, evaluate the user's extraction request. If it is vague or broad, use `AskUserQuestion` to narrow scope **before reading any code or processing any image**.

**For code references (type: `repo`):**
- "how they do the UI" → Ask: "Do you mean the component layout/structure, the data fetching pattern, the styling approach, or the user interaction flow?"
- "the widget stuff" → Ask: "Do you mean the 3-layer component structure, the channel data access, the state management, or the visual design?"
- "how they display data" → Ask: "Do you mean the data fetching hooks, the rendering/formatting, the chart/graph components, or the loading/error states?"
- "the dashboard layout" → Ask: "Do you mean the grid/flex layout structure, the responsive breakpoints, the card/panel arrangement, or the navigation pattern?"

A code request is specific enough if it names a concrete aspect — e.g., "the chart rendering with recharts", "how they use useAgentChannel for real-time updates", "the Tailwind layout for the dashboard grid".

**For image references (type: `image`):**
- Images used as visual design references are generally self-explanatory — no ambiguity resolution needed unless the user's description of what to take from the image is vague
- "use this as inspiration" → Ask: "What specifically should be taken from this design? The overall layout, the color scheme, specific UI elements, the data visualization style, or all of the above?"

**For image assets (type: `asset`):**
- No ambiguity resolution needed — these are files to copy into the widget directory for direct use

When in doubt, ask. After clarification, update the extraction description in your working notes.

### Step 3: Acquire Sources

For each reference, resolve the source based on its type:

**Code repository — local path (type: `repo`):**
1. Verify the path exists using `Bash` (`test -d {path}`)
2. If it doesn't exist, use `AskUserQuestion`:
   - Options: "Correct the path" / "Skip this reference"
3. If it exists, proceed to extraction

**Code repository — GitHub (type: `repo`):**
1. Clone to a temp directory:
   ```bash
   gh repo clone {url} {app-dir}/_ref-{index} -- --depth 1 --single-branch
   ```
2. If clone fails, use `AskUserQuestion` with the exact error message:
   - Options: "Retry" / "Provide alternative URL" / "Skip this reference"
   - On second failure for the same reference, suggest checking `gh auth status`
3. Track the temp directory path for cleanup

**Image — local file (type: `image` or `asset`):**
1. Verify the file exists using `Bash` (`test -f {path}`)
2. If it doesn't exist, use `AskUserQuestion`:
   - Options: "Correct the path" / "Skip this reference"
3. If it exists:
   - For `image` type: read the image using the `Read` tool (which supports images) to view its contents
   - For `asset` type: copy to `{app-dir}/{kebab-case}/src/assets/` (create directory if needed):
     ```bash
     mkdir -p {app-dir}/{kebab-case}/src/assets
     cp {path} {app-dir}/{kebab-case}/src/assets/
     ```

**Image — URL (type: `image` or `asset`):**
1. Use `WebFetch` to verify the URL is accessible
2. If it fails, use `AskUserQuestion`:
   - Options: "Correct the URL" / "Skip this reference"
3. If accessible:
   - For `image` type: use `WebFetch` to describe the image contents for design reference
   - For `asset` type: download to `{app-dir}/{kebab-case}/src/assets/`:
     ```bash
     mkdir -p {app-dir}/{kebab-case}/src/assets
     curl -sL "{url}" -o {app-dir}/{kebab-case}/src/assets/{filename}
     ```

### Step 4: Targeted Extraction

Process each reference based on its type:

#### 4a: Code References (type: `repo`)

Focus on **JavaScript and UI patterns only**:

1. **Locate widget/JS files:** Use `Glob` to find `*.js`, `*.jsx`, `*.ts`, `*.tsx` files, particularly in widget or component directories
2. **Read only relevant files:** Focus on:
   - React component files (especially inner components — Layer 3)
   - Hook usage patterns (`useAgentChannel`, `useChannelUpdateAggregate`, `useParams`, etc.)
   - Layout and styling (Tailwind utility classes, component structure, responsive patterns)
   - Data flow patterns (how channels are read and written)
   - Chart/visualization code (if present)
   - State management (useState, useEffect patterns)
3. **Distill into pattern + snippets:**
   - Write a prose description of the pattern/approach (2-5 paragraphs)
   - Include focused code snippets (10-40 lines each) that demonstrate the pattern
   - **Scope snippets tightly:** Only include the specific functions, hooks, or JSX blocks that implement the pattern
   - Strip unrelated imports, methods, and surrounding code
   - Do NOT include entire files
4. **Write applicability notes:** How this pattern could apply to the widget being built

**Ignore:** Python files, processor code, `doover_config.json` internals, build scripts, Dockerfile, `pyproject.toml`, `__init__.py`, `application.py`, `app_config.py`, etc.

#### 4b: Image Design References (type: `image`)

For images provided as visual design references (mockups, screenshots, design comps):

1. **View the image** using the `Read` tool (which displays images visually)
2. **Describe the visual design** in detail:
   - Overall layout structure (grid, sidebar+main, card-based, etc.)
   - Color scheme and palette
   - Typography and text hierarchy
   - Data visualization elements (charts, gauges, tables, status indicators)
   - UI controls visible (buttons, inputs, toggles, dropdowns)
   - Spacing, padding, and visual rhythm
   - Responsive hints (if visible)
3. **Translate to implementation notes:**
   - Map visual elements to Tailwind CSS classes where possible
   - Identify which platform hooks would be needed for the data shown
   - Note component boundaries (what could be separate sub-components)
4. **Write applicability notes:** How this design should inform the widget build

#### 4c: Image Assets (type: `asset`)

For images the user wants included directly in the widget:

1. **Verify the file was copied** to `{app-dir}/{kebab-case}/src/assets/` during Step 3
2. **Note the asset** with its filename and intended use
3. **Write import guidance** for the Build phase:
   - The asset can be imported in the widget JS: `import myImage from './assets/filename.png';`
   - Or referenced via a relative path in JSX: `<img src="./assets/filename.png" />`
4. **Record in WIDGET_REFERENCES.md** so the Build phase knows what assets are available

### Step 5: Write WIDGET_REFERENCES.md

Create `{app-dir}/.appgen/WIDGET_REFERENCES.md` with the following format:

```markdown
# Widget References

Curated patterns, visual design notes, and assets extracted from references.

## Reference 1: {identifier} (type: repo)

### Aspect: {user's description}

#### Pattern Summary
{2-5 paragraphs describing the JS/UI approach as a reusable pattern}

#### Key Implementation Details
- {detail 1}
- {detail 2}

#### Representative Code
```javascript
{focused snippet, 10-40 lines, tightly scoped to the pattern}
```

#### Applicability Notes
{How this pattern applies to the widget being built}

---

## Reference 2: {identifier} (type: image)

### Visual Design Reference: {user's description}

#### Design Description
{Detailed description of the visual layout, color scheme, component arrangement, etc.}

#### Tailwind/CSS Translation
- Layout: {flex/grid approach and classes}
- Colors: {color palette and theme notes}
- Typography: {text sizes and hierarchy}
- Components: {buttons, cards, inputs observed}

#### Suggested Component Structure
- {Component 1}: {what it renders}
- {Component 2}: {what it renders}

#### Applicability Notes
{How this design should inform the widget build}

---

## Reference 3: {identifier} (type: asset)

### Image Asset: {filename}

- **File:** `{kebab-case}/src/assets/{filename}`
- **Intended use:** {how the user wants this image used in the widget}
- **Import:** `import {varName} from './assets/{filename}';`

---

## Extraction Metadata
- Extracted at: {timestamp}
- References processed: {count} ({repo_count} repos, {image_count} images, {asset_count} assets)
- References skipped: {count, with reasons}
```

Repeat the appropriate Reference section template for each reference based on its type. Use the `repo`, `image`, or `asset` templates as shown above.

### Step 6: User Review Checkpoint

After writing WIDGET_REFERENCES.md, present a summary to the user:
- List each reference and what was extracted
- Note any references or aspects that were skipped
- Use `AskUserQuestion` to ask: "Does this extraction look correct? You can review `.appgen/WIDGET_REFERENCES.md` for full details."
  - Options: "Looks good, proceed" / "I want to review and modify first"
  - If user wants to review: pause and tell them to re-invoke the skill after reviewing

### Step 7: Cleanup

Remove all temporary clone directories:
```bash
rm -rf {app-dir}/_ref-*
```

### Step 8: Update State

Update `.appgen/WIDGET_PHASE.md`:
- Set current phase to "Phase R - References"
- Set status to "completed"
- Add Phase R to completed phases list
- Note that WIDGET_REFERENCES.md was created

## Error Handling

- **Clone fails:** AskUserQuestion with exact error → "Retry" / "Provide alternative URL" / "Skip this reference"
- **Second clone failure (same ref):** Same options, plus suggest checking `gh auth status`
- **Local path not found:** AskUserQuestion → "Correct the path" / "Skip this reference"
- **No relevant JS code found:** Note it in WIDGET_REFERENCES.md, continue with other aspects
- **All references skipped:** Set `has_references: false` in WIDGET_PHASE.md, proceed to Phase 3 without WIDGET_REFERENCES.md

## Completion Criteria

Phase R is complete when:
- [ ] All reference extraction requests clarified (no vague requests remain)
- [ ] All reachable references acquired and processed
- [ ] Image assets copied to `{kebab-case}/src/assets/` (if any asset-type references)
- [ ] WIDGET_REFERENCES.md created with code patterns, design descriptions, and asset records
- [ ] User reviewed the extraction summary
- [ ] All temp directories cleaned up
- [ ] `.appgen/WIDGET_PHASE.md` updated with status "completed"

## Summary for Orchestrator

When returning to the orchestrator, include:
- **Status**: success or failure
- **References processed**: Count and identifiers (broken down by type: repo/image/asset)
- **References skipped**: Count with reasons
- **Code patterns extracted**: List of JS/UI patterns captured
- **Design references described**: List of visual design descriptions
- **Assets copied**: List of image files placed in widget assets directory
- **Errors**: Any issues encountered
