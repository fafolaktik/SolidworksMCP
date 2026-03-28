# SolidWorks MCP Server

## Build & Test

```bash
npm run build        # TypeScript compile (tsc)
npm run dev          # Hot-reload dev server (tsx watch)
npm test             # Run unit tests (vitest)
npm run test:watch   # Watch mode tests
npm run lint         # ESLint check
npm run typecheck    # Type check without emit
```

**Important**: This project requires Windows + SolidWorks to run. The `winax` native module only compiles on Windows. Tests use `USE_MOCK_SOLIDWORKS=true` by default for cross-platform CI.

## Architecture

- **MCP Server** (`src/index.ts`) - Entry point, registers 88+ tools via stdio transport
- **Adapters** (`src/adapters/`) - COM bridge layer with intelligent routing:
  - `winax-adapter.ts` - Direct COM via winax
  - `winax-adapter-enhanced.ts` - Enhanced adapter with complexity analysis
  - `feature-complexity-analyzer.ts` - Routes operations between direct COM and VBA macro fallback
  - `macro-generator.ts` - Generates VBA code for complex operations
- **Tools** (`src/tools/`) - MCP tool implementations (modeling, sketch, drawing, export, analysis, VBA)
- **Core API** (`src/solidworks/api.ts`) - Low-level SolidWorks COM interface

## Key Patterns

### COM Interop
- **`SelectByID2` Callout parameter**: Never pass `null` or `undefined` — both cause "Type mismatch". Use `swApi.selectByID2()` wrapper (in `api.ts`) which passes `new winax.Variant(0, 'dispatch')` (= VBA `Nothing`). All direct `model.Extension.SelectByID2()` calls in tool handlers must go through this wrapper.
- **`RunMacro2` errors parameter**: Must use `new winax.Variant(0, 'pint32')` (= `VT_BYREF|VT_I4`, matching `ByRef Long`) — passing `0` or `new Variant(0, 'byref')` (which gives `VT_BYREF|VT_INT`, platform-sized) causes "Type mismatch". See `runMacro()` in `api.ts`.
- **`run_vba_macro` active document**: VBA macros use `Application.SldWorks.ActiveDoc` which returns whatever the user is viewing, not the MCP `currentModel`. Always call `swApp.ActivateDoc3(currentModel.GetTitle(), true, 2, errors)` before `RunMacro2` to ensure the macro sees the correct document.
- **`.swb` macro entry point**: `RunMacro2` with empty module name only finds procedures named `Main`. Use `Sub Main()` in `.swb` files, not custom names like `CreateSpurGear`.
- **`CreateSpline2(pts, True)`**: Periodic closed splines via `RunMacro2` always return `Nothing` — use connected `CreateLine` segments instead (ensure adjacent lines share the same array element value for exact endpoint coincidence).
- **`FeatureExtrusion3` requires 23 parameters** — not 13. Sketch must be selected as `'SKETCH'` type (not as a feature) for the profile to be recognised.
- **`InsertSketch(true)` is a toggle**: Always check `model.SketchManager.ActiveSketch` before calling it. If not in sketch mode, calling it creates a new sketch instead of exiting.
- **VBA macros**: Never use `MsgBox` (blocks unattended execution). Use `Debug.Print`. Never put comments after `_` line continuation (`_ ' comment` is invalid VBA — `_` must be the last character on the line).

### Code Style
- ESM modules (`"type": "module"` in package.json)
- Zod schemas for all tool input validation
- Winston logging (never use `console.*` - it breaks JSON-RPC stdio transport)
- `@ts-ignore` on winax imports is intentional (no type definitions exist)

### Code Style
- ESM modules (`"type": "module"` in package.json)
- Zod schemas for all tool input validation
- Winston logging (never use `console.*` - it breaks JSON-RPC stdio transport)
- `@ts-ignore` on winax imports is intentional (no type definitions exist)

## Testing
- Mock adapter (`src/adapters/mock-solidworks-adapter.ts`) simulates SolidWorks for CI
- Set `USE_MOCK_SOLIDWORKS=false` for integration tests on Windows with SolidWorks running
