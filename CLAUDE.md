# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

JSON Formatter is a Chrome extension that automatically formats JSON when viewed in a browser tab. It features syntax highlighting, collapsible trees, dark mode, and clickable URLs. The extension is built using Deno for build tooling and TypeScript for the extension code.

## Build System & Development Commands

### Prerequisites
- [Deno](https://deno.land/) - Build system and test runner
- [Node.js](https://nodejs.org/) - Required for esbuild (npm/pnpm for installing Chrome types)

### Common Commands

**Building:**
```bash
deno task build           # Build the extension once
deno task dev             # Build and watch for changes
```

**Testing:**
```bash
deno task test            # Run tests
deno task update-snapshots # Update test snapshots
```

**Installing to Chrome:**
After building, load the `dist` folder as an unpacked extension in Chrome's extension manager.

### Build Architecture

The build system uses the [Wire](https://deno.land/x/wire) library for file transformations:

- **Entry point**: `tasks/build.ts` orchestrates a single build
- **Watch mode**: `tasks/dev.ts` watches and rebuilds on changes
- **Compilation pipeline**: `tasks/lib/compile.ts` defines transformation pipeline
- **esbuild integration**: `tasks/lib/esbuild.ts` bundles TypeScript using esbuild

Entry files follow the pattern `*.entry.ts` (e.g., `content.entry.ts`, `set-json-global.entry.ts`, `options.entry.ts`) and are compiled to `.js` files in `dist/`.

## Extension Architecture

### Content Scripts

The extension uses two content scripts that run on all URLs:

1. **content.js** (`src/content.entry.ts`):
   - Main content script that detects and formats JSON pages
   - Runs at `document_end`
   - Fast exit path: checks for JSON indicators before parsing (< 1ms on non-JSON pages)
   - Detection logic in `src/lib/getResult.ts`:
     - Must have single `<pre>` element in body
     - Must start with `{` or `[`
     - Maximum length: 3,000,000 characters
     - Must parse as valid JSON object or array
   - If JSON detected, replaces content with formatted UI

2. **set-json-global.js** (`src/set-json-global.entry.ts`):
   - Runs at `document_idle` in the `MAIN` world
   - Exposes parsed JSON as global `json` variable for devtools console inspection
   - Fast exit if not a JSON Formatter page

### Core Libraries

- **getResult.ts**: JSON detection and validation logic
- **buildDom.ts**: Recursively builds DOM tree from parsed JSON
  - Creates collapsible tree structure with indent guides
  - Handles objects, arrays, primitives
  - Converts URLs to clickable links
  - Uses template nodes (from `templates.ts`) for efficiency
- **templates.ts**: Pre-created DOM node templates for performance
- **beforeAll.ts**: Initialization code that runs before main logic

### UI Components

- **Toggle buttons**: Raw/Parsed view switcher
- **Collapsible nodes**: Click expander triangles to collapse/expand
  - Cmd/Ctrl+Click to collapse/expand all siblings
- **Theme support**: Light/dark mode via Chrome storage (`themeOverride` setting)
- **Options page**: `src/options/options.entry.ts` and `options.html`

### Styling

CSS is inlined in `src/content.entry.ts` (not imported from separate files). There are two theme variants:
- Base styles in `css` constant
- Dark theme overrides in `darkThemeCss` constant
- Theme selection via Chrome storage or system preference

## Testing

Tests use Deno's test runner with JSDOM for DOM simulation:

- Test files: `tests/getResult.test.ts`
- Fixtures: `tests/fixtures/*.html` with embedded expected results in HTML comments
- Snapshot testing: Use `UPDATE_SNAPSHOTS=1` to update expected results
- JSDOM patches: `checkVisibility` is manually implemented as JSDOM doesn't support it

## Code Patterns

### TypeScript Configuration

- Main config: `tsconfig.json` (for editor/IDE, strict mode enabled)
- Deno config: `deno.json` (for build tasks and tests)
- Build tasks in `tasks/` directory use Deno with ES2016 target

### File Naming

- `*.entry.ts` - Entry points that esbuild will bundle into standalone `.js` files
- Files in `src/lib/` - Shared utilities and core logic
- Build scripts in `tasks/lib/` - Build system utilities

### Entry File Pattern

Files ending in `.entry.ts` are bundled as separate entry points. Each becomes a standalone script in the manifest:
- `content.entry.ts` → `content.js`
- `set-json-global.entry.ts` → `set-json-global.js`
- `options.entry.ts` → `options.js`

## Performance Considerations

- Content script has negligible impact on non-JSON pages (< 1ms)
- Fast-path exit checks before expensive operations
- Template-based DOM creation for efficiency
- `content-visibility: auto` on entries for rendering performance
- 3MB size limit prevents freezing on extremely large JSON
