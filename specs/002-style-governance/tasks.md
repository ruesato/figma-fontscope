# Tasks: Design System Style Governance

**Feature Branch**: `002-style-governance`
**Input**: Design documents from `/specs/002-style-governance/`
**Prerequisites**: plan.md, spec.md, constitution.md
**Target User**: Single UX designer with coding experience, leveraging AI agents

**Implementation Strategy**: This feature will be implemented by a solo designer using AI coding agents. Tasks are designed to be AI-agent-friendly with clear file paths, specific instructions, and parallelization opportunities.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (US1, US2, US3, US4, US5)
- All file paths are absolute from repository root: `/Users/ryanuesato/Documents/src/figma-fontscope/`

---

## Phase 0: Research & Setup (Week 1)

**Purpose**: Answer critical technical questions and establish foundation for implementation

**⚠️ CRITICAL**: All research must be complete before moving to implementation phases

### Research Investigations

- [x] T001 [P] [R1] Research document change detection - Test `figma.on('documentchange')` event reliability in collaborative editing. Compare with `figma.root.lastModified` timestamp approach. Document findings in specs/002-style-governance/research.md section "R1: Document Change Detection"
- [x] T002 [P] [R2] Evaluate state management options - Prototype 7-state machine in Zustand vs custom hooks. Measure bundle size impact. Document decision in specs/002-style-governance/research.md section "R2: State Management"
- [x] T003 [P] [R3] Evaluate virtualization libraries - Test `@tanstack/react-virtual` vs `react-window` with 1000+ tree nodes. Benchmark expand/collapse performance. Document recommendation in specs/002-style-governance/research.md section "R3: Virtualization"
- [x] T004 [P] [R4] Test jsPDF capabilities - Evaluate jsPDF autotable plugin for reports. Test SVG chart embedding. Verify PDF file size <5MB for 25k layers. Document findings in specs/002-style-governance/research.md section "R4: PDF Generation"
- [x] T005 [P] [R5] Document Figma Variables API - Test `figma.variables.getLocalVariables()`, token binding detection via `boundVariables`, mode information. Create code examples in specs/002-style-governance/quickstart.md section "Token Detection API"
- [x] T006 [P] [R6] Design adaptive batching algorithm - Define batch processor state machine (track size, consecutive success count, retry count). Document algorithm in specs/002-style-governance/research.md section "R6: Adaptive Batching"
- [x] T007 [P] [R7] Test version history checkpoint API - Verify `figma.saveVersionHistoryAsync(title)` creates checkpoints. Test rollback workflow. Document in specs/002-style-governance/quickstart.md section "Version History API"

### Documentation Generation

- [x] T008 Create research.md - Consolidate findings from T001-T007 into specs/002-style-governance/research.md with sections for each research question, decisions made, and rationale
- [x] T009 Create data-model.md - Define entities (TextLayer, TextStyle, DesignToken, AuditResult, LibrarySource) with attributes, relationships, and state machines in specs/002-style-governance/data-model.md
- [x] T010 [P] Create message protocol contract - Document UIToMainMessage and MainToUIMessage extensions in specs/002-style-governance/contracts/messages.md including new message types for replacement operations
- [x] T011 [P] Create audit API contract - Define AuditEngine interface (runAudit, cancelAudit, getState) in specs/002-style-governance/contracts/audit-api.md
- [x] T012 [P] Create replacement API contract - Define ReplacementEngine interface (replaceStyle, replaceToken, getBatchStatus) in specs/002-style-governance/contracts/replacement-api.md
- [x] T013 [P] Create export API contract - Define ExportEngine interface (generatePDF, generateCSV) in specs/002-style-governance/contracts/export-api.md

### Technology Decisions

- [x] T014 Make state management decision - Based on T002 research, choose between Zustand or enhanced custom hooks. Document in research.md and update package.json if Zustand chosen
- [x] T015 Make virtualization library decision - Based on T003 research, choose between `@tanstack/react-virtual` or `react-window`. Document in research.md and add dependency to package.json
- [x] T016 Verify jsPDF approach - Based on T004 research, confirm jsPDF suitable or identify alternative. Document final approach in research.md

**Checkpoint**: All research questions answered, all contracts documented, technology stack finalized

---

## Phase 1: Foundational (Weeks 2-3, Part 1)

**Purpose**: Core infrastructure that MUST be complete before ANY user story implementation

**⚠️ CRITICAL**: No user story work can begin until this phase is complete

### Type System & Entities

- [x] T017 [P] Define core entities in src/shared/types.ts - Add TextLayer, TextStyle, DesignToken, LibrarySource, AuditResult types based on data-model.md. Include state enums (AuditState, ReplacementState)
- [x] T018 [P] Define message protocol types in src/shared/types.ts - Add new UIToMainMessage union members (RUN_AUDIT, CANCEL_AUDIT, REPLACE_STYLE, REPLACE_TOKEN, EXPORT_PDF, EXPORT_CSV) and MainToUIMessage members (AUDIT_STARTED, AUDIT_PROGRESS, AUDIT_COMPLETE, AUDIT_ERROR, AUDIT_INVALIDATED, REPLACEMENT_STARTED, etc.)

### State Management Setup

- [x] T019 Implement chosen state management solution - If Zustand: create src/ui/stores/auditStore.ts and src/ui/stores/replacementStore.ts. If custom hooks: enhance src/ui/hooks/useAuditState.ts with 7-state machine logic and create src/ui/hooks/useReplacementState.ts
- [x] T020 Create document change detection hook - Implement src/ui/hooks/useDocumentChange.ts using findings from R1 research. Listen to documentchange events or poll timestamp, emit invalidation events

### Virtualization Setup

- [x] T021 Install and configure virtualization library - Based on T015 decision, add `@tanstack/react-virtual` or `react-window` to package.json. Create wrapper utility in src/ui/utils/virtualization.ts if needed for consistent API

### Base Components

- [x] T022 [P] Create ProgressIndicator component - Build src/ui/components/ProgressIndicator.tsx that displays state-specific messages (validating, scanning, processing, creating_checkpoint) with progress bar or spinner
- [x] T023 [P] Create ErrorDisplay component - Build src/ui/components/ErrorDisplay.tsx that shows error messages with Retry/Rollback/Dismiss actions based on error type
- [x] T024 [P] Create ConfirmationDialog base component - Build src/ui/components/ConfirmationDialog.tsx reusable modal for destructive operations with customizable title, message, and action buttons

### Message Handler Refactoring

- [x] T025 Refactor useMessageHandler hook - Update src/ui/hooks/useMessageHandler.ts to handle new message types from Phase 1 type definitions. Add handlers for AUDIT_INVALIDATED, REPLACEMENT_STARTED, REPLACEMENT_PROGRESS, etc.
- [x] T026 Refactor main code.ts message handler - Update src/main/code.ts to route new UI messages to appropriate engines (audit, replacement, export). Maintain existing plugin initialization logic

**Checkpoint**: Foundation ready - type system complete, state management configured, message protocol extended, base components available

---

## Phase 2: User Story 1 (P1) - Complete Style Audit (Weeks 2-3, Part 2)

**Goal**: Provide complete visibility into text style usage across entire Figma document with library source tracking

**Independent Test**: Run audit on multi-page document with local + team library styles and verify: (1) all text layers discovered, (2) styles grouped by library → hierarchy, (3) unstyled text in "Needs Styling" section, (4) audit completes <30s for 1000 layers, (5) cancel stops immediately

### Audit Engine Core

- [x] T027 [US1] Create AuditEngine state machine - Implement src/main/audit/auditEngine.ts with 7 states (idle, validating, scanning, processing, complete, error, cancelled). Include state transition methods and progress callbacks
- [x] T028 [US1] Create document validator - Implement src/main/audit/validator.ts that counts text layers and enforces size limits (warning at 5001, hard limit at 25001). Called during validating state
- [x] T029 [US1] Create page scanner - Implement src/main/audit/scanner.ts that traverses pages using existing src/main/utils/traversal.ts logic. Emits progress based on pages scanned. Called during scanning state
- [x] T030 [US1] Create metadata processor - Implement src/main/audit/processor.ts that extracts style metadata for each text layer. Resolves library names, builds hierarchy, categorizes layers. Called during processing state

### Style Detection Enhancement

- [x] T031 [US1] Fix library name resolution in styleDetection.ts - Remove placeholder at src/main/utils/styleDetection.ts lines 54-57. Use Figma Plugin API to get library name from styleId. Document approach in code comments
- [x] T032 [US1] Enhance styleLibrary.ts for team libraries - Update src/main/utils/styleLibrary.ts to enumerate team libraries via `figma.teamLibrary.getAvailableLibraryVariablesAsync()` or equivalent. Build LibrarySource objects
- [x] T033 [US1] Implement unstyled text detection - Add function to src/main/utils/styleDetection.ts that identifies text layers with no style assigned. Returns categorized list for "Needs Styling" section
- [x] T033a [US1] Track page locations for unstyled layers - Update src/main/audit/processor.ts to include page information in unstyled layer records. Store as array of { layerId, textPreview, pageName, pageId, parentType } objects. Used by T036 NeedsStylingSection to display unstyled layers grouped by page

### Summary & Analytics

- [x] T034 [US1] Refactor summary.ts for style governance - Update src/main/utils/summary.ts to calculate: style adoption rate, library usage distribution, top 10 styles, unstyled count. Remove font-specific metrics
- [x] T034a [US1] Implement parent-child hierarchy detection - Add function to src/main/utils/summary.ts that parses style names by forward-slash separators (e.g., "Typography/Heading/H1"). Build parent-child relationship map where "Typography" is parent of "Typography/Heading" which is parent of "Typography/Heading/H1". Store hierarchy structure in AuditResult.styleHierarchy for tree rendering

### UI Components for Audit

- [x] T035 [US1] Build StyleTreeView component - Create src/ui/components/StyleTreeView.tsx with library→hierarchy grouping, expandable tree nodes, usage count badges, toggle for grouping mode. Integrate virtualization from T021
- [x] T036 [US1] Build NeedsStylingSect ion in StyleTreeView - Add "Needs Styling" section at peer level with library groups in StyleTreeView. Show count and expandable list of pages
- [x] T037 [US1] Refactor SummaryDashboard for styles - Update src/ui/components/SummaryDashboard.tsx to display style adoption metrics, library distribution chart, unstyled count. Remove font-specific UI
- [x] T038 [US1] Update AuditResults component - Refactor src/ui/components/AuditResults.tsx to use StyleTreeView instead of flat list. Add library/hierarchy toggle, pass audit data from state

### Audit Orchestration

- [x] T039 [US1] Wire audit engine to message handler - Update src/main/code.ts RUN_AUDIT handler to instantiate AuditEngine, call runAudit(), emit progress messages (AUDIT_STARTED, AUDIT_PROGRESS, AUDIT_COMPLETE, AUDIT_ERROR)
- [x] T040 [US1] Implement cancel functionality - Add CANCEL_AUDIT handler in src/main/code.ts that calls AuditEngine.cancel(). Update UI to show cancel button during scanning/processing states

### Document Change Detection

- [x] T041 [US1] Implement document change invalidation - Wire useDocumentChange hook (T020) to audit state. When change detected, set audit results as stale, show warning banner, enable "Re-run Audit" button
- [x] T042 [US1] Add warning banner component - Create src/ui/components/WarningBanner.tsx that displays "Document has been modified. Audit results may be outdated." with "Re-run Audit" CTA when results invalidated

**Checkpoint**: User Story 1 complete - Full audit capability with 7-state progression, document size validation, style tree view, unstyled text detection, and change invalidation

---

## Phase 3: User Story 3 (P3) - Analyze Token Usage (Week 4)

**Goal**: Provide visibility into design token usage in text layers to assess token adoption rates

**Independent Test**: Run audit on document with token-based text layers and verify: (1) tokens view displays all tokens with names/values/counts, (2) selecting token shows affected layers grouped by page/component, (3) mixed usage (style + token) correctly identified, (4) analytics shows token coverage %

### Token Detection

- [x] T043 [P] [US3] Create tokenDetection.ts utility - Implement src/main/utils/tokenDetection.ts that uses Figma Variables API (R5 findings). Detect tokens via `boundVariables` on TextNode, resolve token metadata (name, value, type, collection, mode)
- [x] T044 [P] [US3] Extend AuditResult entity for tokens - Add `tokenInventory: DesignToken[]` field to AuditResult type in src/shared/types.ts. Include token usage tracking
- [x] T045 [US3] Integrate token detection into processor - Update src/main/audit/processor.ts to call tokenDetection.ts for each text layer. Build token inventory alongside style inventory
- [x] T046 [US3] Extend summary.ts for token metrics - Add token coverage calculation to src/main/utils/summary.ts: percentage of layers using tokens, token adoption by collection, mixed usage count

### Token View UI

- [x] T047 [P] [US3] Build TokenView component - Create src/ui/components/TokenView.tsx with tokens grouped by collection, expandable tree showing usage counts, click to show detail panel
- [x] T048 [P] [US3] Build AnalyticsDashboard component - Create src/ui/components/AnalyticsDashboard.tsx that displays style adoption rate, token coverage %, library distribution, top 10 styles, unstyled count (FR-046 through FR-050)
- [x] T049 [US3] Add Tokens tab to App.tsx - Update src/ui/App.tsx navigation to include "Styles" and "Tokens" tabs. Route between StyleTreeView and TokenView components
- [x] T050 [US3] Add Analytics tab to App.tsx - Add "Analytics" tab to src/ui/App.tsx that displays AnalyticsDashboard component with metrics from audit result

**Checkpoint**: User Story 3 complete - Token detection operational, token view displays usage, analytics dashboard shows comprehensive metrics

---

## Phase 4: Detail Panel & Navigation (Week 5)

**Goal**: Enable drill-down from tree to individual layer details with page/component grouping and canvas navigation

**Independent Test**: Select style or token in tree and verify: (1) detail panel shows all layers using it grouped by page/component, (2) clicking layer navigates to it in Figma canvas, (3) search filters 500+ styles in <200ms, (4) visual indicators show assignment status

### Detail Panel Component

- [ ] T051 [US1+US3] Build DetailPanel component - Create src/ui/components/DetailPanel.tsx that displays layers using selected style/token. Group by page → component. Use virtualization for large lists (500+ layers)
- [ ] T052 [US1+US3] Implement click-to-navigate - Add layer click handler in DetailPanel that sends NAVIGATE_TO_LAYER message to main context. Use existing message handler support for navigation
- [ ] T053 [US1+US3] Add visual status indicators - Create badge components in DetailPanel for fully-styled, partially-styled, unstyled status. Color-code for quick visual scanning

### Search & Filter

- [ ] T054 [P] [US1+US3] Build search functionality - Add search input to StyleTreeView and TokenView that filters tree nodes by name. Use native JavaScript filter (no library needed for <1000 styles)
- [ ] T055 [P] [US1+US3] Build filter dropdowns - Add filter controls to StyleTreeView: by library (dropdown), by usage count (slider/dropdown: unused/rarely used/frequently used), by assignment status (checkboxes: fully/partially/unstyled)
- [ ] T056 [US1+US3] Wire search/filter to tree - Connect search input and filter dropdowns to StyleTreeView and TokenView. Filter tree nodes based on criteria, maintain expand/collapse state

### Keyboard Navigation

- [ ] T057 [US1+US3] Implement keyboard shortcuts - Add keyboard event handlers to StyleTreeView: Space=expand/collapse, Enter=select, Arrow keys=navigate tree. Ensure focus visible
- [ ] T058 [US1+US3] Add keyboard shortcuts documentation - Add keyboard shortcut hints to UI (tooltip or help panel): "Space to expand, Enter to select, ↑↓ to navigate"

**Checkpoint**: Detail panel operational, full exploratory workflow functional (tree → detail → canvas), search/filter responsive

---

## Phase 5: User Story 2 (P2) - Migrate Styles Between Libraries (Weeks 6-7)

**Goal**: Enable bulk style replacement between libraries with safety guarantees (version checkpoints, adaptive batching, error recovery)

**Independent Test**: Select style with 100+ usages, choose replacement from different library, confirm, and verify: (1) version checkpoint created, (2) all layers updated, (3) component overrides preserved, (4) batch errors trigger size reduction, (5) rollback available on persistent failure

### Replacement Engine Core

- [ ] T059 [US2] Create ReplacementEngine state machine - Implement src/main/replacement/replacementEngine.ts with 7 states (idle, validating, creating_checkpoint, processing, complete, error). No cancelled state per FR-041a
- [ ] T060 [US2] Create checkpoint manager - Implement src/main/replacement/checkpoint.ts that wraps `figma.saveVersionHistoryAsync(title)` per R7 findings. Called during creating_checkpoint state
- [ ] T061 [US2] Create batch processor with adaptive sizing - Implement src/main/replacement/batchProcessor.ts per R6 algorithm: start at 100 layers/batch, reduce to 25 on errors, increase back to 100 after 5 successes. Track state (batch size, consecutive successes, retry count)
- [ ] T062 [US2] Create error recovery module - Implement src/main/replacement/errorRecovery.ts that classifies errors (transient, persistent, validation, partial) and implements retry with exponential backoff (1s, 2s, 4s per FR-041c)

### Validation & Safety

- [ ] T063 [US2] Implement replacement validation - Add validation function in ReplacementEngine that checks: source ≠ target, source and target both exist, affected layer count > 0, user has edit permissions. Called during validating state
- [ ] T064 [US2] Enhance retry.ts for replacement - Update src/main/utils/retry.ts to support exponential backoff timing (1s, 2s, 4s) and error classification. Used by errorRecovery.ts

### Style Picker UI

- [ ] T065 [P] [US2] Build StylePicker component - Create src/ui/components/StylePicker.tsx that displays all styles from all libraries in searchable/filterable list. Group by library. Allow single selection. Modal dialog format
- [ ] T066 [P] [US2] Build replacement ConfirmationDialog - Enhance src/ui/components/ConfirmationDialog.tsx (from T024) for replacements. Show format: "Replace [Style A] from [Library X] with [Style B] from [Library Y] in [N] text layers?"
- [ ] T067 [US2] Add "Replace" action to DetailPanel - Update src/ui/components/DetailPanel.tsx quick actions to include "Replace Style" button that opens StylePicker (T065) when style selected

### Replacement Orchestration

- [ ] T068 [US2] Wire ReplacementEngine to message handler - Add REPLACE_STYLE handler in src/main/code.ts that instantiates ReplacementEngine, calls replaceStyle(), emits progress messages (REPLACEMENT_STARTED, REPLACEMENT_PROGRESS, REPLACEMENT_CHECKPOINT_CREATED, REPLACEMENT_COMPLETE, REPLACEMENT_ERROR)
- [ ] T069 [US2] Implement replacement progress tracking - Update ProgressIndicator component to handle replacement states. Show "Creating checkpoint...", "Replacing batch X of Y...", with progress bar
- [ ] T070 [US2] Implement rollback confirmation - Add rollback dialog to ErrorDisplay component. When persistent error detected, show "Operation failed. Rollback to version checkpoint '[timestamp]'?" with Rollback/Keep Changes buttons. Send ROLLBACK_TO_CHECKPOINT message on Rollback

### Partial Failure Handling

- [ ] T071 [US2] Implement partial failure reporting - Update ReplacementEngine to collect failed layer IDs during processing. On completion with failures, emit REPLACEMENT_COMPLETE_WITH_WARNINGS message including failed layer list and reasons
- [ ] T072 [US2] Build partial failure summary UI - Add summary display in ErrorDisplay for partial failures: "Replacement completed with warnings: [X] of [Y] layers updated. [Z] layers failed: [reasons]. View failed layers?" with expand/collapse for details

### Post-Replacement Audit Invalidation

- [ ] T073 [US2] Invalidate audit after replacement - When REPLACEMENT_COMPLETE received in useMessageHandler, mark audit results as stale, show warning banner, prompt user to re-run audit to see updated state

**Checkpoint**: User Story 2 complete - Style replacement fully functional with version checkpoints, adaptive batching, retry logic, rollback capability, partial failure handling

---

## Phase 6: User Story 4 (P4) - Replace Tokens Across Document (Week 8)

**Goal**: Enable bulk token replacement with same safety model as style replacement

**Independent Test**: Select token with 40+ usages, choose replacement token, confirm, and verify: (1) version checkpoint created, (2) all token bindings updated, (3) change reflected in next audit, (4) error handling matches style replacement

### Token Replacement Extension

- [ ] T074 [US4] Extend ReplacementEngine for tokens - Add replaceToken() method to src/main/replacement/replacementEngine.ts. Reuse existing state machine, checkpoint, batching, error recovery from Phase 5
- [ ] T075 [US4] Implement token binding update - Create token replacement logic in ReplacementEngine that uses `setBoundVariable()` or equivalent Figma API per R5 findings. Update `boundVariables` on TextNode to new token ID

### Token Picker UI

- [ ] T076 [P] [US4] Build TokenPicker component - Create src/ui/components/TokenPicker.tsx (adapt StylePicker from T065). Display all tokens grouped by collection. Filter by token type. Modal dialog format
- [ ] T077 [P] [US4] Build token replacement ConfirmationDialog - Adapt ConfirmationDialog component for tokens. Show: "Replace token [Token A] with [Token B] in [N] text layers?" Include token values for clarity
- [ ] T078 [US4] Add "Replace Token" action to DetailPanel - Update src/ui/components/DetailPanel.tsx to show "Replace Token" button when token selected in TokenView

### Token Replacement Orchestration

- [ ] T079 [US4] Wire token replacement to message handler - Add REPLACE*TOKEN handler in src/main/code.ts that calls ReplacementEngine.replaceToken(), emits progress messages (reuse REPLACEMENT*\* message types)
- [ ] T080 [US4] Test token replacement safety model - Verify that token replacement follows same checkpoint → batch → retry → rollback workflow as style replacement. Test with 100+ affected layers

**Checkpoint**: User Story 4 complete - Token replacement operational with safety parity to style replacement

---

## Phase 7: User Story 5 (P5) - Export Audit Report (Week 9)

**Goal**: Generate PDF executive reports and CSV data exports for stakeholder communication and analysis

**Independent Test**: Run audit, click "Export PDF", and verify: (1) PDF contains executive summary, style inventory, adoption visualizations, document metadata, (2) CSV contains one row per layer with all metadata, (3) both include audit timestamp

### PDF Export

- [ ] T081 [P] [US5] Create PDF generator - Implement src/export/pdfGenerator.ts using jsPDF per R4 findings. Generate multi-page PDF with sections: executive summary (metrics), style inventory (tables), adoption visualizations (charts or tables), document metadata
- [ ] T082 [P] [US5] Implement PDF chart generation - Add chart generation to pdfGenerator.ts. Use jsPDF-autotable for tables, embed SVG for charts if needed. Include: style adoption chart, library distribution chart, top 10 styles
- [ ] T083 [US5] Add document metadata to PDF - Include in PDF header/footer: document filename, total page count, total text layer count, audit timestamp, plugin version

### CSV Export

- [ ] T084 [P] [US5] Create CSV generator - Implement src/export/csvGenerator.ts using PapaParse. Generate CSV with columns: layer ID, layer name, text content preview, style name, style source, token name, token value, page name, component context, assignment status
- [ ] T085 [US5] Add CSV metadata header - Include CSV header rows with document metadata before data rows: filename, audit timestamp, total layers, total styles, total tokens

### Export UI

- [ ] T086 [P] [US5] Build ExportPanel component - Create src/ui/components/ExportPanel.tsx with "Export PDF Report" and "Export CSV Data" buttons. Show file size estimates. Display success message with download link on completion
- [ ] T087 [US5] Add Export section to App.tsx - Add "Export" section to AnalyticsDashboard or separate Export view in src/ui/App.tsx. Show ExportPanel component

### Export Orchestration

- [ ] T088 [US5] Wire PDF export to message handler - Add EXPORT_PDF handler in src/main/code.ts that calls pdfGenerator.generatePDF() with audit result, creates blob URL, sends download link to UI via EXPORT_PDF_COMPLETE message
- [ ] T089 [US5] Wire CSV export to message handler - Add EXPORT_CSV handler in src/main/code.ts that calls csvGenerator.generateCSV() with audit result, creates blob URL, sends download link to UI via EXPORT_CSV_COMPLETE message
- [ ] T090 [US5] Implement file download in UI - Add download functionality in ExportPanel that receives blob URL from EXPORT\_\*\_COMPLETE messages, creates anchor element, triggers download with appropriate filename (e.g., "style-audit-2025-11-20.pdf")

### Export Performance

- [ ] T091 [US5] Test export performance at scale - Verify PDF and CSV generation completes in <60 seconds for 25k layer audit results. Optimize formatters if needed. Test browser memory usage during generation

**Checkpoint**: User Story 5 complete - Full export capability operational with PDF and CSV formats, suitable for stakeholder presentations

---

## Phase 8: Performance Optimization (Week 10)

**Goal**: Ensure Warning Zone (5k-25k layers) performance meets spec targets through virtualization and optimization

**Independent Test**: Run audit on 5000-layer test document and verify: (1) tree view with 1000+ styles renders in <500ms, (2) detail panel with 5000+ layers scrolls at 60fps, (3) total memory usage <200MB, (4) audit completes in 2-10 minutes, (5) UI remains responsive

### Virtualization Integration

- [ ] T092 Create virtualized tree for StyleTreeView - Integrate virtualization library (from T021 decision) into src/ui/components/StyleTreeView.tsx. Only render visible tree nodes to reduce DOM size. Target: <500ms render for 1000+ styles
- [ ] T093 Create virtualized list for DetailPanel - Integrate virtualization into src/ui/components/DetailPanel.tsx for layer lists. Support smooth scrolling with 5000+ layers at 60fps
- [ ] T094 Create virtualized list for TokenView - Apply virtualization to src/ui/components/TokenView.tsx token tree using same approach as StyleTreeView

### Progressive Loading

- [ ] T095 Implement progressive batch rendering - Update AuditEngine processor to display results in batches of 100 styles. Load first batch immediately, render remaining batches in background with requestIdleCallback or setTimeout
- [ ] T096 Reduce progress update frequency - Update src/main/audit/auditEngine.ts and replacementEngine.ts to emit progress updates every 50 layers (instead of every 10) for Warning Zone documents (>5000 layers)

### Algorithm Optimization

- [ ] T097 Optimize summary calculations - Update src/main/utils/summary.ts to use Map data structures for O(1) lookups instead of array iterations. Profile and optimize hot paths for 25k layer datasets
- [ ] T098 Profile memory usage - Use browser DevTools to measure memory usage during 25k layer audit. Identify and fix memory leaks. Ensure total usage <200MB per spec requirement

### Performance Testing

- [ ] T099 Create 5k layer test document - Build test Figma document with 5000 text layers across multiple pages using mix of local and team library styles. Use for performance benchmarking
- [ ] T100 Validate Warning Zone performance - Run audit on T099 test document. Measure: audit completion time (target: 2-10 min), tree render time (target: <500ms), memory usage (target: <200MB), UI responsiveness (no blocking >1s)

**Checkpoint**: Warning Zone performance meets spec targets, virtualization operational, memory usage optimized

---

## Phase 9: Polish & Edge Cases (Week 11)

**Goal**: Handle all edge cases from spec, add accessibility features, implement keyboard navigation, polish UX

**Independent Test**: Test all 28 edge cases from spec Edge Cases section and verify each handled correctly with appropriate messaging and fallback behavior

### Edge Case Handling

- [ ] T101 [P] Handle empty document edge case - Add empty state to AuditResults: "No text layers found in document". Disable audit-dependent features when no text layers detected
- [ ] T102 [P] Handle zero usage style edge case - Display styles with 0 usage count in tree with grayed-out badge "Unused". Allow visibility into available but unused styles
- [ ] T103 [P] Handle deleted/missing style edge case - Mark layers using deleted styles as "unstyled" with warning indicator. Show tooltip: "Assigned style no longer exists"
- [ ] T104 [P] Handle network issues edge case - When library access fails, show warning: "Some team libraries unavailable. Showing local styles only." Allow audit to continue with local styles
- [ ] T105 [P] Handle replacement interruption edge case - Already covered by version history checkpoint in Phase 5. Add explicit error message on interruption: "Operation interrupted. Use File > Version History to rollback."
- [ ] T106 [P] Handle same source/target validation - Add validation in ReplacementEngine (already in T063): "Source and target styles cannot be the same". Show error message in ConfirmationDialog
- [ ] T107 [P] Handle nested components edge case - Test replacement in nested component instances. Verify structure preserved and overrides maintained. Already covered by Figma API, add regression test
- [ ] T108 [P] Handle multiple team libraries - Already supported by library grouping in StyleTreeView. Verify each library shows as separate group with library name label
- [ ] T109 [P] Handle mixed token modes - Display mode information in TokenView per FR-020. Show token value for each mode if multi-mode token detected
- [ ] T110 [P] Handle document changes during audit - Already implemented in T041. Verify warning shows immediately when change detected: "Document has been modified. Audit results may be outdated."

### Accessibility

- [ ] T111 Add ARIA labels to tree nodes - Add aria-label to StyleTreeView and TokenView tree nodes: "Style [name], [count] usages, [expanded/collapsed]". Ensure screen reader announces context
- [ ] T112 Add ARIA labels to buttons - Add aria-label to all action buttons: "Run Audit", "Cancel Audit", "Replace Style", "Export PDF Report", etc. Describe action clearly for screen readers
- [ ] T113 Implement focus management in modals - Trap focus within ConfirmationDialog, StylePicker, TokenPicker when open. Restore focus to trigger element on close. Prevent keyboard trap in main UI
- [ ] T114 Test screen reader announcements - Test with VoiceOver (macOS) or NVMA (Windows). Verify state changes announced: "Audit started", "Scanning page 5 of 10", "Audit complete, 247 styles found"

### Loading States

- [ ] T115 [P] Add loading skeletons - Create skeleton components for StyleTreeView, TokenView, DetailPanel that show during initial load and state transitions. Replace with actual content when data available
- [ ] T116 [P] Add empty states for all views - Create empty state messages: "Run audit to see style inventory" (StyleTreeView), "No tokens detected" (TokenView), "Select a style to view details" (DetailPanel)

### Keyboard Navigation

- [ ] T117 Document keyboard shortcuts - Add keyboard shortcuts panel or help tooltip showing: "Space: Expand/collapse, Enter: Select, ↑↓: Navigate tree, Tab: Move between sections, Esc: Close dialogs"
- [ ] T118 Test keyboard-only workflow - Verify entire plugin can be operated with keyboard only: navigate tree, select styles, open pickers, confirm operations, export reports. No mouse required

### Final Polish

- [ ] T119 Review all error messages - Audit all error messages for clarity and actionability. Ensure each error suggests next step: "Retry", "Check permissions", "Contact support", etc.
- [ ] T120 Add version information - Display plugin version in UI footer or about panel. Useful for debugging and support requests
- [ ] T121 Test all 28 edge cases systematically - Create checklist of all edge cases from spec Edge Cases section. Test each one and verify behavior matches spec requirements
- [ ] T122 Final UX polish pass - Review all transitions, animations, hover states, focus indicators. Ensure consistent styling using Figma design tokens from tailwind.config.js

**Checkpoint**: Production-ready quality, all edge cases handled, fully accessible, keyboard navigation functional

---

## Dependencies & Execution Order

### Phase Dependencies

1. **Phase 0 (Research & Setup)**: No dependencies - MUST complete first
2. **Phase 1 (Foundational)**: Depends on Phase 0 completion - BLOCKS all user stories
3. **Phase 2 (US1)**: Depends on Phase 1 completion - Can proceed independently
4. **Phase 3 (US3)**: Depends on Phase 1 completion - Can proceed in parallel with Phase 2
5. **Phase 4 (Detail & Nav)**: Depends on Phase 2 AND Phase 3 completion (needs both style and token views)
6. **Phase 5 (US2)**: Depends on Phase 1 completion - Can start after Foundational, but benefits from Phase 4 detail panel
7. **Phase 6 (US4)**: Depends on Phase 5 completion (reuses ReplacementEngine)
8. **Phase 7 (US5)**: Depends on Phase 3 completion (needs analytics data) - Can proceed in parallel with Phase 5/6
9. **Phase 8 (Performance)**: Depends on Phase 2-7 completion (optimizes all features)
10. **Phase 9 (Polish)**: Depends on Phase 2-8 completion (final quality pass)

### User Story Independence

- **User Story 1 (P1 - Audit)**: Independent foundation - no dependencies on other stories
- **User Story 3 (P3 - Tokens)**: Builds on US1 audit engine but independently testable
- **User Story 2 (P2 - Style Replace)**: Independent of other stories, uses Phase 1 foundation
- **User Story 4 (P4 - Token Replace)**: Extends US2 replacement engine
- **User Story 5 (P5 - Export)**: Uses US1 and US3 data but independently testable

### Critical Path for Solo Designer

**Week 1**: Phase 0 (Research) → **BLOCKS EVERYTHING**
**Weeks 2-3**: Phase 1 (Foundational) → **BLOCKS ALL USER STORIES**
**Weeks 2-3**: Phase 2 (US1 - Audit) → Can start immediately after Phase 1
**Week 4**: Phase 3 (US3 - Tokens) → Can start in parallel with Phase 2 if Phase 1 done
**Week 5**: Phase 4 (Detail Panel) → Needs Phase 2 + Phase 3 complete
**Weeks 6-7**: Phase 5 (US2 - Style Replace) → Independent, highest business value
**Week 8**: Phase 6 (US4 - Token Replace) → Quick extension of Phase 5
**Week 9**: Phase 7 (US5 - Export) → Can happen earlier if desired
**Week 10**: Phase 8 (Performance) → Optimizes all previous work
**Week 11**: Phase 9 (Polish) → Final quality pass

### Parallel Opportunities for AI Agents

**Within Phase 0**: T001-T007 can ALL run in parallel (7 research tasks simultaneously)
**Within Phase 0**: T010-T013 can ALL run in parallel (4 contract documents simultaneously)
**Within Phase 1**: T017-T018, T022-T024 can run in parallel (type definitions + base components)
**Within Phase 2**: T031-T033 can run in parallel (style detection enhancements)
**Within Phase 2**: T035-T037 can run in parallel (UI components)
**Within Phase 3**: T043-T044, T047-T048 can run in parallel
**Within Phase 5**: T065-T066 can run in parallel (both UI components)
**Within Phase 7**: T081-T082, T084-T085, T086 can run in parallel
**Within Phase 9**: T101-T110 can ALL run in parallel (10 edge case handlers)

---

## Implementation Strategy

### MVP First (Fastest Time to Value)

**Goal**: Ship working audit + replacement in 7 weeks

1. Complete Phase 0 (Research) - Week 1
2. Complete Phase 1 (Foundational) - Weeks 2-3
3. Complete Phase 2 (US1 - Audit) - Weeks 2-3
4. Complete Phase 5 (US2 - Style Replace) - Weeks 6-7
5. **STOP and DEPLOY MVP**: Core value delivered (audit + migration)

### Incremental Delivery (Recommended for Solo Designer)

1. **Week 1**: Phase 0 → Foundation of decisions
2. **Weeks 2-3**: Phase 1 + Phase 2 → Working audit (US1) → **Demo checkpoint**
3. **Week 4**: Phase 3 → Add tokens (US3) → **Demo checkpoint**
4. **Week 5**: Phase 4 → Enhanced navigation → **Demo checkpoint**
5. **Weeks 6-7**: Phase 5 → Style replacement (US2) → **MVP LAUNCH**
6. **Week 8**: Phase 6 → Token replacement (US4) → **v1.1 release**
7. **Week 9**: Phase 7 → Export reports (US5) → **v1.2 release**
8. **Week 10**: Phase 8 → Performance for large docs → **v1.3 release**
9. **Week 11**: Phase 9 → Polish + edge cases → **v1.0 final**

### Using AI Agents Effectively

**Research Phase (Week 1)**:

- Launch all 7 research tasks in parallel to different agents
- Consolidate findings yourself, make technology decisions
- Generate documentation with agent assistance

**Implementation Phases**:

- Launch parallel tasks to multiple agents (marked [P])
- Sequential tasks: complete dependencies first, then launch next task
- Each task is scoped for 1-4 hours of agent time
- Review agent output, iterate if needed, move to next task

**Quality Gates**:

- End of each Phase: Test user story independently before proceeding
- Use checkpoints to validate: "Does US1 work on its own?"
- Don't proceed to next phase if previous phase broken

---

## Task Summary

- **Total Tasks**: 124
- **Research Tasks**: 16 (Phase 0)
- **Foundational Tasks**: 10 (Phase 1)
- **User Story 1 (P1)**: 18 tasks (Phase 2)
- **User Story 3 (P3)**: 8 tasks (Phase 3)
- **Detail & Navigation**: 8 tasks (Phase 4)
- **User Story 2 (P2)**: 15 tasks (Phase 5)
- **User Story 4 (P4)**: 7 tasks (Phase 6)
- **User Story 5 (P5)**: 11 tasks (Phase 7)
- **Performance**: 9 tasks (Phase 8)
- **Polish**: 22 tasks (Phase 9)

**MVP Scope** (Phases 0-1-2-5): 59 tasks = ~7 weeks
**Full v1.0**: All 124 tasks = ~11 weeks

---

## Notes for Solo Designer Using AI Agents

- **Start small**: Phase 0 research is critical - don't skip it
- **Use checkpoints**: Test each user story independently before moving on
- **Leverage parallelism**: Launch multiple [P] tasks to agents simultaneously
- **Review agent work**: AI agents are powerful but need oversight - review each output
- **Document as you go**: Update research.md, data-model.md as you learn
- **Test in Figma often**: Manual testing in real documents catches issues early
- **Commit frequently**: After each task or logical group, commit to version control
- **MVP is valuable**: Phases 0-1-2-5 deliver core value (audit + migration) in 7 weeks
- **Incremental releases**: Ship Phase 2 (audit), then Phase 5 (replacement), then remaining features
- **Constitution compliance**: Re-check constitution.md after Phase 1 design (as noted in plan.md)

---

## File Path Reference

All paths relative to repository root: `/Users/ryanuesato/Documents/src/figma-fontscope/`

**New files to create**:

- `specs/002-style-governance/research.md`
- `specs/002-style-governance/data-model.md`
- `specs/002-style-governance/quickstart.md`
- `specs/002-style-governance/contracts/messages.md`
- `specs/002-style-governance/contracts/audit-api.md`
- `specs/002-style-governance/contracts/replacement-api.md`
- `specs/002-style-governance/contracts/export-api.md`
- `src/main/audit/auditEngine.ts`
- `src/main/audit/validator.ts`
- `src/main/audit/scanner.ts`
- `src/main/audit/processor.ts`
- `src/main/replacement/replacementEngine.ts`
- `src/main/replacement/checkpoint.ts`
- `src/main/replacement/batchProcessor.ts`
- `src/main/replacement/errorRecovery.ts`
- `src/main/utils/tokenDetection.ts`
- `src/ui/components/StyleTreeView.tsx`
- `src/ui/components/TokenView.tsx`
- `src/ui/components/DetailPanel.tsx`
- `src/ui/components/StylePicker.tsx`
- `src/ui/components/AnalyticsDashboard.tsx`
- `src/ui/components/ExportPanel.tsx`
- `src/ui/components/WarningBanner.tsx`
- `src/ui/hooks/useDocumentChange.ts`
- `src/ui/hooks/useReplacementState.ts`
- `src/ui/hooks/useVirtualization.ts`
- `src/ui/utils/virtualization.ts`
- `src/export/pdfGenerator.ts`
- `src/export/csvGenerator.ts`
- `src/export/formatters.ts`

**Files to refactor**:

- `src/main/code.ts` (message handler)
- `src/main/utils/styleDetection.ts` (fix library resolution)
- `src/main/utils/styleLibrary.ts` (team library enumeration)
- `src/main/utils/summary.ts` (style governance metrics)
- `src/main/utils/retry.ts` (exponential backoff)
- `src/ui/App.tsx` (multi-view navigation)
- `src/ui/components/ProgressIndicator.tsx` (state-specific messages)
- `src/ui/components/ErrorDisplay.tsx` (retry/rollback actions)
- `src/ui/components/SummaryDashboard.tsx` (style metrics)
- `src/ui/components/AuditResults.tsx` (tree view)
- `src/ui/hooks/useAuditState.ts` (7-state model)
- `src/ui/hooks/useMessageHandler.ts` (new message types)
- `src/shared/types.ts` (new entities and messages)

**Files to deprecate/remove**:

- `src/main/utils/fontMetadata.ts` (font-specific logic removed in pivot)
