# Tasks: Font Audit Plugin

**Input**: Design documents from `/specs/001-font-audit/`
**Prerequisites**: plan.md (required), spec.md (required for user stories), research.md, data-model.md, contracts/plugin-protocol.md

**Tests**: This specification does NOT request test implementation. Test tasks are excluded per requirements.

**Organization**: Tasks are grouped by user story to enable independent implementation and testing of each story.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (e.g., US1, US2, US3)
- Include exact file paths in descriptions

## Path Conventions

This is a Figma plugin with dual-context architecture:
- `src/main/` - Plugin main code (Figma sandbox context)
- `src/ui/` - React UI (iframe context)
- `src/shared/` - Shared types and protocol
- `src/export/` - PDF/CSV generation utilities

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Project initialization and basic structure

- [ ] T001 Initialize Figma plugin project using `npx create-figma-plugin` with TypeScript and React
- [ ] T002 [P] Configure package.json with dependencies: Figma Plugin DS, Tailwind CSS, jsPDF, Papa Parse
- [ ] T003 [P] Setup Tailwind CSS configuration in tailwind.config.js with Figma Plugin DS token mapping
- [ ] T004 [P] Configure Vite build for dual-context plugin in vite.config.ts
- [ ] T005 [P] Setup ESLint and Prettier configuration files (.eslintrc.js, .prettierrc)
- [ ] T006 Create directory structure per plan.md: src/{main,ui,shared,export} with subdirectories
- [ ] T007 Configure TypeScript strict mode in tsconfig.json for main and UI contexts
- [ ] T008 Update manifest.json with plugin permissions (current page read, selection access, UI capabilities)

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Core infrastructure that MUST be complete before ANY user story can be implemented

**⚠️ CRITICAL**: No user story work can begin until this phase is complete

- [ ] T009 [P] Define shared type definitions in src/shared/types.ts: TextLayerData, StyleAssignment, ComponentContext, LineHeight, RGBA
- [ ] T010 [P] Define message protocol types in src/shared/types.ts: UIToMainMessage, MainToUIMessage unions
- [ ] T011 [P] Define AuditResult and AuditSummary types in src/shared/types.ts
- [ ] T012 [P] Define StyleMatchSuggestion and PropertyMatchMap types in src/shared/types.ts
- [ ] T013 Create main entry point skeleton in src/main/code.ts with postMessage handler structure
- [ ] T014 Create UI entry point skeleton in src/ui/App.tsx with React root and message listener
- [ ] T015 Implement useMessageHandler hook in src/ui/hooks/useMessageHandler.ts for main-to-UI messages
- [ ] T016 Implement useAuditState hook in src/ui/hooks/useAuditState.ts for audit state management
- [ ] T017 [P] Setup global CSS with Figma Plugin DS tokens in src/ui/styles/globals.css
- [ ] T018 [P] Create utility for building component hierarchy paths in src/main/utils/hierarchy.ts
- [ ] T019 [P] Create exponential backoff retry utility in src/main/utils/retry.ts (1s, 2s, 4s intervals per FR-014)

**Checkpoint**: Foundation ready - user story implementation can now begin in parallel

---

## Phase 3: User Story 1 - Complete Font and Text Style Discovery (Priority: P1) 🎯 MVP

**Goal**: Discover 100% of text layers across all node types with complete font, style, and component hierarchy metadata

**Independent Test**: Run audit on test file with 50+ text layers using various text styles from different libraries, verify all layers discovered with accurate metadata

### Implementation for User Story 1

- [ ] T020 [P] [US1] Implement text node traversal using findAllWithCriteria in src/main/audit/traversal.ts (enable skipInvisibleInstanceChildren per research.md)
- [ ] T021 [P] [US1] Implement font metadata extraction in src/main/audit/text-analyzer.ts (fontFamily, fontSize, fontWeight, lineHeight, color, opacity)
- [ ] T022 [US1] Implement text style detection logic in src/main/audit/text-analyzer.ts (getRangeTextStyleId, getStyleById, handle figma.mixed)
- [ ] T023 [US1] Implement style assignment status calculation in src/main/audit/text-analyzer.ts (fully-styled/partially-styled/unstyled)
- [ ] T024 [US1] Implement property match detection for partial styles in src/main/audit/text-analyzer.ts (PropertyMatchMap calculation)
- [ ] T025 [US1] Implement batch font loading optimization in src/main/utils/font-loader.ts (collect unique fonts, parallel loadFontAsync)
- [ ] T026 [US1] Implement library source detection in src/main/audit/text-analyzer.ts (local vs remote styles, extract library name)
- [ ] T027 [US1] Integrate traversal + analysis in src/main/code.ts handleRunAudit function with progress updates (every 100 nodes)
- [ ] T028 [US1] Implement 10,000 layer validation check in src/main/code.ts (FR-017 error message if exceeded)
- [ ] T029 [US1] Implement AuditSummary calculation in src/main/code.ts (totalTextLayers, uniqueFontFamilies, styleCoveragePercent, librariesInUse, hiddenLayersCount)
- [ ] T030 [US1] Create SummaryDashboard component in src/ui/components/SummaryDashboard.tsx displaying all summary metrics
- [ ] T031 [US1] Create AuditResults component skeleton in src/ui/components/AuditResults.tsx with hierarchical view structure
- [ ] T032 [US1] Implement incremental loading in src/ui/components/AuditResults.tsx (display 500 items, "Load More" button per research.md)
- [ ] T033 [US1] Create LayerItem component in src/ui/components/LayerItem.tsx displaying text layer metadata with visual indicators
- [ ] T034 [US1] Implement visual badges in src/ui/components/LayerItem.tsx (styled/unstyled/partial match indicators, library source badges per FR-016)
- [ ] T035 [US1] Create ProgressIndicator component in src/ui/components/ProgressIndicator.tsx showing percentage and counts
- [ ] T036 [US1] Integrate progress updates in src/ui/App.tsx (display ProgressIndicator during audit)
- [ ] T037 [US1] Implement error handling UI in src/ui/App.tsx (display AUDIT_ERROR messages with errorType-specific styling)
- [ ] T038 [US1] Add empty state handling in src/ui/components/AuditResults.tsx ("No text layers found" message per edge cases)

**Checkpoint**: At this point, User Story 1 should be fully functional - complete discovery with summary dashboard and basic results display

---

## Phase 4: User Story 2 - Search and Filter by Fonts and Styles (Priority: P2)

**Goal**: Enable rapid filtering of audit results by font name, style name, library source, or compliance status in <200ms

**Independent Test**: Run audit returning 200+ layers from 3+ libraries, test search queries and filters verify <200ms response

### Implementation for User Story 2

- [ ] T039 [P] [US2] Create SearchFilter component in src/ui/components/SearchFilter.tsx with text input and filter dropdown UI
- [ ] T040 [US2] Implement in-memory search logic in src/ui/components/SearchFilter.tsx (filter by font family, style name, library name)
- [ ] T041 [US2] Implement filter options in src/ui/components/SearchFilter.tsx (unstyled only, partial matches, by library, by component type)
- [ ] T042 [US2] Add search debouncing in src/ui/components/SearchFilter.tsx (optimize for <200ms response per FR-009)
- [ ] T043 [US2] Integrate SearchFilter with AuditResults in src/ui/components/AuditResults.tsx (pass filtered results to LayerItem list)
- [ ] T044 [US2] Implement filter state management in src/ui/hooks/useAuditState.ts (track active filters, clear functionality)
- [ ] T045 [US2] Add "Clear search" button in src/ui/components/SearchFilter.tsx (restore full results instantly)
- [ ] T046 [US2] Add result count indicator in src/ui/components/SearchFilter.tsx (e.g., "Showing 25 of 200 layers")

**Checkpoint**: At this point, User Stories 1 AND 2 should both work - full discovery + rapid search/filter

---

## Phase 5: User Story 3 - Navigate to Layers (Priority: P2)

**Goal**: Click any text layer in results to navigate directly to that layer in Figma canvas and layer tree

**Independent Test**: Select various items in results (main component, instance, hidden text) and verify each click focuses the layer in Figma

### Implementation for User Story 3

- [ ] T047 [US3] Implement NAVIGATE_TO_LAYER message handler in src/main/code.ts (getNodeByIdAsync, set selection, scrollAndZoomIntoView)
- [ ] T048 [US3] Add click handler to LayerItem component in src/ui/components/LayerItem.tsx (send NAVIGATE_TO_LAYER message)
- [ ] T049 [US3] Implement hidden layer reveal logic in src/main/code.ts (layer stays visible while focused per spec clarification)
- [ ] T050 [US3] Add navigation success/error feedback in src/ui/App.tsx (toast notification or visual checkmark per FR-010)
- [ ] T051 [US3] Handle navigation errors in src/main/code.ts (deleted layers, wrong page, permission issues)
- [ ] T052 [US3] Add hover effect to LayerItem in src/ui/components/LayerItem.tsx (indicate clickability)

**Checkpoint**: At this point, User Stories 1, 2, AND 3 should work - discovery + search + navigation

---

## Phase 6: User Story 4 - Text Style Match Suggestions (Priority: P2)

**Goal**: Suggest text styles for unstyled text with 80%+ similarity using weighted algorithm (30/30/20/15/5)

**Independent Test**: Create test file with unstyled text matching existing styles at 90%+, verify suggestions appear with similarity indicators

### Implementation for User Story 4

- [ ] T053 [P] [US4] Implement font family similarity calculation in src/main/audit/style-matcher.ts (exact match: 1.0, else 0.0)
- [ ] T054 [P] [US4] Implement font size similarity calculation in src/main/audit/style-matcher.ts (linear scale, 50%+ diff = 0.0)
- [ ] T055 [P] [US4] Implement font weight similarity calculation in src/main/audit/style-matcher.ts (400-point max diff per research.md)
- [ ] T056 [P] [US4] Implement line height similarity calculation in src/main/audit/style-matcher.ts (normalize PIXELS/PERCENT/AUTO to pixels)
- [ ] T057 [P] [US4] Implement color similarity calculation in src/main/audit/style-matcher.ts (Euclidean distance in RGB space)
- [ ] T058 [US4] Implement weighted similarity aggregation in src/main/audit/style-matcher.ts (30/30/20/15/5 weights, return 0-1 score)
- [ ] T059 [US4] Implement style library enumeration in src/main/audit/style-matcher.ts (collect all available text styles)
- [ ] T060 [US4] Implement suggestion generation in src/main/audit/style-matcher.ts (compare unstyled text to all styles, filter ≥80%, rank by score, max 5 per FR edge case)
- [ ] T061 [US4] Integrate style matching into audit flow in src/main/code.ts (call style-matcher for unstyled text only)
- [ ] T062 [US4] Build DifferingProperty array in src/main/audit/style-matcher.ts (property name, text value, style value)
- [ ] T063 [US4] Add close match indicator to LayerItem in src/ui/components/LayerItem.tsx (badge for unstyled text with suggestions)
- [ ] T064 [US4] Create style match detail view in src/ui/components/LayerItem.tsx (expandable section showing suggestions with similarity %)
- [ ] T065 [US4] Display property comparison in style match view in src/ui/components/LayerItem.tsx (matching properties in green, differing in orange)
- [ ] T066 [US4] Update potentialMatchesCount in summary calculation in src/main/code.ts (count unstyled with matchSuggestions.length > 0)

**Checkpoint**: All P2 user stories complete - discovery + search + navigation + style suggestions

---

## Phase 7: User Story 5 - Export Comprehensive Audit Report (Priority: P3)

**Goal**: Generate PDF report with 7 sections: summary, style coverage, font inventory, unstyled analysis, partial matches, component breakdown, hidden text

**Independent Test**: Run audit on file with styled/unstyled text from multiple libraries, export PDF, verify all sections match audit data

### Implementation for User Story 5

- [ ] T067 [P] [US5] Create PDF export data transformer in src/export/pdf-generator.ts (convert AuditResult to PDFExportData structure)
- [ ] T068 [US5] Implement jsPDF lazy loading in src/export/pdf-generator.ts (dynamic import per research.md)
- [ ] T069 [US5] Generate PDF Section 1 - Summary Dashboard in src/export/pdf-generator.ts (metrics table from AuditSummary)
- [ ] T070 [US5] Generate PDF Section 2 - Text Style Usage by Library in src/export/pdf-generator.ts (StyleUsageEntry table)
- [ ] T071 [US5] Generate PDF Section 3 - Font Inventory in src/export/pdf-generator.ts (FontInventoryEntry table with families, weights, sizes)
- [ ] T072 [US5] Generate PDF Section 4 - Unstyled Text Analysis in src/export/pdf-generator.ts (filter unstyled layers, display with suggestions)
- [ ] T073 [US5] Generate PDF Section 5 - Partial Style Matches in src/export/pdf-generator.ts (filter partially-styled, show property differences)
- [ ] T074 [US5] Generate PDF Section 6 - Component Breakdown in src/export/pdf-generator.tsx (ComponentBreakdownEntry table by type)
- [ ] T075 [US5] Generate PDF Section 7 - Hidden Text Inventory in src/export/pdf-generator.ts (filter visible:false layers)
- [ ] T076 [US5] Add PDF metadata in src/export/pdf-generator.ts (timestamp, file name, plugin version per FR-013)
- [ ] T077 [US5] Implement style coverage visualization in src/export/pdf-generator.ts (bar chart or pie chart using jsPDF drawing)
- [ ] T078 [P] [US5] Create CSV export generator in src/export/csv-generator.ts using Papa Parse (flat table per CSVRow structure)
- [ ] T079 [US5] Add "Export PDF" button to UI in src/ui/components/AuditResults.tsx (trigger export, download blob)
- [ ] T080 [US5] Add "Export CSV" button to UI in src/ui/components/AuditResults.tsx (trigger CSV export)
- [ ] T081 [US5] Implement export progress indicator in src/ui/components/AuditResults.tsx (show generating... for large exports)
- [ ] T082 [US5] Add export error handling in src/export/pdf-generator.ts (file size check <5MB per assumptions, display warning if exceeded)

**Checkpoint**: All user stories complete - full feature set including professional exports

---

## Phase 8: Polish & Cross-Cutting Concerns

**Purpose**: Improvements that affect multiple user stories

- [ ] T083 [P] Add hierarchical grouping options to AuditResults in src/ui/components/AuditResults.tsx (by font family→weight→size, by text style→library, by compliance status per FR-008)
- [ ] T084 [P] Implement view switcher in src/ui/components/AuditResults.tsx (toggle between grouping modes)
- [ ] T085 Add cancel audit functionality in src/main/code.ts (cancelFlag check after each node, send AUDIT_ERROR on cancel)
- [ ] T086 Add "Cancel" button to ProgressIndicator in src/ui/components/ProgressIndicator.tsx (send CANCEL_AUDIT message)
- [ ] T087 [P] Optimize progress update throttling in src/main/code.ts (every 100 nodes OR 500ms per research.md)
- [ ] T088 [P] Add edge case handling for mixed fonts in src/main/audit/text-analyzer.ts (detect figma.mixed, set fontFamily to "Mixed Fonts")
- [ ] T089 [P] Add edge case handling for corrupted nodes in src/main/audit/text-analyzer.ts (display "Unable to read layer properties", continue audit)
- [ ] T090 [P] Add edge case handling for detached/missing styles in src/main/audit/text-analyzer.ts (display "Detached style" or "Missing style" indicator)
- [ ] T091 Implement API retry logic integration in src/main/audit/text-analyzer.ts (use retry utility from T019 for font loading failures)
- [ ] T092 [P] Add accessibility attributes to UI components (ARIA labels, keyboard navigation support)
- [ ] T093 [P] Create README.md at repository root with setup instructions, development commands, plugin description
- [ ] T094 Validate against quickstart.md instructions (ensure setup works from scratch)
- [ ] T095 [P] Code cleanup and formatting pass (ESLint + Prettier on all files)
- [ ] T096 [P] Performance validation (test <10s for 1000 layers, <200ms search per constitution)

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies - can start immediately
- **Foundational (Phase 2)**: Depends on Setup completion - BLOCKS all user stories
- **User Stories (Phase 3-7)**: All depend on Foundational phase completion
  - US1 (P1) can start after Phase 2
  - US2 (P2) depends on US1 (requires results to filter)
  - US3 (P2) depends on US1 (requires results to navigate from)
  - US4 (P2) can run in parallel with US2/US3 after US1 complete
  - US5 (P3) depends on US1-4 complete (exports all features)
- **Polish (Phase 8)**: Depends on desired user stories being complete

### User Story Dependencies

- **US1 (P1)**: Foundation - MUST be complete first
- **US2 (P2)**: Requires US1 results but independently testable
- **US3 (P2)**: Requires US1 results but independently testable
- **US4 (P2)**: Can start after US1, independent of US2/US3
- **US5 (P3)**: Benefits from all features but can work with partial implementation

### Parallel Opportunities

- Phase 1: T002, T003, T004, T005 can run in parallel
- Phase 2: T009-T012, T017-T019 can run in parallel (type definitions + utilities)
- US1: T020-T021 can start in parallel (traversal + analysis are independent initially)
- US4: T053-T057 can run in parallel (all property similarity calculations are independent)
- US5: T067, T078 can run in parallel (PDF and CSV generators are independent)
- Polish: Most tasks marked [P] can run in parallel

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Complete Phase 1: Setup
2. Complete Phase 2: Foundational (CRITICAL - blocks all stories)
3. Complete Phase 3: User Story 1 (Font and Text Style Discovery)
4. **STOP and VALIDATE**: Test discovery on real Figma files, verify 100% coverage
5. Deploy as v0.1 (basic audit functionality)

### Incremental Delivery

1. Setup + Foundational → Foundation ready
2. Add US1 → Test independently → v0.1 MVP (Discovery only)
3. Add US2 + US3 → Test independently → v0.2 (Discovery + Search + Navigation)
4. Add US4 → Test independently → v0.3 (+ Style Suggestions)
5. Add US5 → Test independently → v1.0 (+ Professional Exports)
6. Each version adds value without breaking previous functionality

---

## Notes

- [P] tasks = different files, no dependencies, can run in parallel
- [Story] label (US1-US5) maps task to specific user story for traceability
- No test tasks included (spec does not request test implementation)
- Each user story should be independently completable and testable
- Commit after completing logical groups of tasks
- Stop at any checkpoint to validate story works independently
- Performance targets: <10s for 1000 layers (FR-011), <200ms search (FR-009), <200ms navigation (Constitution IV)
- Hard limit: 10,000 layers maximum with validation error (FR-017)
