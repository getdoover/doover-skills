# Widget Architecture

Structure and conventions for building Doover widget apps (React + Module Federation).

## Three-Layer Component Pattern

Every widget follows a mandatory three-layer structure:

```javascript
import React from 'react';
import { useParams } from 'react-router';
import RemoteComponentWrapper from 'customer_site/RemoteComponentWrapper';
import { useRemoteParams } from 'customer_site/useRemoteParams';
import { useAgent } from 'customer_site/hooks';

// ── Layer 3: Inner component ────────────────────────────────────
// Your actual widget UI. Use useParams() for agentId.
function MyWidgetInner({ ui_element_props }) {
  const { agentId } = useParams();

  return (
    <div className="flex flex-col gap-4 p-4">
      <h1 className="text-sm font-semibold text-slate-700">My Widget</h1>
    </div>
  );
}

// ── Layer 2: Hooks wrapper ──────────────────────────────────────
// Fetches agent data. Shows loading until ready.
function MyWidgetWithAgent(props) {
  const { agentId } = useRemoteParams();
  const { agent } = useAgent(agentId);

  if (!agent) {
    return (
      <div className="flex items-center justify-center min-h-[200px] text-slate-400 text-sm">
        Loading...
      </div>
    );
  }

  return <MyWidgetInner {...props} />;
}

// ── Layer 1: RemoteComponentWrapper (outermost) ─────────────────
// Provides Redux store + React Query context.
export default function MyWidget(props) {
  return (
    <RemoteComponentWrapper>
      <MyWidgetWithAgent {...props} />
    </RemoteComponentWrapper>
  );
}
```

### Layer Purpose

| Layer | Purpose | Required |
|-------|---------|----------|
| Layer 1: RemoteComponentWrapper | Provides Redux store and React Query QueryClient so hooks work | **Yes** |
| Layer 2: Hooks wrapper | Calls `useRemoteParams()` + `useAgent()`, shows loading state | **Yes** |
| Layer 3: Inner component | All widget UI, state, and business logic | **Yes** |

### Key Rules

- Layer 3 uses `useParams()` from `react-router` for `agentId` (not passed as prop)
- Layer 2 uses `useRemoteParams()` from `customer_site/useRemoteParams` (wrapper around `useParams`)
- The default export must be the Layer 1 wrapper — this is what Module Federation exposes
- `ui_element_props` is passed through from the platform and contains UI state for this element

## Module Federation Configuration

### rsbuild.config.ts

```javascript
import { defineConfig } from '@rsbuild/core';
import { pluginReact } from '@rsbuild/plugin-react';
import { pluginModuleFederation } from '@module-federation/rsbuild-plugin';
import ConcatenatePlugin from './ConcatenatePlugin.ts';

const WIDGET_NAME = 'MyWidget';

const mfConfig = {
    name: WIDGET_NAME,
    remotes: {
        customer_site: 'customer_site@[window.dooverAdminSite_remoteUrl]',
    },
    exposes: {
        [`./${WIDGET_NAME}`]: `./src/${WIDGET_NAME}`,
    },
    shared: {
        react: { singleton: true, requiredVersion: '^18.3.1', eager: true },
        'react-dom': { singleton: true, requiredVersion: '^18.3.1', eager: true },
        'react-router': { singleton: true, requiredVersion: '^7.7.0', eager: true },
        'customer_site/hooks': { singleton: true, requiredVersion: false },
        'customer_site/RemoteAccess': { singleton: true, requiredVersion: false },
        'customer_site/queryClient': { singleton: true, requiredVersion: false },
        '@refinedev/core': { singleton: true, eager: true, requiredVersion: false },
        '@tanstack/react-query': { singleton: true, eager: true, requiredVersion: false },
    },
};

export default defineConfig({
    tools: {
        rspack: {
            plugins: [
                new ConcatenatePlugin({
                    source: './dist',
                    destination: '../assets',
                    name: `${WIDGET_NAME}.js`,
                    ignore: ['main.js'],
                }),
            ],
        },
    },
    output: { injectStyles: true },
    plugins: [
        pluginReact(),
        pluginModuleFederation(mfConfig),
    ],
    performance: {
        chunkSplit: { strategy: 'all-in-one' },
    },
});
```

### Shared Modules from customer_site

These are available to every widget via Module Federation:

| Import | Source | Contents |
|--------|--------|----------|
| `customer_site/hooks` | `./src/data/hooks` | All data hooks + `dataProvider` singleton |
| `customer_site/useRemoteParams` | `./src/hooks/useRemoteParams` | `useRemoteParams()` — wraps `useParams()` |
| `customer_site/RemoteComponentWrapper` | `./src/interpreter/wrappers/RemoteComponentWrapper` | Redux + React Query provider wrapper |
| `customer_site/RemoteAccess` | `./src/interpreter/remoteAccess` | Data access utilities |
| `customer_site/utils` | `./src/utils/utils` | `cn()` classname merge utility, `isEqual()`, `parseUnits()` |

### Shared Libraries (Singleton)

These are provided by the host application (customer_site) and shared with all widgets:

- `react` ^18.3.1
- `react-dom` ^18.3.1
- `react-router` ^7.7.0
- `@tanstack/react-query` (React Query 5)
- `@refinedev/core`

Widgets do **not** need to bundle these — they are shared at runtime.

### ConcatenatePlugin

The custom `ConcatenatePlugin.ts` concatenates the build output into a single JS file (`assets/{WidgetName}.js`). This file is what gets deployed to the platform via `file_deployments` in `doover_config.json`.

## Build Output

```bash
cd {widget-dir} && npm run build
```

Produces `assets/{PascalCase}.js` — a single self-contained JavaScript file that the platform loads at runtime via Module Federation.

## Styling

### Tailwind CSS

The Doover platform uses **Tailwind CSS v4** with a CSS-variable-based theme. Widgets render inside the platform's DOM, so **Tailwind utility classes work in widgets** when `output.injectStyles: true` is set in rsbuild config.

Use standard Tailwind utility classes for layout and styling:

```javascript
<div className="flex flex-col gap-4 p-4">
  <h3 className="text-sm font-semibold text-slate-700">Section Title</h3>
  <input
    className="flex-1 min-w-0 h-9 rounded-md border border-slate-200 bg-transparent px-3 text-sm shadow-sm placeholder:text-slate-400 focus:outline-none focus:ring-1 focus:ring-slate-400"
    placeholder="Enter value"
  />
  <button
    className="inline-flex h-9 items-center justify-center rounded-md px-4 text-sm font-medium shadow-sm disabled:opacity-50"
    style={{ backgroundColor: '#2563eb', color: '#ffffff' }}
    onClick={handleClick}
    disabled={isPending}
  >
    Submit
  </button>
</div>
```

### Styling Guidelines

- **Use Tailwind classes** for layout, spacing, typography, borders, and states
- **Use inline `style` prop** for dynamic colors or values that come from data/API
- **Use `disabled:opacity-50`** for disabled states
- **Use `focus:outline-none focus:ring-1`** for focus indicators
- The `cn()` utility is available from `customer_site/utils` for conditional class merging:
  ```javascript
  import { cn } from 'customer_site/utils';
  <div className={cn("base-classes", isActive && "active-classes")} />
  ```

### Platform Theme Variables

The platform defines CSS custom properties that Tailwind maps to semantic tokens. Useful ones for widgets:

| Token | Usage |
|-------|-------|
| `text-foreground` | Primary text color |
| `text-muted-foreground` | Secondary/dimmed text |
| `bg-background` | Page background |
| `bg-card` | Card/panel background |
| `bg-primary` / `text-primary-foreground` | Primary action colors |
| `bg-destructive` | Destructive action colors |
| `border-border` | Default border color |
| `rounded-md` / `rounded-lg` | Border radius (maps to `--radius` token) |

### No Shadcn Import

Shadcn components exist in the platform codebase (`@/components/ui/`) but are **not** exposed via Module Federation. Widgets cannot import them directly. Instead, replicate the patterns using Tailwind classes (the shadcn components are just Tailwind + CVA under the hood).

## Processor Side (Python)

Every widget app has a paired Python processor that registers the widget with the platform:

```python
from pydoover.cloud.processor import Application
from pydoover.ui import RemoteComponent

WIDGET_NAME = "MyWidget"        # PascalCase — matches JS component
FILE_CHANNEL = "my_widget"      # snake_case — matches file_deployments name

class MyWidgetApp(Application):
    async def setup(self):
        self.ui_manager.set_children([
            RemoteComponent(
                name=WIDGET_NAME,
                display_name=WIDGET_NAME,
                component_url=FILE_CHANNEL,
            ),
        ])

    async def on_aggregate_update(self, event):
        await self.ui_manager.push_async(even_if_empty=True)
        await self.api.update_aggregate(
            self.agent_id,
            "ui_state",
            {"state": {"children": {self.app_key: {"defaultOpen": True}}}},
        )
```

### Name Mapping

| Format | Example | Used For |
|--------|---------|----------|
| PascalCase | `MyWidget` | JS component name, `RemoteComponent.name`, MF scope |
| snake_case | `my_widget` | `RemoteComponent.component_url`, `file_deployments` name, Python module |
| kebab-case | `my-widget` | Widget directory name, npm package name |

## File Structure

```
{app-dir}/
├── doover_config.json         # App metadata + file_deployments + ui_state
├── pyproject.toml             # Python project with export-config script
├── build.sh                   # Lambda packaging
├── src/{app_name}/
│   ├── __init__.py            # Lambda handler
│   ├── application.py         # RemoteComponent + on_aggregate_update
│   └── app_config.py          # SubscriptionConfig + export()
└── {kebab-case}/              # Widget directory
    ├── rsbuild.config.ts      # Module Federation config
    ├── ConcatenatePlugin.ts   # Build output concatenation
    ├── package.json           # npm dependencies
    └── src/{PascalCase}.js    # Widget component (edit this)
```
