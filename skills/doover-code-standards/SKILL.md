---
name: doover-code-standards
description: Coding standards that must be followed when writing any code in Doover projects. Covers code conventions, import preferences, and CI linting requirements. Always apply these rules when writing or modifying code.
---

# Doover Code Standards

These rules apply to **all code** written in Doover projects. Follow them at all times unless the user explicitly requests otherwise.

## 1. Prefer Existing Patterns

Before introducing new imports, libraries, abstractions, or conventions, examine the existing codebase and solve the problem using what is already in use. Match the project's current coding style, patterns, and import choices. Only reach for something new when the existing tools genuinely cannot solve the problem or the user explicitly asks for a different approach.

## 2. CI Checks — All Code Must Pass

All code must pass the CI checks defined in the Doover project workflows. Write code that satisfies all three checks:

### ESLint (flat config, `eslint.config.mjs`)

- **No unused imports.** The `unused-imports` plugin is set to `"error"` — any unused import will fail CI. Remove imports that are not referenced.
- **No unused variables** unless prefixed with `_` (e.g. `_unusedArg`). Unused parameters after the last used one also warn.
- **React Hooks rules enforced.** Follow the rules of hooks — no conditional hooks, correct dependency arrays.
- **TypeScript-ESLint recommended rules** are active. Avoid `any` where possible, use proper type annotations.
- **React recommended rules** with JSX runtime (no need to import React for JSX).

### Prettier

- **Double quotes** for strings (`singleQuote: false`). All other settings are Prettier defaults.
- Use consistent formatting: 2-space indentation, trailing commas (ES5), semicolons, 80-char print width.
- When in doubt, match what Prettier would produce.

### TypeScript (`tsc --noEmit`, strict mode)

- **Strict mode is on.** This includes `strictNullChecks`, `noImplicitAny`, `strictFunctionTypes`, etc.
- **No unused locals** (`noUnusedLocals: true`) — declare only variables you use.
- **No unused parameters** (`noUnusedParameters: true`) — prefix unused params with `_`.
- Target is ES2020 with `moduleResolution: "bundler"`.

### General Rule

Before finishing any code change, mentally verify it would pass all three checks. If unsure, prefer the stricter interpretation.
