# Implementation Plan: Design System Style Governance

**Branch**: `002-style-governance` | **Date**: 2025-11-20 | **Spec**: [spec.md](./spec.md)

## Summary

This feature represents a **strategic pivot** from font-specific auditing to comprehensive design system governance. It provides complete visibility into text style and design token usage across entire Figma documents, enabling data-driven design system management and bulk library migrations that reduce manual work from days to minutes.

**Core Capabilities**:

- 7-state audit engine with document size validation (warning at 5k layers, hard limit at 25k)
- Style and token inventory with library source tracking
- Cross-library style replacement with adaptive batching (100→25→100 layers/batch)
- Error recovery with 3x retry + automatic rollback via version history
- Document change detection with audit invalidation
- PDF/CSV export for stakeholder reporting

## Technical Context

**Language/Version**: TypeScript 5.3+ (strict mode enabled)

**Primary Dependencies**:

- React 18.2.0 (UI layer)
- Figma Plugin API 1.109.0 (core functionality)
- jsPDF 2.5.1 (PDF export)
- PapaParse 5.4.1 (CSV export)
- @tanstack/react-virtual 3.0+ (tree virtualization - to be added)

**Storage**: In-memory audit results (no persistence), Figma document as source of truth

**Testing**: Vitest 1.1.0 (unit tests), manual testing for Figma integration

**Target Platform**: Figma Desktop App (Chromium-based, requires ES2017 transpilation)

**Project Type**: Figma Plugin (dual-context architecture: main/sandbox + UI/iframe)

**Performance Goals**:

- Audit completion: <30s for 1,000 layers, <90s for 5,000 layers, 2-10min for 5k-25k layers
- Replacement operations: ~100 layers/second optimal, ~25 layers/second degraded
- UI responsiveness: <200ms for search/filter operations
- Tree rendering: <500ms for 1,000+ styles with virtualization

**Constraints**:

- ES2017 transpilation required (no nullish coalescing `??` or optional chaining `?.`)
- Document size limits: Warning at 5,001 layers, hard limit at 25,000 layers
- Browser memory constraints for large result sets (<200MB for 25k layer documents)
- No file system access (all assets inlined in HTML via custom Vite plugin)
- Figma API rate limiting considerations for bulk operations

**Scale/Scope**:

- Target documents: 100-5,000 text layers (optimal zone), up to 25,000 (maximum supported)
- Style inventory: Up to 500 unique styles from multiple libraries
- Token support: Basic detection and replacement (chain visualization deferred to v1.1)
- Export formats: PDF (executive report), CSV (raw data for analysis)

## Constitution Check

_GATE: Must pass before Phase 0 research. Re-checked after Phase 1 design._

### ✅ I. Audit Accuracy & Completeness (NON-NEGOTIABLE)

**Status**: COMPLIANT - Enhanced

**Evidence**:

- Spec FR-002 requires detection of "all text layers regardless of their location (top-level frames, groups, component instances, nested components)"
- Existing `traversal.ts` implements recursive traversal
- Style governance extends with library source tracking (FR-008), token detection (FR-016), and enhanced metadata

**Enhancement**: Added library name resolution and token binding detection without compromising existing completeness guarantees.

### ✅ II. Performance & Responsiveness

**Status**: COMPLIANT - Requires Optimization

**Evidence**:

- Performance targets explicitly defined: FR-005 (<30s for 1,000 layers), FR-007h (reinforced)
- Document size validation prevents overload (FR-007e/f/g: warning at 5k, hard limit at 25k)
- Adaptive batching maintains responsiveness during replacements (FR-041b: 100→25 layers/batch)
- Mitigation strategies defined for Warning Zone (virtualization, progressive rendering)

**Action Required**: Implement virtualized tree rendering for 5k-25k layer documents (Phase 8).

### ✅ III. Figma API Fidelity

**Status**: COMPLIANT with Research Requirement

**Evidence**:

- Uses documented APIs: `textStyleId`, `getStyleByIdAsync`, Variables API for tokens
- Version history checkpoint API (FR-038) documented in Figma Plugin API
- Error taxonomy handles API errors, permissions, rate limiting explicitly

**Research Required**: Library name resolution (R5) - current placeholder in `styleDetection.ts` line 54-57 needs proper implementation using `importable libraries API`.

### ✅ IV. User Experience & Clarity

**Status**: COMPLIANT - Significant Enhancement

**Evidence**:

- 7-state model provides clear feedback (validating, scanning, processing, creating_checkpoint, etc.)
- User-friendly error messages (Edge Cases section: "Display a clear message stating...")
- Click-to-navigate preserved (FR-033)
- Tree organization by library/hierarchy improves discoverability (FR-021, FR-022)
- Confirmation dialogs for destructive actions (FR-039)

### ⚠️ V. Data Integrity & Reproducibility

**Status**: COMPLIANT WITH CAVEAT

**Evidence**:

- Version history checkpoints ensure rollback capability (FR-038)
- Document modification detection invalidates stale results (FR-007a/b/c/d)
- Error recovery with rollback prevents corrupted states (FR-041g)

**CAVEAT**: Document change detection requires technical implementation. Figma Plugin API doesn't provide built-in document hash. Two approaches:

1. Store `document.lastModified` timestamp at audit time
2. Listen to `figma.on('documentchange', ...)` events

**Research Required**: R1 - Test `documentchange` event reliability in collaborative editing scenarios.

### ✅ VI. Extensibility & Maintainability

**Status**: COMPLIANT

**Evidence**:

- Existing architecture separates audit engine (`src/main/utils/`), UI (`src/ui/`), message protocol (`src/shared/types.ts`)
- New entities (TextStyle, DesignToken, LibrarySource) extend existing type system naturally
- Token detection follows same pattern as style detection
- Replacement operations add new message types to existing protocol
- Data structures include optional fields for future features (`matchSuggestions?`)

### Summary

**✅ CONSTITUTION CHECK PASSED** - No violations. One caveat (document change detection) requires technical research in Phase 0 but doesn't violate the principle.

## Project Structure

### Documentation (this feature)

```text
specs/002-style-governance/
├── plan.md              # This file
├── research.md          # Phase 0: Document change detection, state management, virtualization
├── data-model.md        # Phase 1: Entity definitions, relationships, state machines
├── quickstart.md        # Phase 1: Figma API examples (library resolution, token API, version history)
├── contracts/           # Phase 1: API boundaries
│   ├── messages.md      # UIToMainMessage, MainToUIMessage extensions
│   ├── audit-api.md     # AuditEngine interface
│   ├── replacement-api.md # ReplacementEngine interface
│   └── export-api.md    # ExportEngine interface
├── tasks.md             # Phase 2: Generated by /speckit.tasks command (NOT this command)
└── checklists/          # Generated by /speckit.specify
    └── requirements.md  # Specification quality checklist
```

### Source Code (Repository Root)

```text
figma-fontscope/
├── src/
│   ├── main/                    # Figma sandbox context (main thread)
│   │   ├── code.ts              # Entry point, message handler (REFACTOR)
│   │   ├── audit/               # NEW: Audit orchestration
│   │   │   ├── auditEngine.ts   # 7-state audit state machine
│   │   │   ├── validator.ts     # Document size validation (FR-007e/f/g)
│   │   │   ├── scanner.ts       # Page/layer traversal (scanning state)
│   │   │   └── processor.ts     # Metadata extraction (processing state)
│   │   ├── replacement/         # NEW: Style/token replacement
│   │   │   ├── replacementEngine.ts  # 7-state replacement state machine
│   │   │   ├── checkpoint.ts    # Version history management
│   │   │   ├── batchProcessor.ts # Adaptive batching (100→25→100)
│   │   │   └── errorRecovery.ts # Retry logic, rollback workflow
│   │   └── utils/               # EXISTING + REFACTORED
│   │       ├── traversal.ts     # KEEP: Page traversal logic
│   │       ├── fontMetadata.ts  # DEPRECATE: Font-specific (remove in pivot)
│   │       ├── styleDetection.ts # REFACTOR: Fix library name resolution
│   │       ├── styleLibrary.ts  # ENHANCE: Team library enumeration
│   │       ├── tokenDetection.ts # NEW: Design token/variable detection
│   │       ├── hierarchy.ts     # KEEP: Component context building
│   │       ├── summary.ts       # REFACTOR: Analytics for styles/tokens
│   │       └── retry.ts         # ENHANCE: Exponential backoff (1s, 2s, 4s)
│   │
│   ├── ui/                      # React UI context (iframe)
│   │   ├── App.tsx              # REFACTOR: Multi-view navigation
│   │   ├── index.tsx            # KEEP: React mount point
│   │   ├── components/          # EXISTING + NEW
│   │   │   ├── StyleTreeView.tsx        # NEW: Library→hierarchy tree (FR-021)
│   │   │   ├── TokenView.tsx            # NEW: Token collection view (FR-028)
│   │   │   ├── DetailPanel.tsx          # NEW: Layer list for selected style
│   │   │   ├── StylePicker.tsx          # NEW: Cross-library picker (FR-036)
│   │   │   ├── ConfirmationDialog.tsx   # NEW: Replacement confirmation (FR-039)
│   │   │   ├── AnalyticsDashboard.tsx   # NEW: Charts/metrics (FR-046-050)
│   │   │   ├── ExportPanel.tsx          # NEW: PDF/CSV export UI
│   │   │   ├── ProgressIndicator.tsx    # ENHANCE: State-specific messages
│   │   │   ├── ErrorDisplay.tsx         # ENHANCE: Retry/rollback actions
│   │   │   ├── SummaryDashboard.tsx     # REFACTOR: Style/token metrics
│   │   │   └── AuditResults.tsx         # REFACTOR: Tree view instead of flat list
│   │   ├── hooks/               # EXISTING + NEW
│   │   │   ├── useAuditState.ts         # REFACTOR: 7-state model, invalidation
│   │   │   ├── useMessageHandler.ts     # ENHANCE: Replacement messages
│   │   │   ├── useDocumentChange.ts     # NEW: Change detection listener
│   │   │   ├── useReplacementState.ts   # NEW: Replacement operation state
│   │   │   └── useVirtualization.ts     # NEW: Virtual scrolling for large trees
│   │   └── styles/
│   │       └── globals.css      # KEEP: Figma design tokens
│   │
│   ├── export/                  # NEW: Export generation
│   │   ├── pdfGenerator.ts      # jsPDF report builder
│   │   ├── csvGenerator.ts      # PapaParse table export
│   │   └── formatters.ts        # Data transformation
│   │
│   ├── shared/
│   │   └── types.ts             # REFACTOR: New entities (TextStyle, DesignToken, etc.)
│   │
│   └── types.ts                 # KEEP: Plugin type augmentations
│
├── tests/                       # NEW: Test suite
│   ├── unit/
│   │   ├── auditEngine.test.ts
│   │   ├── batchProcessor.test.ts
│   │   ├── tokenDetection.test.ts
│   │   └── summary.test.ts
│   └── integration/
│       └── messageFlow.test.ts
│
├── manifest.json                # KEEP: Plugin metadata
├── vite.config.ts               # KEEP: Dual-context build
├── vite-plugin-figma.ts         # KEEP: HTML inlining
├── tailwind.config.js           # KEEP: Figma design tokens
└── package.json                 # UPDATE: Add @tanstack/react-virtual
```

**Structure Decision**: Preserve existing dual-context architecture (src/main for sandbox, src/ui for iframe). This is fundamental to Figma plugins. New capabilities (audit engine, replacement engine, export) fit naturally into this structure without requiring monorepo or multi-project organization.

## Complexity Tracking

**No constitution violations to justify.**

## Research Topics (Phase 0)

### R1: Document Change Detection Mechanism

**Question**: How to reliably detect document modifications to invalidate audit results (FR-007a)?

**Why Critical**: Spec requires showing warning when audit is stale. Figma Plugin API doesn't provide built-in document hash or guaranteed change notifications.

**Investigation Plan**:

1. Test `figma.on('documentchange', callback)` event behavior
2. Test in collaborative editing scenarios (multiple users, one running audit)
3. Compare with timestamp approach (`figma.root.lastModified`)
4. Measure performance impact of event listeners on 25k layer documents
5. Test false positive rate (events firing when document unchanged)

**Acceptance Criteria**: Detection strategy works reliably in both solo and collaborative editing without excessive false positives.

**Fallback**: If `documentchange` unreliable, use timestamp + prominent "Re-run Audit" button always visible.

### R2: State Management Library Choice

**Question**: Continue custom hook-based state or adopt library (Zustand, Valtio, Jotai)?

**Current State**: `useAuditState.ts` uses singleton closure pattern
**New Requirement**: 7-state audit + 7-state replacement + document invalidation tracking

**Investigation Plan**:

1. Prototype 7-state machine in Zustand vs custom hooks
2. Measure bundle size impact (plugin code must be inlined)
3. Evaluate TypeScript state transition type safety
4. Test developer experience for state machine debugging

**Acceptance Criteria**: State management clearly represents state machines, doesn't bloat bundle >5KB, integrates with React hooks.

**Decision Matrix**:
| Library | Bundle Size | Type Safety | DX | State Machine Support |
|---------|-------------|-------------|----|-----------------------|
| Custom Hooks | 0 KB | Manual | Good | Manual transitions |
| Zustand | ~1 KB | Excellent | Excellent | Manual transitions |
| Valtio | ~3 KB | Good | Excellent | Manual transitions |
| XState | ~15 KB | Excellent | Complex | Built-in state machines |

**Recommendation**: Zustand if bundle size acceptable, otherwise enhance custom hooks with explicit state transition guards.

### R3: Virtualization Library for Large Trees

**Question**: Which library handles tree structures with 500+ nodes best?

**Requirement**: Spec mitigation for 5k-25k layer documents requires "Virtualized lists: Only render visible tree nodes to reduce DOM size"

**Investigation Plan**:

1. Test `react-window` vs `@tanstack/react-virtual` vs `react-virtual`
2. Benchmark tree expand/collapse with 1,000 style nodes
3. Verify iframe compatibility in Figma plugin environment
4. Measure rendering performance improvement for Warning Zone docs
5. Test keyboard navigation with virtualized lists

**Acceptance Criteria**: Tree view renders smoothly with 1,000+ styles, <100ms expand/collapse latency, maintains keyboard accessibility.

**Candidates**:

- `react-window`: Mature, 9KB, proven reliability
- `@tanstack/react-virtual`: Modern, 5KB, better TypeScript support, framework-agnostic
- `react-virtual`: Older, 12KB, falling out of maintenance

**Recommendation**: `@tanstack/react-virtual` for modern API and TS support.

### R4: PDF Generation Library Capabilities

**Question**: Can jsPDF handle required visualizations (FR-051)?

**Current**: jsPDF 2.5.1 already in package.json
**Required**: Executive summary, style inventory tables, adoption charts, metadata

**Investigation Plan**:

1. Test jsPDF autotable plugin for tabular data
2. Evaluate chart generation (jsPDF has limited built-in charting)
3. Test SVG embedding for visualizations (use Chart.js to SVG, embed in PDF)
4. Benchmark multi-page report generation performance
5. Verify PDF file size for large documents (<5MB for 25k layer audit)

**Acceptance Criteria**: jsPDF produces spec-compliant PDF reports without adding >50KB dependencies for charting.

**Fallback**: If charting too complex, generate simpler table-only reports for MVP, defer charts to v1.1.

### R5: Figma Variables API for Token Detection

**Question**: Complete API surface for detecting and replacing design tokens?

**Requirement**: FR-016 through FR-020 (token detection), FR-042 through FR-045 (token replacement)

**Investigation Plan**:

1. Document `figma.variables.getLocalVariables()` usage
2. Test `figma.variables.getVariableById(id)` for token metadata
3. Understand token binding to text properties (`boundVariables` on TextNode)
4. Test token replacement via `setBoundVariable()` or similar
5. Investigate mode information (`resolvedType`, `valuesByMode`)
6. Test collection grouping and enumeration

**Acceptance Criteria**: Clear API contract with code examples for token detection and replacement.

**Deliverable**: `quickstart.md` section with working token detection/replacement examples.

### R6: Adaptive Batching Implementation

**Question**: Best approach for 100→25→100 adaptive batch sizing (FR-041b)?

**Spec Requirements**:

- Start 100 layers/batch
- Reduce to 25 on errors
- Increase back to 100 after 5 consecutive successes
- Retry failed batches 3x with exponential backoff

**Investigation Plan**:

1. Design batch processor state machine (track size, consecutive success count, retry count)
2. Test Figma API timeout behavior with varying batch sizes
3. Profile actual throughput (validate spec's 100 layers/s estimate)
4. Implement partial batch failure handling (FR-041f: mark failed layers, continue)
5. Test error classification (transient vs persistent)

**Acceptance Criteria**: Batch processor passes unit tests for all state transitions and error scenarios.

**Deliverable**: `batchProcessor.ts` with full test coverage.

### R7: Version History Checkpoint API

**Question**: Correct Figma API for creating version checkpoints (FR-038)?

**Requirement**: "System MUST create automatic version history checkpoint before executing any style replacement operation"

**Investigation Plan**:

1. Identify checkpoint creation API (likely `figma.saveVersionHistoryAsync(title)`)
2. Test checkpoint timing in replacement workflow (creating_checkpoint state)
3. Verify checkpoint appears in Figma's File > Version History UI
4. Test user rollback workflow (manual undo via version history)
5. Document any API limitations or required permissions

**Acceptance Criteria**: Checkpoint creation succeeds before bulk operations, version history shows timestamped savepoint with custom title.

**Deliverable**: `quickstart.md` section with version history API examples.

## Implementation Phases

### Phase 0: Research & Setup (Week 1)

**Goal**: Answer all research questions, establish technical foundation

**Deliverables**:

- `research.md` with findings from R1-R7
- `data-model.md` with complete entity definitions
- `contracts/` directory with message protocol, API boundaries
- `quickstart.md` with Figma API code examples
- Test infrastructure (Vitest config, mock Figma API utilities)
- Decision on state management library (R2)
- Decision on virtualization library (R3)

**Exit Criteria**: Team can confidently answer:

- "How do we detect document changes?" (R1)
- "Which state management approach?" (R2)
- "Which virtualization library?" (R3)
- "Can jsPDF handle our reports?" (R4)
- "How do we detect and replace tokens?" (R5)
- "How do we implement adaptive batching?" (R6)
- "How do we create version checkpoints?" (R7)

**Dependencies**: None (first phase)

---

### Phase 1: 7-State Audit Engine (Weeks 2-3)

**Goal**: Implement User Story 1 (P1) - Complete Style Audit

**User Story Coverage**:

- P1: Complete Style Audit of Document
- All 5 acceptance scenarios from User Story 1

**Scope**:

- Refactor `useAuditState.ts` for 7-state model (idle → validating → scanning → processing → complete/error/cancelled)
- Implement document size validation (FR-007e/f/g)
- Build style inventory with library source detection (FR-008, FR-009)
- Implement document change detection and invalidation (FR-007a/b/c/d)
- Create `StyleTreeView.tsx` with library→hierarchy grouping (FR-021, FR-022)
- Build unstyled text detection and grouping (FR-013, FR-014, FR-015)

**Technical Tasks**:

1. Create `src/main/audit/auditEngine.ts` with state machine
2. Create `src/main/audit/validator.ts` for document size checks
3. Refactor `src/main/utils/styleLibrary.ts` for team library enumeration
4. Fix library name resolution in `styleDetection.ts` (remove placeholder at line 54-57)
5. Build `src/ui/components/StyleTreeView.tsx` with expand/collapse
6. Implement `src/ui/hooks/useDocumentChange.ts` for invalidation
7. Refactor `code.ts` message handler to emit state-specific progress updates

**Dependencies**: Phase 0 (R1, R2 decisions)

**Acceptance Criteria**:

- ✅ Audit scans all pages and displays complete style inventory (Scenario 1)
- ✅ Styles grouped by library source with expandable hierarchy (Scenario 2)
- ✅ Unstyled text grouped in "Needs Styling" section (Scenario 3)
- ✅ Audit completes in <30s for 1,000 layers with progress indication (Scenario 4)
- ✅ Cancel button stops audit immediately, returns to idle (Scenario 5)
- ✅ Warning displayed at 5,001+ layers
- ✅ Hard limit blocks audit at 25,001+ layers
- ✅ Document modification triggers stale data warning

**Exit Criteria**: Core audit functionality complete, tree view displays styles correctly.

---

### Phase 2: Token Detection & Analytics (Week 4)

**Goal**: Implement User Story 3 (P3) - Analyze Token Usage

**User Story Coverage**:

- P3: Analyze Token Usage
- All 4 acceptance scenarios from User Story 3

**Scope**:

- Implement token detection via Figma Variables API (FR-016, FR-017, FR-018)
- Build token inventory with usage tracking (FR-019, FR-020)
- Create `TokenView.tsx` component (FR-028, FR-029, FR-030)
- Build `AnalyticsDashboard.tsx` with style + token metrics (FR-046 through FR-050)
- Extend summary calculation for token coverage

**Technical Tasks**:

1. Create `src/main/utils/tokenDetection.ts` with variable binding detection
2. Extend `AuditResult` entity to include `tokenInventory: DesignToken[]`
3. Build `src/ui/components/TokenView.tsx` with collection grouping
4. Build `src/ui/components/AnalyticsDashboard.tsx` with coverage charts
5. Update `summary.ts` to calculate token coverage percentage (FR-048)
6. Add "Tokens" tab to UI navigation

**Dependencies**: Phase 0 (R5), Phase 1 (audit engine)

**Acceptance Criteria**:

- ✅ Tokens view displays all detected tokens with names/values/counts (Scenario 1)
- ✅ Selecting token shows all layers using it, grouped by page/component (Scenario 2)
- ✅ Mixed usage (style + token) correctly identified (Scenario 3)
- ✅ Analytics dashboard shows token coverage % alongside style adoption (Scenario 4)
- ✅ Token detection handles mode information (FR-020)

**Exit Criteria**: Token visibility complete, analytics dashboard shows comprehensive metrics.

---

### Phase 3: Detail Panel & Navigation (Week 5)

**Goal**: Complete interactive exploration capabilities

**User Story Coverage**:

- Enhancement to P1 (drill-down from tree to layer details)
- Enhancement to P3 (drill-down from token to layer details)

**Scope**:

- Build `DetailPanel.tsx` showing layers using selected style (FR-031, FR-032)
- Implement page/component grouping in detail panel (FR-033)
- Enhance click-to-navigate functionality (FR-034)
- Add search/filter for styles and tokens (FR-025, FR-026)
- Implement visual indicators for style status (FR-027)

**Technical Tasks**:

1. Create `src/ui/components/DetailPanel.tsx` with virtualized layer list
2. Implement search functionality (Fuse.js or native filtering)
3. Add filter dropdowns (by library, by usage count, by assignment status)
4. Test click-to-navigate with existing `handleNavigateToLayer`
5. Add visual badges (fully-styled, partially-styled, unstyled)
6. Implement keyboard shortcuts for tree navigation

**Dependencies**: Phase 1 (tree view), Phase 2 (token view)

**Acceptance Criteria**:

- ✅ Selecting style in tree shows all layers using it (FR-031)
- ✅ Detail panel groups layers by page and component (FR-032)
- ✅ Clicking layer focuses it in Figma canvas (FR-033)
- ✅ Search filters 500+ styles in <200ms (FR-025)
- ✅ Filter by library, usage, status works correctly (FR-026)
- ✅ Visual indicators clearly show assignment status (FR-027)

**Exit Criteria**: Full exploratory workflow functional (tree → detail → canvas navigation).

---

### Phase 4: Style Replacement Engine (Weeks 6-7)

**Goal**: Implement User Story 2 (P2) - Migrate Styles Between Libraries

**User Story Coverage**:

- P2: Migrate Styles Between Libraries
- All 5 acceptance scenarios from User Story 2

**Scope**:

- Build 7-state replacement state machine (FR-041)
- Implement version history checkpoint creation (FR-038)
- Build adaptive batching with 100→25→100 logic (FR-041b)
- Implement retry with exponential backoff (FR-041c)
- Error recovery and rollback workflow (FR-041d/e/f/g/h)
- Build `StylePicker.tsx` for cross-library selection (FR-036)
- Build `ConfirmationDialog.tsx` for replacement preview (FR-039)

**Technical Tasks**:

1. Create `src/main/replacement/replacementEngine.ts` with state machine
2. Create `src/main/replacement/checkpoint.ts` for version history API
3. Create `src/main/replacement/batchProcessor.ts` with adaptive algorithm
4. Create `src/main/replacement/errorRecovery.ts` with retry + rollback
5. Build `src/ui/components/StylePicker.tsx` with library filtering
6. Build `src/ui/components/ConfirmationDialog.tsx` with affected layer preview
7. Create `src/ui/hooks/useReplacementState.ts` for operation tracking
8. Extend message protocol with replacement messages

**Dependencies**: Phase 0 (R6, R7), Phase 3 (detail panel provides affected layer list)

**Acceptance Criteria**:

- ✅ Style picker displays all available styles from all libraries (Scenario 1)
- ✅ Confirmation dialog shows correct format and affected count (Scenario 2)
- ✅ Version checkpoint created before starting replacement (Scenario 3)
- ✅ All layers updated, component overrides preserved (Scenario 4)
- ✅ Component instances maintain override states (Scenario 5)
- ✅ Batch size adapts on errors (100→25)
- ✅ Transient errors retry 3x with backoff (1s, 2s, 4s)
- ✅ Persistent errors trigger rollback confirmation
- ✅ Partial failures collected and reported

**Exit Criteria**: Style replacement fully functional with all safety guarantees.

---

### Phase 5: Token Replacement (Week 8)

**Goal**: Implement User Story 4 (P4) - Replace Tokens Across Document

**User Story Coverage**:

- P4: Replace Tokens Across Document
- All 3 acceptance scenarios from User Story 4

**Scope**:

- Extend replacement engine for token operations (FR-042, FR-043)
- Reuse checkpoint, batching, error recovery from Phase 4 (FR-044, FR-045)
- Build token picker UI (similar to style picker)
- Extend confirmation dialog for token replacements

**Technical Tasks**:

1. Extend `replacementEngine.ts` with `replaceToken()` method
2. Implement token binding update via `setBoundVariable()` API
3. Build token picker component (adapt StylePicker)
4. Add token replacement confirmation dialog (adapt ConfirmationDialog)
5. Test token replacement with various binding types (color, typography)

**Dependencies**: Phase 0 (R5), Phase 4 (replacement engine)

**Acceptance Criteria**:

- ✅ Token picker displays all available tokens from collections (Scenario 1)
- ✅ Confirmation shows affected layer count and token details (Scenario 2)
- ✅ All layers updated with new token, change reflected in next audit (Scenario 3)
- ✅ Token replacement follows same safety model as style replacement (checkpoint, retry, rollback)
- ✅ Failed token updates reported with layer IDs

**Exit Criteria**: Token replacement operational with safety parity to style replacement.

---

### Phase 7: Polish & Edge Cases (Week 9)

**Goal**: Handle all edge cases, final UX polish

**User Story Coverage**:

- Enhancement to all user stories (robustness, accessibility, edge cases)

**Scope**:

- Implement all 28 edge cases from spec "Edge Cases" section
- Keyboard navigation for tree view
- Empty states for zero results
- Loading skeletons during state transitions
- Accessibility review (ARIA labels, focus management)
- Error message refinement

**Technical Tasks**:

1. Handle "no text layers found" gracefully (empty state)
2. Handle "style exists but zero usage" display
3. Handle "missing/deleted style" warnings
4. Handle "network issues prevent library access" (spec note)
5. Handle "same source and target style" validation error
6. Add keyboard shortcuts (Space=expand/collapse, Enter=select, Arrow keys=navigate)
7. Implement focus trapping in modals
8. Add ARIA labels to tree nodes, buttons, dialogs
9. Test screen reader announcements for state changes
10. Add loading skeletons for async operations

**Dependencies**: Phase 1-6 (all major features complete)

**Acceptance Criteria**:

- ✅ All 28 edge cases from spec handled correctly
- ✅ Keyboard-only navigation fully functional
- ✅ Screen reader announces state changes appropriately
- ✅ Empty states provide clear guidance
- ✅ Loading states smooth and informative
- ✅ Focus management prevents keyboard traps

**Exit Criteria**: Production-ready quality, all edge cases handled, fully accessible.

---

### Phase 8: Performance Optimization (Week 10)

**Goal**: Ensure Warning Zone (5k-25k layers) performance

**User Story Coverage**:

- Performance enhancement for all user stories at scale

**Scope**:

- Implement virtualized tree rendering (from spec mitigation strategies)
- Progressive loading for large result sets
- Optimize summary calculation for 25k+ data points
- Memory profiling and optimization
- Reduce real-time update frequency for large batches

**Technical Tasks**:

1. Integrate `@tanstack/react-virtual` (chosen in Phase 0)
2. Implement virtual scrolling in StyleTreeView
3. Implement virtual scrolling in DetailPanel layer list
4. Add progressive batch rendering (display first 100 styles, load rest in background)
5. Optimize summary.ts algorithms (use Map for O(1) lookups)
6. Profile memory usage with 25k layer test document
7. Implement reduced progress update frequency (50 layers vs 10)

**Dependencies**: Phase 0 (R3), Phase 1-7 (all major features complete)

**Acceptance Criteria**:

- ✅ Tree view with 1,000+ styles renders in <500ms
- ✅ Detail panel with 5,000+ layers scrolls smoothly (60fps)
- ✅ Total memory usage <200MB for 25k layer document
- ✅ Audit completes 5k-25k layer documents in 2-10 minutes
- ✅ UI remains responsive during Warning Zone audits

**Exit Criteria**: Warning Zone (5k-25k layers) performance meets spec targets.

---

### Phase 9: Export & Reporting (Week 11)

**Goal**: Implement User Story 5 (P5) - Export Audit Report

**User Story Coverage**:

- P5: Export Audit Report for Stakeholders
- All 4 acceptance scenarios from User Story 5

**Scope**:

- Build PDF report generator with jsPDF (FR-051, FR-052)
- Build CSV export with PapaParse (FR-053)
- Include document metadata in exports (FR-053)
- Build `ExportPanel.tsx` UI component

**Technical Tasks**:

1. Create `src/export/pdfGenerator.ts` with executive summary, tables, charts
2. Create `src/export/csvGenerator.ts` with one-row-per-layer format
3. Create `src/export/formatters.ts` for data transformation
4. Build `src/ui/components/ExportPanel.tsx` with PDF/CSV buttons
5. Implement file download in iframe context (blob URLs, anchor click)
6. Test export performance with large datasets (25k layer audit)

**Dependencies**: Phase 0 (R4), Phase 2 (analytics dashboard data), Phase 8 (all features optimized)

**Acceptance Criteria**:

- ✅ PDF contains executive summary, style inventory, adoption visualizations (Scenario 1)
- ✅ PDF includes document metadata (filename, timestamp, page count, metrics) (Scenario 2)
- ✅ CSV exports all text layer metadata for analysis (Scenario 3)
- ✅ CSV properly formatted for spreadsheet software (Scenario 4)
- ✅ Both formats include audit timestamp
- ✅ Export completes in <60 seconds for 25k layer documents

**Exit Criteria**: Full export functionality operational, reports suitable for stakeholder presentations.

---

## Timeline Summary

| Phase       | Duration  | Completion     | User Stories | Key Deliverables                                        |
| ----------- | --------- | -------------- | ------------ | ------------------------------------------------------- |
| **Phase 0** | Week 1    | End of Week 1  | Research     | research.md, data-model.md, contracts/, quickstart.md   |
| **Phase 1** | Weeks 2-3 | End of Week 3  | P1           | 7-state audit engine, style tree view, change detection |
| **Phase 2** | Week 4    | End of Week 4  | P3           | Token detection, analytics dashboard                    |
| **Phase 3** | Week 5    | End of Week 5  | Enhancement  | Detail panel, search/filter, navigation                 |
| **Phase 4** | Weeks 6-7 | End of Week 7  | P2           | Style replacement with safety guarantees                |
| **Phase 5** | Week 8    | End of Week 8  | P4           | Token replacement                                       |
| **Phase 6** | Week 9    | End of Week 9  | Polish       | Edge cases, accessibility, keyboard nav                 |
| **Phase 7** | Week 10   | End of Week 10 | Scale        | Performance optimization for 5k-25k layers              |
| **Phase 8** | Week 11   | End of Week 11 | P5           | PDF/CSV export                                          |

**MVP Delivery Point**: End of Phase 4 (Week 7) delivers core value (P1, P2: audit + replacement)
**Full v1.0 Release**: End of Phase 8 (Week 11) with all features from spec

## Risk Mitigation

### Technical Risks

**Risk 1: Document change detection unreliable in collaborative editing**

- **Likelihood**: Medium
- **Impact**: High (breaks stale data warning)
- **Mitigation**: Phase 0 research (R1) tests `documentchange` events with multiple users. If unreliable, fallback to timestamp + always-visible "Re-run Audit" button.
- **Contingency**: Manual refresh acceptable for MVP, automated detection in v1.1

**Risk 2: Figma API rate limiting during bulk replacements**

- **Likelihood**: Medium
- **Impact**: Medium (slow replacements, user frustration)
- **Mitigation**: Adaptive batching (Phase 4) designed for this. Exponential backoff spaces requests on 429 errors.
- **Contingency**: If rate limiting severe, reduce default batch size to 50 layers

**Risk 3: Browser memory exhaustion on 25k+ layer documents**

- **Likelihood**: Low (hard limit prevents this)
- **Impact**: Critical (browser crash, data loss)
- **Mitigation**: Hard limit at 25k (FR-007g). Warning Zone virtualization (Phase 7) reduces DOM size.
- **Contingency**: Lower hard limit to 20k if crashes occur during testing

**Risk 4: Version history checkpoint API undocumented/unreliable**

- **Likelihood**: Low
- **Impact**: High (breaks safety guarantees)
- **Mitigation**: Phase 0 research (R7) tests API. Figma Plugin API typically well-documented.
- **Contingency**: If unavailable, show prominent warning: "Create manual checkpoint before proceeding" with required user confirmation

### Timeline Risks

**Risk 5: 11-week timeline ambitious for solo developer**

- **Likelihood**: Medium
- **Impact**: Medium (delays v1.0 release)
- **Mitigation**: Phases prioritized by user story priority (P1→P2→P3→P4→P5). MVP can ship after Phase 4 (7 weeks) with core functionality.
- **Contingency**: Defer Phase 5-6 (token replacement, export) to v1.1 for faster MVP

**Risk 6: Research phase uncovers blockers**

- **Likelihood**: Low
- **Impact**: Critical (invalidates assumptions)
- **Mitigation**: Phase 0 designed to surface showstoppers early. All research questions answerable.
- **Contingency**: If blocker found, pivot spec assumptions or defer feature

### Dependency Risks

**Risk 7: Figma Plugin API breaking changes**

- **Likelihood**: Low
- **Impact**: High (plugin breaks)
- **Mitigation**: Constitution Principle III requires staying current. Monitor changelog, update within 30 days.
- **Contingency**: Pin specific Figma Plugin API version, test updates before deploying

## Next Steps

1. **Immediate**: Review this plan with stakeholders
   - Confirm 11-week timeline acceptable
   - Confirm MVP scope (Phases 1-4 sufficient or full v1.0 required?)
   - Confirm resource allocation (solo developer or team?)

2. **Phase 0 Kickoff**: Begin research investigations (R1-R7)
   - Allocate 1 week for research
   - Generate `research.md`, `data-model.md`, `contracts/`, `quickstart.md`
   - Make technology decisions (state management, virtualization)

3. **After Phase 0**: Run `/speckit.tasks` to generate Phase 1 task breakdown
   - Detailed task list for 7-state audit engine
   - Assignable work items with acceptance criteria
   - Dependency tracking between tasks

4. **Implementation**: Begin Phase 1 (Weeks 2-3)
   - Start with audit engine state machine
   - Implement document size validation
   - Build style tree view

**Questions for Stakeholder Decision**:

1. **MVP Scope**: Ship after Phase 4 (7 weeks, P1+P2 only) or full v1.0 (11 weeks, all features)?
2. **Testing Philosophy**: Set up comprehensive unit test coverage in Phase 0, or continue manual testing?
3. **Virtualization Library**: Any preference between `react-window` (mature) vs `@tanstack/react-virtual` (modern)?
4. **Document Size Limit**: Keep 25k hard limit, or build "batch audit by page" feature for larger documents?

---

**Plan Status**: ✅ Ready for Phase 0 execution
**Constitution Compliance**: ✅ All 6 principles upheld
**Next Command**: `/speckit.tasks` (after Phase 0 research complete)
