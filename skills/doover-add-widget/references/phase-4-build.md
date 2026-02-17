# Phase 4: Widget Build

Generate the widget JavaScript code based on the build plan.

## Phase Objective

Execute the build plan from `WIDGET_PLAN.md` to generate the widget's JavaScript component code. The plan contains all decisions already made — follow it exactly without deviation.

## Execution Context

This phase runs as a subagent spawned by the orchestrator (SKILL.md). You have full access to:
- `Read` tool for reading documentation and existing files
- `Write` / `Edit` tools for creating/modifying code
- `Bash` tool for running commands (npm install for new deps)

**Note:** Do NOT use `AskUserQuestion` — all questions should have been resolved in Phase 3 (Plan). If the plan is unclear, report this as an error.

Your job: Write the widget JavaScript code following WIDGET_PLAN.md, then return a summary.

## Prerequisites

- Phase 3 must be completed (WIDGET_PLAN.md exists)
- Widget is scaffolded (Phase 2 completed, `{kebab-case}/` directory exists)

## Steps

### Step 1: Read State and Plan

1. Read `{app-dir}/.appgen/WIDGET_PHASE.md` for:
   - Widget name (PascalCase, kebab-case)
   - App directory path

2. Read `{app-dir}/.appgen/WIDGET_PLAN.md` for:
   - Data flow (channels read/written)
   - Visual layout and component tree
   - Hooks summary (Layer 2 vs Layer 3)
   - External dependencies
   - Implementation notes
   - Reference patterns applied

### Step 2: Load Platform Documentation

Read the widget platform documentation for correct patterns and hook signatures:
- `{platform-docs-path}/references/widget-architecture.md` — 3-layer pattern, styling conventions, shared modules
- `{platform-docs-path}/references/widget-hooks.md` — Hook signatures, return values, data access patterns

Use these for correct hook signatures and Tailwind patterns. The WIDGET_PLAN.md tells you *what* to build; the platform docs tell you *how* to build it.

### Step 3: Install External Dependencies (if needed)

If WIDGET_PLAN.md lists external packages in the External Dependencies section:

```bash
cd {app-dir}/{kebab-case} && npm install {package1} {package2} ...
```

Skip this step if no external packages are listed.

### Step 4: Write Widget Component

The scaffolded widget at `{app-dir}/{kebab-case}/src/{PascalCase}.js` has three layers:

- **Layer 1** (`RemoteComponentWrapper`) — **DO NOT MODIFY.** Provides Redux store + React Query context.
- **Layer 2** (hooks wrapper, e.g., `{PascalCase}WithAgent`) — Modify only if additional hooks need setup at this level (e.g., extra loading guards for new data sources). Always keeps `useRemoteParams()` for `agentId` and `useAgent(agentId)` with the loading guard.
- **Layer 3** (inner component, e.g., `{PascalCase}Inner`) — **This is the primary target.** Replace the placeholder content with the actual widget implementation.

#### Writing the Inner Component (Layer 3)

Replace the placeholder content with the actual implementation:

1. **Add imports** at the top of the file:
   - Platform hooks from `customer_site/hooks` (e.g., `useAgentChannel`, `useChannelUpdateAggregate`)
   - `useParams` from `react-router` (for `agentId` in Layer 3)
   - React hooks (`useState`, `useEffect`, `useCallback`) as needed
   - External libraries if listed in WIDGET_PLAN.md

2. **Add hook calls** inside the component function:
   - `const { agentId } = useParams();`
   - Channel subscriptions: `const { aggregate, isLoading } = useAgentChannel(agentId, "channel_name");`
   - Mutation hooks: `const { mutate, isPending } = useChannelUpdateAggregate(agentId, "channel_name");`
   - Permission checks: `const canControl = useHasAgentPermission(AGENT_PERMISSION.DEVICE_CONTROL);`

3. **Add local state** (`useState`) for UI interactions:
   - Form input values
   - Toggle/selection states

4. **Add handler functions**:
   - `onClick`, `onChange`, `onSubmit` handlers that call mutation hooks
   - Data transformation/formatting functions

5. **Add loading/error states**:
   ```javascript
   if (isLoading) {
     return (
       <div className="flex items-center justify-center min-h-[200px] text-slate-400 text-sm">
         Loading...
       </div>
     );
   }
   ```

6. **Add JSX** — build the visual layout:
   - Use Tailwind CSS utility classes for all styling
   - Use `className` with Tailwind classes (flex, grid, p-4, text-sm, etc.)
   - Use platform CSS variables for theme colors where appropriate (e.g., `hsl(var(--primary))`)
   - Use `cn()` from `customer_site/utils` if conditional class merging is needed

#### Important Conventions

- **DO NOT** create separate CSS files — use Tailwind utility classes exclusively
- **DO NOT** invent hook signatures — use exactly what's documented in `widget-hooks.md`
- **DO NOT** modify the `rsbuild.config.ts` or `ConcatenatePlugin.ts` files
- All channel hook calls must include `agentId` as the first parameter
- Use optional chaining (`aggregate?.field`) when accessing aggregate data
- Provide fallback values for display (`aggregate?.temperature ?? 'N/A'`)
- Disable mutation buttons while `isPending` is true

#### Code Organization

If the widget is complex (>150 lines in the inner component), split into sub-components:

```
{kebab-case}/src/
├── {PascalCase}.js       (main — 3-layer structure, always exists)
├── components/            (optional — sub-components)
│   ├── DataTable.js
│   └── ControlPanel.js
└── utils/                 (optional — helper functions)
    └── formatters.js
```

Sub-components are simple React function components. Import them from the inner component in the main file:

```javascript
import DataTable from './components/DataTable';
import ControlPanel from './components/ControlPanel';
```

Sub-components receive data and callbacks as props — they do NOT call platform hooks directly. Keep all hook calls in the main component (Layer 3) and pass results down.

### Step 5: Update State

Update `.appgen/WIDGET_PHASE.md`:
- Set current phase to "Phase 4 - Build"
- Set status to "completed"
- Add Phase 4 to completed phases list
- Note files created/modified

## Completion Criteria

Phase 4 is complete when:
- [ ] Widget component code written following WIDGET_PLAN.md exactly
- [ ] All platform hooks used correctly (verified against widget-hooks.md)
- [ ] Tailwind CSS used for all styling (no CSS files)
- [ ] External dependencies installed (if any)
- [ ] 3-layer component structure preserved (Layer 1 untouched)
- [ ] Loading/error states handled
- [ ] `.appgen/WIDGET_PHASE.md` updated with status "completed"

## Summary for Orchestrator

When returning to the orchestrator, include:
- **Status**: success or failure
- **Files created/modified**: List of JS files touched
- **Hooks used**: Which platform hooks are in use
- **External deps installed**: Any npm packages added
- **Component structure**: Main component + any sub-components created
- **Errors**: Any issues encountered
