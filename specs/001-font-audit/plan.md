# Implementation Plan: Font Audit Plugin

**Branch**: `001-font-audit` | **Date**: 2025-11-16 | **Spec**: [spec.md](./spec.md)
**Input**: Feature specification from `/specs/001-font-audit/spec.md`

## Summary

Build a Figma plugin that provides comprehensive font and text style auditing with component hierarchy awareness, style compliance analysis, and exportable reports. The plugin will discover 100% of text layers, analyze text style assignments across libraries, suggest close style matches (80%+ similarity using weighted algorithm), and generate professional PDF/CSV exports. Optimized for a single inexperienced designer leveraging AI coding tools, using create-figma-plugin toolkit + Vite + React + TypeScript stack.

## Technical Context

**Language/Version**: TypeScript 5+ (strict mode enabled)

**Primary Dependencies**:
- Figma Plugin API (manifest version 2+)
- React 18+ (UI iframe)
- Vite (build tool with HMR)
- Figma Plugin DS + Tailwind CSS (styling)
- jsPDF (PDF export, lazy-loaded)
- Papa Parse (CSV export)
- Vitest (unit testing)

**Storage**: N/A (in-memory processing only per NFR-001, no persistent storage)

**Testing**: Vitest for unit tests (similarity calculation, traversal logic, utilities), manual testing for Figma API integration and UI interactions

**Target Platform**: Figma Desktop + Web (Plugin API dual-context: main code in sandbox + UI in iframe)

**Project Type**: Figma Plugin (specialized web application with postMessage communication between contexts)

**Performance Goals**:
- Audit 1000 text layers in <10 seconds (FR-011)
- Search/filter response <200ms for up to 1000 items (FR-009)
- Progress indicators for operations >2 seconds (FR-012)
- Incremental loading for result sets >500 items (Constitution Principle II)

**Constraints**:
- In-memory processing only, no external data transmission (NFR-001, NFR-003)
- Hard limit: 10,000 text layers maximum (FR-017)
- PDF file size <5MB for typical audits (per assumptions)
- Official Figma Plugin API only (Constitution Principle III)
- <200ms filter/search response time (FR-009)

**Scale/Scope**:
- Typical: 50-2000 text layers per audit
- Maximum: 10,000 text layers (hard limit with error message)
- Text styles: 1-10 libraries with 20-100 styles total
- Export size: <5MB PDF, unlimited CSV rows

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

### Principle I: Audit Accuracy & Completeness (NON-NEGOTIABLE)
**Requirement**: Discover 100% of text layers without false negatives

✅ **PASS** - Implementation approach:
- Use `figma.currentPage.findAllWithCriteria({ types: ['TEXT'] })` for fast, comprehensive traversal
- Enable `figma.skipInvisibleInstanceChildren = true` for performance while maintaining coverage
- Capture all required metadata per FR-002: font family, weight, size, line height, color, opacity, parent type, component path, override status, visibility state, text style assignment
- Unit tests verify 100% discovery in mock node trees with various structures

### Principle II: Performance & Responsiveness
**Requirement**: <10s for 1000 layers, non-blocking UI, progress indicators, incremental loading

✅ **PASS** - Implementation approach:
- `findAllWithCriteria` is 100x faster than `findAll` per Figma docs
- `skipInvisibleInstanceChildren` optimization enabled
- Progress indicators via postMessage updates during traversal
- Incremental loading: display results in batches of 500 items
- Non-blocking: main thread processes audit, UI thread renders results
- Performance tests verify <10s target with 1000-layer mock files

### Principle III: Figma API Fidelity
**Requirement**: Official APIs only, graceful error handling, compatibility maintenance

✅ **PASS** - Implementation approach:
- Only documented Figma Plugin API methods used (no private/undocumented APIs)
- Retry logic with exponential backoff (1s, 2s, 4s) for transient failures (FR-014)
- Graceful degradation: continue audit on node failures, mark as "Unable to read"
- Font loading: `await figma.loadFontAsync()` before reading text properties
- Handle `figma.mixed` for mixed-property text nodes
- Version pinning in package.json, update within 30 days of breaking changes

### Principle IV: User Experience & Clarity
**Requirement**: Visual indicators, click-to-navigate <200ms filters, clear labels

✅ **PASS** - Implementation approach:
- Figma Plugin DS components for consistent, professional UI
- Visual badges: styled/unstyled/partial match indicators (FR-016)
- Click-to-navigate: `figma.currentPage.selection = [node]` + `viewport.scrollAndZoomIntoView()`
- Search/filter: in-memory array operations, <200ms guaranteed for up to 1000 items
- Clear labels: "Unstyled text" not "Missing textStyleId", actionable empty states
- Manual testing verifies all interactions meet responsiveness targets

### Principle V: Data Integrity & Reproducibility
**Requirement**: Deterministic results, exact metadata capture, complete exports

✅ **PASS** - Implementation approach:
- No randomization, no timestamps in sort keys (deterministic ordering)
- Exact metadata capture: no rounding of font sizes, no color approximation
- Capture raw Figma API values without interpretation
- Export fidelity: PDF/CSV contain 100% of audit data with timestamps for audit trail
- Unit tests verify identical results on repeated runs with same input

### Principle VI: Extensibility & Maintainability
**Requirement**: Clear separation of concerns, prepared for v1.1+ features

✅ **PASS** - Implementation approach:
- Directory structure separates: `audit/` (engine), `ui/` (React components), `export/` (PDF/CSV)
- Data structures support future fields: `styleAssignment`, `styleMatchSuggestion` extensible
- Shared types in `shared/types.ts` for cross-context communication
- Comments explain non-obvious Figma API patterns (font loading, mixed properties)
- Architecture ready for v1.1 text style replacement features (clear data flow)

**No Constitution Violations** - All 6 principles satisfied by design

## Project Structure

### Documentation (this feature)

```text
specs/001-font-audit/
├── plan.md              # This file
├── research.md          # Phase 0: Figma API patterns, similarity algorithm, architecture
├── data-model.md        # Phase 1: Entity definitions, relationships, validation
├── quickstart.md        # Phase 1: Setup instructions, development workflow
├── contracts/           # Phase 1: PostMessage protocol specification
│   └── plugin-protocol.md
├── checklists/
│   └── requirements.md  # Specification quality checklist
└── spec.md              # Feature specification (already exists)
```

### Source Code (repository root)

```text
figma-fontscope/
├── src/
│   ├── main/                    # Plugin main code (Figma sandbox context)
│   │   ├── code.ts              # Entry point, postMessage handler
│   │   ├── audit/               # Core audit engine
│   │   │   ├── traversal.ts     # findAllWithCriteria, node discovery
│   │   │   ├── text-analyzer.ts # Extract font/style metadata from TextNode
│   │   │   ├── style-matcher.ts # Weighted similarity calculation (80%+)
│   │   │   └── types.ts         # Audit engine type definitions
│   │   └── utils/
│   │       ├── hierarchy.ts     # Build component hierarchy paths
│   │       ├── retry.ts         # Exponential backoff for API failures
│   │       └── font-loader.ts   # Batch font loading optimization
│   │
│   ├── ui/                      # React UI (iframe context)
│   │   ├── App.tsx              # Root component, message orchestration
│   │   ├── components/
│   │   │   ├── AuditResults.tsx       # Results display with grouping options
│   │   │   ├── SummaryDashboard.tsx   # Metrics overview, coverage %
│   │   │   ├── SearchFilter.tsx       # <200ms filtering UI
│   │   │   ├── LayerItem.tsx          # Clickable row with visual indicators
│   │   │   └── ProgressIndicator.tsx  # Non-blocking progress display
│   │   ├── hooks/
│   │   │   ├── useMessageHandler.ts   # postMessage communication
│   │   │   └── useAuditState.ts       # Audit result state management
│   │   └── styles/
│   │       └── globals.css            # Tailwind + Figma Plugin DS tokens
│   │
│   ├── shared/                  # Shared between main & UI contexts
│   │   └── types.ts             # Message protocol, audit data structures
│   │
│   └── export/                  # PDF/CSV generation (lazy-loaded)
│       ├── pdf-generator.ts     # jsPDF export with 7-section report
│       └── csv-generator.ts     # Papa Parse CSV export
│
├── tests/                       # Vitest unit tests
│   ├── audit/
│   │   ├── traversal.test.ts
│   │   ├── text-analyzer.test.ts
│   │   ├── style-matcher.test.ts     # Test weighted similarity algorithm
│   │   └── hierarchy.test.ts
│   └── utils/
│       ├── retry.test.ts
│       └── font-loader.test.ts
│
├── manifest.json                # Figma plugin manifest (auto-generated by create-figma-plugin)
├── vite.config.ts               # Vite + Tailwind configuration
├── tailwind.config.js           # Tailwind + Figma DS token integration
├── tsconfig.json                # TypeScript strict mode
├── package.json                 # pnpm dependencies
├── .eslintrc.js                 # ESLint + Figma plugin rules
├── .prettierrc                  # Prettier configuration
└── README.md                    # Setup instructions, development commands
```

**Structure Decision**: Figma Plugin architecture with dual contexts (main sandbox + UI iframe). Using create-figma-plugin toolkit for scaffolding, then customizing with Figma Plugin DS + Tailwind styling. Clear separation between audit engine (main context, pure logic), UI components (iframe, React), and export utilities (lazy-loaded in UI). This structure supports the constitution's extensibility principle and enables independent testing of core audit logic.

## Complexity Tracking

No complexity violations detected. All architectural decisions align with constitution principles and beginner + AI tool optimization goals.

## Phase 0: Research (Output: research.md)

**Goal**: Resolve all technical unknowns and document Figma Plugin API patterns, similarity algorithm, and architectural decisions.

**Deliverable**: `research.md` containing:
1. **Figma Plugin API Patterns**
   - Text node traversal best practices (`findAllWithCriteria` vs `findAll`)
   - Text style detection methods (`getRangeTextStyleId`, `getStyleById`)
   - Component hierarchy traversal (instance → mainComponent chains)
   - Font loading optimization (batch loading strategy)
   - Performance optimizations (`skipInvisibleInstanceChildren`)

2. **Similarity Algorithm Design**
   - Weighted similarity formula implementation (font family 30%, size 30%, weight 20%, line height 15%, color 5%)
   - Normalization approach for each property (0-1 scoring)
   - Threshold application (80%+ for suggestions)
   - Property comparison details (handling mixed properties, missing values)

3. **Architecture Decisions**
   - Dual-context plugin structure (main + UI)
   - PostMessage protocol design
   - State management approach (UI-side state with React hooks)
   - Progress indicator strategy (non-blocking updates)
   - Incremental loading design (500-item batches)

4. **Technology Integration**
   - create-figma-plugin setup + customization steps
   - Figma Plugin DS + Tailwind configuration
   - jsPDF lazy loading strategy
   - Vitest test setup for Figma plugin context

## Phase 1: Design (Output: data-model.md, contracts/, quickstart.md)

### data-model.md

**Entity Definitions:**

1. **TextLayerData** - Discovered text node with all metadata
   - Fields: id, content, fontFamily, fontSize, fontWeight, lineHeight, color, opacity, visible, componentContext, styleAssignment, matchSuggestions
   - Validation: All fields required except matchSuggestions (only for unstyled text)
   - Relationships: Contains ComponentContext, StyleAssignment

2. **StyleAssignment** - Text style application details
   - Fields: styleId, styleName, libraryName, assignmentStatus, propertyMatches
   - Validation: styleId required if assignmentStatus is 'fully-styled' or 'partially-styled'
   - Relationships: Referenced by TextLayerData

3. **StyleMatchSuggestion** - Recommended text style for unstyled text
   - Fields: suggestedStyleId, suggestedStyleName, libraryName, similarityScore, matchingProperties, differingProperties
   - Validation: similarityScore must be 0.80-1.00
   - Relationships: Array in TextLayerData.matchSuggestions

4. **ComponentContext** - Component hierarchy metadata
   - Fields: componentType, hierarchyPath, overrideStatus
   - Validation: hierarchyPath is array of strings (breadcrumb)
   - Relationships: Embedded in TextLayerData

5. **AuditResult** - Complete audit output
   - Fields: textLayers, summary, timestamp, fileName
   - Validation: timestamp ISO 8601 format
   - Relationships: Contains array of TextLayerData, AuditSummary

6. **AuditSummary** - Aggregate metrics
   - Fields: totalTextLayers, uniqueFontFamilies, styleCoveragePercent, librariesInUse, potentialMatchesCount, hiddenLayersCount
   - Validation: Percentages 0-100, counts non-negative
   - Relationships: Embedded in AuditResult

### contracts/plugin-protocol.md

**PostMessage Protocol Specification:**

```typescript
// UI → Main Messages
type UIToMainMessage =
  | { type: 'RUN_AUDIT'; scope: 'page' | 'selection' }
  | { type: 'NAVIGATE_TO_LAYER'; layerId: string }
  | { type: 'CANCEL_AUDIT' };

// Main → UI Messages
type MainToUIMessage =
  | { type: 'AUDIT_STARTED' }
  | { type: 'AUDIT_PROGRESS'; progress: number; current: number; total: number }
  | { type: 'AUDIT_COMPLETE'; result: AuditResult }
  | { type: 'AUDIT_ERROR'; error: string; errorType: 'VALIDATION' | 'API' | 'UNKNOWN' }
  | { type: 'NAVIGATE_SUCCESS' }
  | { type: 'NAVIGATE_ERROR'; error: string };

// Message flow examples, error handling, validation rules
```

### quickstart.md

**Setup Instructions:**
1. Prerequisites: Node 18+, pnpm, Figma desktop app
2. Project initialization: `npx create-figma-plugin` → customize structure
3. Install dependencies: Figma Plugin DS, Tailwind, jsPDF, Papa Parse, Vitest
4. Configure Vite + Tailwind for dual-context build
5. Development workflow: `pnpm dev` (watch mode), manual testing in Figma
6. Testing: `pnpm test` (Vitest unit tests), manual Figma integration tests
7. Build: `pnpm build` → generates plugin bundle for distribution

## Phase 2: Tasks (Output: tasks.md - NOT created by /speckit.plan)

Phase 2 task generation is handled by `/speckit.tasks` command and is not part of this planning phase.

## Notes for Implementation

### Beginner + AI Tool Optimization

1. **Small, testable functions**: Break similarity calculation into separate functions for each property (easier for AI to generate/modify)
2. **Strong typing**: TypeScript interfaces for all data structures (helps AI generate type-safe code)
3. **Clear naming**: `calculateFontFamilySimilarity()` not `calcFFS()` (AI can infer purpose)
4. **Separation of concerns**: Audit logic independent of UI (easier to test and generate with AI)
5. **Incremental development**: Build traversal → metadata extraction → style detection → similarity → UI → export (small, testable chunks)

### Common Pitfalls to Avoid

1. **Don't use `findAll()`** - Use `findAllWithCriteria()` for 100x performance boost
2. **Always load fonts first** - `await figma.loadFontAsync(fontName)` before reading text properties
3. **Handle `figma.mixed`** - Text can have mixed properties within one node
4. **Lazy-load jsPDF** - Don't bloat main bundle, use dynamic import
5. **Enable performance flags** - `figma.skipInvisibleInstanceChildren = true`
6. **Test with large files** - Constitution requires <10s for 1000 layers

### Critical Implementation Details

- **Similarity calculation**: Normalize all properties to 0-1 scale before applying weights
- **Progress indicators**: Send postMessage updates every 100 nodes processed
- **Incremental loading**: Virtual scrolling or pagination for >500 results
- **Click-to-navigate**: Use `figma.currentPage.selection` + `viewport.scrollAndZoomIntoView()`
- **Hidden layer reveal**: Layer stays visible while focused, auto-hides on navigation away
- **Retry logic**: Exponential backoff at 1s, 2s, 4s intervals for API failures
- **10k layer limit**: Validate count before processing, display error with split-file suggestion
