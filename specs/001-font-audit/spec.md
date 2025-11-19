# Feature Specification: Font Audit Plugin

**Feature Branch**: `001-font-audit`
**Created**: 2025-11-16
**Updated**: 2025-11-16 (v2.0 - Added text style detection and analysis)
**Status**: Draft
**Input**: Figma plugin that audits font usage and text style compliance in design files

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Complete Font and Text Style Discovery (Priority: P1)

As a design system maintainer, I need to scan my entire Figma file to discover all fonts and text styles used across all text layers, including those in component hierarchies, hidden layers, and nested instances, so that I can ensure complete visibility into typography usage and style compliance.

**Why this priority**: This is the core value proposition - comprehensive font and text style discovery is the foundation for all other features and directly addresses the primary user need for design system governance.

**Independent Test**: Can be fully tested by running an audit on a test file with 50+ text layers using various text styles from different libraries, including unstyled text, and verifying that all text layers are discovered with accurate font and style metadata.

**Acceptance Scenarios**:

1. **Given** a Figma file with 100 text layers across main components, instances, and plain frames, **When** user clicks "Run Audit", **Then** all 100 text layers are discovered and displayed with font family, weight, size, and text style information
2. **Given** a file with text in hidden layers and collapsed component variants, **When** audit runs, **Then** hidden text is discovered and marked with a "hidden" indicator
3. **Given** nested component instances (Button → Card → Page), **When** audit runs, **Then** component hierarchy path is captured (e.g., "Page → Card → Button → Label")
4. **Given** text with overridden font properties in component instances, **When** audit runs, **Then** override status is correctly identified and displayed
5. **Given** text layers using text styles from different libraries (e.g., "Body/Large" from "Core Design System"), **When** audit runs, **Then** text style name and library source are captured and displayed
6. **Given** text layers with no text style applied, **When** audit runs, **Then** these layers are flagged as "unstyled" in the results
7. **Given** text that partially matches a text style (some properties match, others differ), **When** audit runs, **Then** the layer is flagged with "partial match" indicator and shows which properties differ

---

### User Story 2 - Search and Filter by Fonts and Styles (Priority: P2)

As a designer reviewing audit results, I need to search and filter the discovered fonts and text styles by name, library source, component type, or compliance status, so that I can quickly identify specific typography issues without manually scanning through hundreds of results.

**Why this priority**: With comprehensive discovery returning potentially hundreds of items across multiple libraries, search/filter is essential for making the data actionable and enabling rapid issue identification.

**Independent Test**: Can be tested by running an audit that returns 200+ text layers using text styles from 3+ libraries, then testing search queries (e.g., "Roboto", "Body/Large", "Core Design System") and filters (e.g., "show only unstyled", "library X only") to verify results update in <200ms and match expected criteria.

**Acceptance Scenarios**:

1. **Given** audit results with 200 text layers, **When** user types "Inter" in search box, **Then** results filter to show only layers using Inter font family in <200ms
2. **Given** results containing styled and unstyled text, **When** user applies "unstyled only" filter, **Then** only text layers without text styles are displayed
3. **Given** results with text from multiple libraries, **When** user filters by library name (e.g., "Core Design System"), **Then** only text using styles from that library is shown
4. **Given** results with partial style matches, **When** user filters by "partial matches", **Then** only text with partial style matches is displayed
5. **Given** an active search filter, **When** user clears search, **Then** full results are restored instantly

---

### User Story 3 - Navigate to Layers (Priority: P2)

As a designer reviewing audit results, I need to click on any discovered text layer in the audit view to navigate directly to that layer in Figma, so that I can quickly inspect and fix typography issues in context.

**Why this priority**: Actionability is critical - users need to go from discovery to remediation efficiently. This feature bridges the audit data to actual design work.

**Independent Test**: Can be tested by selecting various items in the audit results (main component text, instance text, hidden text) and verifying that each click focuses the corresponding layer in Figma's canvas and layer tree.

**Acceptance Scenarios**:

1. **Given** audit results displayed, **When** user clicks on a text layer entry, **Then** Figma focuses that layer in the canvas and layer tree
2. **Given** a hidden text layer in results, **When** user clicks on it, **Then** Figma navigates to the layer and reveals it while focused (layer remains visible until user clicks elsewhere or navigates away, then returns to hidden state)
3. **Given** text in a nested component instance, **When** user clicks on it, **Then** Figma expands the component hierarchy and focuses the text layer

---

### User Story 4 - Text Style Match Suggestions (Priority: P2)

As a design system maintainer, I need to see suggested text style matches for unstyled text (80%+ similar styles), so that I can identify opportunities to improve design system compliance and reduce manual style comparisons.

**Why this priority**: Identifying close style matches helps teams improve design system adoption by highlighting text that should be using styles, reducing manual comparison work.

**Independent Test**: Can be tested by creating a test file with unstyled text that closely matches existing text styles (e.g., same font family, size, weight) and verifying the audit suggests appropriate style matches with similarity indicators.

**Acceptance Scenarios**:

1. **Given** unstyled text with properties matching 90% of a text style, **When** audit runs, **Then** the layer displays a "close match" indicator with the suggested style name
2. **Given** unstyled text matching multiple styles at 80%+ similarity, **When** audit displays results, **Then** all close matches are shown ranked by similarity percentage
3. **Given** unstyled text with no close matches (<80% similarity to any style), **When** audit runs, **Then** no style suggestions are displayed for that layer
4. **Given** a close match suggestion displayed, **When** user views details, **Then** system shows which properties match and which differ from the suggested style

---

### User Story 5 - Export Comprehensive Audit Report (Priority: P3)

As a design system maintainer, I need to export the audit results as a PDF report with summary metrics, text style coverage analysis, detailed tables, and metadata, so that I can share findings with stakeholders and maintain documentation for compliance tracking.

**Why this priority**: Professional documentation is valuable for design system governance and stakeholder communication, but the core audit functionality delivers value without it.

**Independent Test**: Can be tested by running an audit on a file with styled and unstyled text from multiple libraries, clicking "Export PDF", and verifying the generated PDF contains all expected sections including text style analysis and matches audit data.

**Acceptance Scenarios**:

1. **Given** completed audit with 50 text layers using various text styles, **When** user clicks "Export PDF", **Then** PDF is generated within 5 seconds with summary metrics, text style coverage, font inventory table, and component breakdown
2. **Given** audit with styled, unstyled, and partially styled text, **When** PDF is exported, **Then** report includes sections for text style usage by library, unstyled text analysis, and partial style matches
3. **Given** audit with close style match suggestions, **When** PDF is exported, **Then** report includes table showing unstyled text with their suggested style matches
4. **Given** an exported PDF, **When** stakeholder opens it, **Then** report includes style coverage visualization, timestamp, file name, and library counts

---

## Clarifications

### Session 2025-11-16

- Q: How should the plugin handle text layer content and audit data from a privacy/storage perspective? → A: All processing in-memory only; no data persisted outside Figma; export only on explicit user action (PDF/CSV)
- Q: Which properties should be included in the similarity calculation and how should they be weighted? → A: Weighted: font family (30%), size (30%), weight (20%), line height (15%), color (5%) - prioritizes typography over color
- Q: What is the maximum number of text layers the plugin should support before displaying a performance warning or limitation? → A: Hard limit: 10,000 layers maximum - display error and suggest splitting file if exceeded
- Q: How should the plugin handle Figma API rate limiting or temporary failures during an audit? → A: Retry with backoff: Attempt up to 3 retries with exponential backoff (1s, 2s, 4s) before failing; continue audit for successful nodes
- Q: What does "temporarily reveals" mean for hidden layer navigation? → A: Focus-based: Reveal while layer is selected/focused; hide when user clicks elsewhere or navigates away

---

### Edge Cases

- What happens when a text layer has mixed font properties (multiple fonts in one text block)? System should capture the primary/first font and flag as "mixed fonts" if multiple fonts detected.
- How does the system handle corrupted or inaccessible text nodes? Display error indicator in results with message "Unable to read layer properties" and continue audit.
- What happens when user runs audit on an empty selection or file with no text? Display empty state message: "No text layers found. Try selecting frames with text or running on the entire page."
- How does the system handle extremely large files (5000+ text layers)? Implement incremental loading (display results in batches of 500) with progress indicator. If file contains >10,000 text layers, display error message: "File too large (X layers detected). Maximum supported: 10,000 layers. Please audit a smaller selection or split into multiple files."
- What happens when Figma API rate limits are hit or temporary failures occur? Implement retry logic with exponential backoff (3 retries at 1s, 2s, 4s intervals). If all retries fail, mark the node as "Unable to read layer properties" and continue audit. If text properties are unavailable after retries, gracefully degrade by showing available metadata and logging warning for missing properties.
- What happens when a text layer has a detached or deleted text style? Display "Detached style" or "Missing style" indicator and capture the last known style name if available.
- How does the system handle text styles from external libraries that are unavailable? Display library name with "unavailable" indicator and show captured style properties.
- What happens when multiple text styles have 80%+ similarity to unstyled text? Display all matches ranked by similarity percentage (highest first) with maximum of 5 suggestions.
- How does the system handle text with local style overrides on top of applied text styles? Flag as "styled with overrides" and show which properties are overridden.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: System MUST traverse all node types (frames, groups, components, instances, auto-layout containers) to discover text layers without omissions
- **FR-002**: System MUST capture font family, weight, size, line height, color (hex/rgba), opacity, parent type, component path, override status, visibility state, and text style information for each text layer
- **FR-003**: System MUST detect and capture text style assignments including full style name and library source (e.g., "Body/Large" from "Core Design System")
- **FR-004**: System MUST identify partial style matches where text properties partially match a text style, and indicate which properties match vs differ
- **FR-005**: System MUST suggest close style matches for unstyled text (80%+ property similarity calculated using weighted formula: font family 30%, size 30%, weight 20%, line height 15%, color 5%) and rank suggestions by similarity percentage
- **FR-006**: System MUST identify and distinguish between main component text, instance text, and plain text layers
- **FR-007**: System MUST detect and flag hidden text layers with clear visual indicator (e.g., opacity styling or "hidden" badge)
- **FR-008**: System MUST display results in hierarchical views supporting multiple grouping options: by font family → weight → size, by text style → library → usage, and by style compliance (styled/unstyled/partially styled)
- **FR-009**: System MUST provide search functionality that filters results by font name, text style name, library name, component name, or compliance status in <200ms
- **FR-010**: System MUST implement click-to-navigate functionality that focuses the selected text layer in Figma's canvas and layer tree
- **FR-011**: System MUST complete audit of 1000 text layers in <10 seconds including text style analysis and similarity calculations
- **FR-012**: System MUST display progress indicator for operations exceeding 2 seconds
- **FR-013**: System MUST generate PDF export containing summary metrics, text style coverage analysis, font inventory table, text style usage by library, unstyled text analysis, partial style matches, close match suggestions, component analysis, override analysis, hidden text inventory, and timestamp
- **FR-014**: System MUST handle corrupted or inaccessible nodes gracefully without stopping the audit, implementing retry logic with exponential backoff (up to 3 retries at 1s, 2s, 4s intervals) for API failures before marking node as failed
- **FR-015**: System MUST display summary dashboard showing total unique fonts, total text layers analyzed, text style coverage percentage (styled vs unstyled), libraries in use count, potential style matches found, component coverage stats, and scan timestamp
- **FR-016**: System MUST display visual indicators for text style assignment status (styled/unstyled/partial match), library source badges, partial match warnings, and close match suggestions
- **FR-017**: System MUST validate text layer count before processing and display error message if selection contains >10,000 text layers, suggesting user audit a smaller selection or split the file

### Non-Functional Requirements

**Security & Privacy:**
- **NFR-001**: System MUST process all audit data in-memory only with no persistent storage outside Figma's native storage
- **NFR-002**: System MUST only export data (PDF/CSV) on explicit user action
- **NFR-003**: System MUST NOT transmit text content or audit data to external servers

### Key Entities

- **Text Layer**: Represents a single text node in Figma with properties: font family, weight, size, line height, color, opacity, content preview (first 50 chars), node ID, parent node reference, text style assignment (if any)
- **Text Style Assignment**: Information about applied text style including: style name, library source name, assignment status (fully styled/partially styled/unstyled), property match details (which properties match/differ from style definition)
- **Style Match Suggestion**: Recommended text style for unstyled text including: suggested style name, library source, similarity percentage (80-100% calculated using weighted formula: font family 30%, size 30%, weight 20%, line height 15%, color 5%), property comparison (matching and differing properties)
- **Component Context**: Metadata describing the component hierarchy: component type (main/instance/plain), component path (breadcrumb), override status (default/overridden/detached), visibility state (visible/hidden)
- **Audit Result**: Collection of discovered text layers with metadata, organized hierarchically with multiple view options (font family → weight → size, text style → library → usage, compliance status), including summary statistics (unique font count, total layers, style coverage percentage, libraries in use, potential matches count, scan timestamp, file name)
- **Font Group**: Aggregation of text layers sharing the same font family and weight, used for hierarchical display and grouping logic
- **Style Library**: Reference to a Figma library containing text styles, including: library name, text styles available, usage count in current file

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Users can complete a full font and text style audit of a 100-page Figma file in under 30 seconds from clicking "Run Audit" to viewing results with style analysis
- **SC-002**: System discovers 100% of text layers and correctly identifies text style assignments in test files with zero false negatives (<1% acceptable false negative rate in production based on user reports)
- **SC-003**: Users can locate and navigate to any specific font usage or text style in under 10 seconds using search and click-to-navigate features
- **SC-004**: Design system maintainers report 80% reduction in time spent manually checking font and text style compliance compared to manual inspection
- **SC-005**: Plugin maintains responsive UI during audits including style similarity calculations - users can interact with Figma without lag or blocking
- **SC-006**: Search and filter operations (including style library filtering) return results in under 200ms for result sets up to 1000 items
- **SC-007**: Style match suggestions achieve 90%+ accuracy rate (users confirm suggested matches are appropriate) for unstyled text with close style matches
- **SC-008**: Exported PDF reports contain accurate data including text style coverage analysis matching the audit results with 100% fidelity
- **SC-009**: 95% of audits complete successfully without errors or crashes when tested against diverse real-world Figma files with multiple text style libraries
- **SC-010**: Users can identify all unstyled text in a file and view suggested style matches in under 15 seconds using filters and match indicators

## Assumptions

- Users have basic familiarity with Figma's layer structure, component concepts, and text styles
- Figma files follow standard practices (components aren't excessively nested beyond 10 levels)
- Users run audits on files they have read access to (no permission errors expected)
- Text style information is accessible through Figma Plugin API for all styles in use
- PDF export uses standard jsPDF library with acceptable file size (<5MB for typical audits)
- Most audit runs will analyze between 50-2000 text layers (typical design file scope), with hard limit at 10,000 layers
- Users access the plugin from Figma desktop app or web browser (not mobile)
- Font metadata available through Figma Plugin API is sufficient (no need for external font APIs)
- Text color captured as hex/rgba is sufficient (no special color space requirements)
- Files using text styles typically have 1-10 libraries enabled with 20-100 text styles total
- Weighted similarity calculation (font family 30%, size 30%, weight 20%, line height 15%, color 5%) at 80%+ threshold provides useful style suggestions
- Users understand text style concepts and the difference between styled and unstyled text
- Audit data retention is not required between plugin sessions (in-memory processing only)

## Future Scope (Out of v1.0)

The following features are planned for future releases but explicitly excluded from v1.0:

**v1.1 - Text Style and Font Replacement:**
- Bulk text style replacement across selected layers
- Font family replacement for multiple text layers
- Automatic style assignment for unstyled text with close matches
- Version history checkpoint creation before bulk operations
- Preview mode for changes before applying

**v1.2 - Design Tokens & Variables:**
- Typography variable detection and display
- Color variable usage in text
- Font family variables (when available)
- Variable replacement and migration features

**v2.0 - Enterprise Features:**
- Compliance rule configuration and enforcement
- Shareable web reports with unique URLs
- Historical tracking and comparison
- Multi-file batch processing
- Team-wide compliance dashboards
- CI/CD integration
