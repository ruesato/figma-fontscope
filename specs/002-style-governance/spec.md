# Feature Specification: Design System Style Governance

**Feature Branch**: `002-style-governance`
**Created**: 2025-11-20
**Status**: Draft
**Input**: User description: "Design system governance and management of text styles and design tokens across entire Figma documents"

## Overview

This feature transforms the plugin from a font auditing tool into a comprehensive design system governance platform for managing text styles and design tokens across entire Figma documents. Design system teams currently lack visibility into style adoption, cannot efficiently identify deprecated patterns, and have no scalable method to migrate between style libraries. This feature provides complete visibility and control over text style usage, enabling data-driven design system management and bulk library migrations that reduce manual work from days to minutes.

### Context

Design system managers are blind to how their text styles and tokens are actually used across documents. Current Figma capabilities provide no way to see which styles are adopted versus ignored, identify deprecated style usage at scale, or efficiently migrate from old to new style libraries. Without proper tooling, maintaining consistency across large design organizations is a manual, error-prone process with no metrics to demonstrate design system ROI.

### Strategic Direction

This represents a complete pivot from font-specific auditing to design system governance. The existing font audit codebase will be replaced with new style and token management capabilities focused on design system team workflows.

## Clarifications

### Session 2025-11-20

- Q: How should the system handle concurrent modifications to the document (e.g., user adds/deletes layers during audit, or document changes during replacement)? → A: Invalidate audit results when document changes detected, show warning, require re-audit
- Q: What states should audit and replacement operations track for UI implementation and error handling? → A: Detailed state model: idle → validating → scanning → processing → complete/error/cancelled
- Q: What should happen when documents exceed practical size limits? → A: Warning at 5,000 text layers (with performance notice), hard limit at 25,000 layers (with error message)
- Q: How should bulk replacement operations handle Figma API constraints and batching? → A: Adaptive batching - start at 100 layers/batch, reduce to 25 on errors, increase back to 100 on success
- Q: How should the system recover from mid-operation failures? → A: Auto-retry transient failures (network, timeout) 3x with exponential backoff, then rollback to version checkpoint on persistent failure

## User Scenarios & Testing _(mandatory)_

### User Story 1 - Complete Style Audit of Document (Priority: P1)

Alex, a design system manager, needs to understand how text styles from the team's design system library are being used across a 100-page product design file before deprecating old styles in favor of a new type scale.

**Why this priority**: This is the foundational capability that enables all other workflows. Without comprehensive visibility into current style usage, design system teams cannot make informed decisions about migrations, deprecations, or system improvements.

**Independent Test**: Can be fully tested by running an audit on a multi-page document and verifying that all text layers are discovered, categorized by style assignment status, and display accurate usage metrics.

**Acceptance Scenarios**:

1. **Given** a Figma document with 100+ pages containing text layers with various style assignments, **When** Alex runs the style audit, **Then** the system scans all pages and displays a complete inventory of all text styles from local and team library sources with usage counts
2. **Given** the audit has completed, **When** Alex views the results, **Then** styles are grouped by library source and hierarchy with expandable tree navigation showing which pages contain each style
3. **Given** a document with text layers that have no style assigned, **When** the audit completes, **Then** unstyled text layers are grouped in a "Needs Styling" section showing count and locations
4. **Given** a document with 1000+ text layers, **When** the audit runs, **Then** it completes in under 30 seconds with progress indication
5. **Given** an audit in progress, **When** Alex cancels the operation, **Then** the audit stops immediately and returns to the initial state

---

### User Story 2 - Migrate Styles Between Libraries (Priority: P2)

Alex needs to migrate all instances of deprecated "Typography/Old/Heading-1" from the legacy library to "Typography/Display/Large" in the new design system library across the entire document.

**Why this priority**: Library migration is the primary high-value workflow that justifies the audit capability. This is where design system teams save days of manual work and eliminate migration errors.

**Independent Test**: Can be fully tested by selecting a source style, choosing a replacement from a different library, confirming the operation, and verifying all affected layers now use the new style with version history preserved.

**Acceptance Scenarios**:

1. **Given** an audit showing "Typography/Old/Heading-1" used in 127 text layers across 15 pages, **When** Alex selects this style and chooses "Replace with style from library", **Then** a cross-library style picker displays all available styles from local and team libraries
2. **Given** Alex has selected "Typography/Display/Large" as the replacement, **When** reviewing the confirmation dialog, **Then** the dialog shows "Replace [Typography/Old/Heading-1] from [Legacy DS] with [Typography/Display/Large] from [New DS v2] in 127 text layers?" with affected layer count
3. **Given** Alex confirms the replacement, **When** the operation begins, **Then** the system creates a version history checkpoint before making any changes
4. **Given** the replacement operation completes successfully, **When** Alex reviews the document, **Then** all 127 text layers now use the new style and maintain their original component override status
5. **Given** a replacement operation affecting layers in components, **When** the operation completes, **Then** component instances preserve their override states correctly

---

### User Story 3 - Analyze Token Usage (Priority: P3)

Alex wants to understand which design tokens are being used in text layers to assess token adoption rates and identify text layers using direct color values instead of semantic tokens.

**Why this priority**: Token visibility supports the broader design system maturity journey but is less urgent than core style management. Basic detection delivers value while deferring complex chain visualization.

**Independent Test**: Can be fully tested by running an audit on a document with token-based text layers and verifying that tokens are detected, counted, and displayed with their values.

**Acceptance Scenarios**:

1. **Given** a document with text layers using design tokens for color, **When** the audit completes, **Then** a separate "Tokens" view displays all detected tokens with their names, values, and usage counts
2. **Given** the token view is displayed, **When** Alex selects a specific token, **Then** the detail panel shows all text layers using that token grouped by page and component
3. **Given** text layers using both styles and tokens (e.g., style for typography, token for color), **When** viewing the audit results, **Then** the system correctly identifies this mixed usage pattern
4. **Given** token usage data has been collected, **When** Alex views the analytics dashboard, **Then** token coverage percentage is displayed alongside style adoption metrics

---

### User Story 4 - Replace Tokens Across Document (Priority: P4)

Alex needs to migrate text layers from the deprecated "color.text.primary-old" token to the new "semantic.text.default" token across the entire document.

**Why this priority**: Token replacement follows the same safety model as style replacement but is lower priority since token adoption is typically less mature than style adoption in most design systems.

**Independent Test**: Can be fully tested by selecting a source token, choosing a replacement token, confirming the operation, and verifying all affected layers now use the new token.

**Acceptance Scenarios**:

1. **Given** an audit showing "color.text.primary-old" used in 45 text layers, **When** Alex selects this token and chooses "Replace token", **Then** a token picker displays all available tokens from the document's collections
2. **Given** Alex has selected "semantic.text.default" as the replacement, **When** reviewing the confirmation dialog, **Then** the dialog shows the affected layer count and token details
3. **Given** Alex confirms the token replacement, **When** the operation completes, **Then** all 45 text layers now reference the new token and the change is reflected in the next audit

---

### User Story 5 - Export Audit Report for Stakeholders (Priority: P5)

Alex needs to generate a comprehensive PDF report of style adoption metrics to present to design leadership, demonstrating design system ROI and identifying areas needing improvement.

**Why this priority**: Reporting supports communication and metrics tracking but is not required for the core workflow of auditing and migrating styles.

**Independent Test**: Can be fully tested by running an audit, selecting "Export PDF", and verifying the generated report contains all expected sections with accurate data.

**Acceptance Scenarios**:

1. **Given** a completed audit with style and token data, **When** Alex clicks "Export PDF Report", **Then** a PDF is generated containing executive summary metrics, style inventory by library, and adoption visualizations
2. **Given** the export operation is triggered, **When** the PDF is generated, **Then** it includes document metadata (file name, timestamp, page count) and key metrics (style adoption rate, library distribution, deprecated style count)
3. **Given** Alex needs detailed data analysis, **When** selecting "Export CSV", **Then** a CSV file is generated with one row per text layer including all metadata (style, token, page, component context)
4. **Given** the CSV export completes, **When** Alex opens it in spreadsheet software, **Then** all columns are properly formatted and data is structured for filtering and pivot table analysis

---

### Edge Cases

- **What happens when a document has no text layers?** Display a clear message stating "No text layers found in document" and disable audit-dependent features
- **What happens when a style exists in the inventory but has zero usage?** Display the style in the tree with a usage count of 0, allowing visibility into available but unused styles
- **What happens when a text layer uses a deleted/missing style?** Mark the layer as "unstyled" with a warning indicator that the assigned style no longer exists
- **What happens when network issues prevent library access during audit?** Continue audit with local styles only and display a warning about library styles not being accessible
- **What happens when a replacement operation is interrupted?** Version history checkpoint allows rollback; display error message with option to retry or cancel
- **What happens when replacing a style that's already in use in the document?** Allow the operation but show a confirmation that some layers may already use the target style
- **What happens when a user tries to replace a style with itself?** Show an error message: "Source and target styles cannot be the same"
- **What happens with nested component instances during style replacement?** Preserve the nested component structure and override states, applying style changes only to the text layers
- **What happens when a document uses styles from multiple team libraries?** Display all libraries as separate groups in the tree view with library name labels
- **What happens when audit is run on a document with mixed tokens (variables mode A vs mode B)?** Detect and display token usage with mode information included in token metadata
- **What happens when the document changes during an audit?** Invalidate the audit results, display a warning message "Document has been modified. Audit results may be outdated. Please re-run the audit.", and provide a clear "Re-run Audit" button
- **What happens when the document changes during a replacement operation?** Complete the current replacement operation (protected by version history checkpoint), then invalidate any cached audit data and prompt user to re-audit to see updated state
- **What happens in collaborative documents where another user makes changes?** Treat document modifications from any source (local user or collaborator) as triggering invalidation; system does not distinguish between change sources
- **What happens when a document has exactly 10,000 or 50,000 text layers?** 10,000 layers triggers the warning (Enterprise Zone starts at 10,001), 50,000 layers is allowed (hard limit is >50,000, meaning 50,001+ blocked)
- **What happens if layer count changes between warning acceptance and audit start?** Re-validate layer count; if it now exceeds hard limit, block with error; if it dropped below warning threshold, proceed without warning

## Operation State Model

### Audit Operation States

Audit operations follow a defined state machine to enable accurate progress indication and proper error handling:

1. **idle**: No audit is running; UI shows "Run Audit" button; cached results may be displayed if available
2. **validating**: Initial checks before starting audit (document accessibility, page count); typically <1 second
3. **scanning**: Traversing document pages and discovering text nodes; progress percentage based on pages scanned
4. **processing**: Extracting style metadata, detecting tokens, building inventory; progress based on layers processed
5. **complete**: Audit finished successfully; results displayed; transitions back to idle state with cached results
6. **error**: Audit failed due to error (network, permissions, API limits); error message displayed; user can retry
7. **cancelled**: User cancelled operation; partial results discarded; returns to idle state

**State Transitions**:

- idle → validating (user clicks "Run Audit")
- validating → scanning (validation passed)
- validating → error (validation failed)
- scanning → processing (all pages discovered)
- scanning → cancelled (user clicks "Cancel")
- processing → complete (all layers processed successfully)
- processing → error (unrecoverable error during processing)
- processing → cancelled (user clicks "Cancel")
- complete → idle (results displayed)
- error → idle (user dismisses error or clicks "Retry")
- cancelled → idle (operation cleanup complete)

### Replacement Operation States

Style and token replacement operations follow a similar pattern with an additional checkpoint phase:

1. **idle**: No replacement running; UI shows replacement options
2. **validating**: Checking that source and target are valid, counting affected layers; typically <1 second
3. **creating_checkpoint**: Creating Figma version history save point; duration varies by document size
4. **processing**: Applying style/token changes to each affected layer; progress based on layers updated
5. **complete**: Replacement finished successfully; audit results invalidated; user prompted to re-audit
6. **error**: Replacement failed; version history allows rollback; error message displayed with rollback option
7. **cancelled**: Not applicable - replacement operations cannot be cancelled once checkpoint is created (too risky)

**State Transitions**:

- idle → validating (user confirms replacement)
- validating → creating_checkpoint (validation passed)
- validating → error (validation failed - invalid source/target)
- creating_checkpoint → processing (checkpoint created successfully)
- creating_checkpoint → error (checkpoint creation failed)
- processing → complete (all layers updated successfully)
- processing → error (error during layer updates - version history allows rollback)
- complete → idle (audit invalidated, user proceeds)
- error → idle (user dismisses error or initiates rollback)

### UI Progress Indication Mapping

Each state maps to specific UI feedback:

- **validating**: Indeterminate spinner with "Validating..." message
- **scanning**: Progress bar with percentage and "Scanning page X of Y" message
- **processing**: Progress bar with percentage and "Processing layer X of Y" message
- **creating_checkpoint**: Indeterminate spinner with "Creating version checkpoint..." message
- **complete**: Success checkmark with summary statistics
- **error**: Error icon with specific error message and action buttons (Retry/Rollback/Dismiss)
- **cancelled**: Brief "Cancelled" message before returning to idle

## Performance & Scalability Boundaries

### Document Size Limits

The system enforces tiered boundaries to balance usability with performance and reliability:

**Optimal Performance Zone (0-1,000 text layers)**:

- Target: Complete audit in under 30 seconds
- Behavior: Normal operation with no warnings
- Expected user experience: Instant results, smooth interactions

**Acceptable Performance Zone (1,001-5,000 text layers)**:

- Target: Complete audit in under 60 seconds
- Behavior: Normal operation, audit may take longer but remains acceptable
- Expected user experience: Noticeable wait time but predictable completion

**Standard Performance Zone (5,001-10,000 text layers)**:

- Target: Complete audit in 2-3 minutes
- Behavior: Normal operation with no warnings
- Expected user experience: Extended wait time but acceptable for enterprise workflows

**Enterprise Zone (10,001-50,000 text layers)**:

- Trigger: When document analysis detects >10,000 text layers
- Warning message displayed BEFORE audit starts: "This document contains [X] text layers. Audit may take several minutes and UI may become less responsive. Continue?"
- User options: "Continue Audit" or "Cancel"
- Expected audit time: 3-15 minutes depending on layer count (5-8 min for 25k, 10-15 min for 50k)
- Expected user experience: Significant wait time, possible UI lag, progress indication critical

**Hard Limit (>50,000 text layers)**:

- Trigger: When document analysis detects >50,000 text layers
- Error message: "This document contains [X] text layers, which exceeds the maximum supported limit of 50,000. Please reduce document scope or audit smaller sections separately."
- Behavior: Audit operation blocked, cannot proceed
- Rationale: Beyond this threshold, browser memory constraints and Figma API performance make reliable operation impossible

### Performance Degradation Characteristics

As layer count increases, performance degrades in these areas:

1. **Audit Duration**: Roughly linear with layer count (1000 layers ≈ 30s, 5000 layers ≈ 60s, 25000 layers ≈ 5-8 min, 50000 layers ≈ 10-15 min)
2. **Memory Usage**: Increases with layer count and cached audit data size
3. **UI Responsiveness**: Tree view rendering slows with >500 unique styles; search/filter remain performant
4. **Replacement Operations**: Duration scales linearly with affected layer count; UI updates may lag for >1000 layers

### Mitigation Strategies

For documents in the Enterprise Zone (10k-50k layers):

- **Progressive rendering**: Display audit results in batches to avoid blocking UI
- **Virtualized lists**: Only render visible tree nodes to reduce DOM size
- **Chunked processing**: Process layers in batches with yield points to prevent browser timeouts
- **Reduced real-time updates**: Update progress indicator less frequently (every 50 layers vs every 10)

### Batching Strategy for Replacement Operations

Bulk style and token replacements use adaptive batching to balance performance with reliability:

**Adaptive Batch Size Algorithm**:

1. **Initial batch size**: 100 layers per batch
2. **On successful batch**: Continue with current batch size
3. **On batch error** (timeout, API error, network issue):
   - Reduce batch size to 25 layers
   - Retry the failed batch with smaller size
   - Continue with 25 layers/batch for remainder of operation
4. **On consistent success** (5 consecutive batches at reduced size):
   - Gradually increase batch size back to 100 layers
   - Increment by 25 layers per batch (25 → 50 → 75 → 100)

**Performance Characteristics**:

- **Optimal conditions**: ~100 layers/second (1000 layer replacement ≈ 10 seconds)
- **After error recovery**: ~25 layers/second (1000 layer replacement ≈ 40 seconds)
- **Progress accuracy**: Updated after each batch completes (every 1-4 seconds depending on batch size)

**Error Handling**:

- **Transient errors** (network timeout, temporary API unavailability): Retry current batch up to 3 times with exponential backoff (1s, 2s, 4s)
- **Persistent errors** (permissions, invalid style ID): Fail entire operation, trigger rollback via version history
- **Partial batch failures**: If individual layers within a batch fail, mark those layers as failed, continue with remaining batches, report failed layer IDs at completion

## Error Recovery & Reliability

### Error Taxonomy

Errors are classified into categories with specific recovery strategies:

**Transient Errors** (automatically retried):

- Network timeouts or connection interruptions
- Temporary Figma API unavailability (5xx server errors)
- Rate limiting responses (429 Too Many Requests)
- Browser resource exhaustion (recoverable via retry with smaller batch)

**Persistent Errors** (trigger rollback):

- Permission denied (user lacks edit access to document)
- Invalid style ID (source or target style deleted/inaccessible)
- Invalid token ID (token deleted or collection disabled)
- API authentication failures
- Document locked by another operation

**Validation Errors** (prevent operation start):

- Source and target style are identical
- No text layers selected for replacement
- Document exceeds hard limit (>25,000 layers)
- Required libraries not enabled

**Partial Failures** (mark and continue):

- Individual layer locked or hidden
- Component instance with broken main component reference
- Mixed text styles in single layer (applies to first character's style)

### Recovery Workflows

**Audit Operation Failure Recovery**:

1. **During scanning state**:
   - Transient error: Retry current page scan up to 3 times
   - Persistent error: Transition to error state, display error message with "Retry Audit" button
   - User action: Click "Retry Audit" restarts from beginning (no partial audit results)

2. **During processing state**:
   - Transient error: Retry current layer processing up to 3 times
   - Persistent error: Transition to error state, discard partial results
   - Partial results NOT cached (all-or-nothing approach for data consistency)

**Replacement Operation Failure Recovery**:

1. **Before checkpoint created** (validating state):
   - Any error: Transition to error state, no rollback needed (document not modified)
   - User action: Fix issue and retry replacement

2. **After checkpoint created** (processing state):
   - **Transient error in batch**:
     - Retry batch 1st time: Wait 1 second, retry with current batch size
     - Retry batch 2nd time: Wait 2 seconds, retry with reduced batch size (25 layers)
     - Retry batch 3rd time: Wait 4 seconds, retry with minimum batch size (10 layers)
     - After 3 failed retries: Classify as persistent error, proceed to rollback

   - **Persistent error detected**:
     - Immediately stop processing remaining batches
     - Transition to error state
     - Display error message: "Replacement failed: [error details]. Document has been restored to checkpoint. Click 'View Version History' to confirm."
     - Version history checkpoint allows manual or automatic rollback

   - **Partial layer failures within successful batch**:
     - Continue processing (do not count as batch failure)
     - Collect failed layer IDs and reasons
     - At completion, display summary: "Replacement completed with warnings: [X] of [Y] layers updated. [Z] layers failed: [reasons]. View failed layers?"

3. **Version History Rollback**:
   - Automatic trigger on persistent errors during replacement
   - User presented with: "Operation failed. Rollback to version checkpoint '[timestamp]'?" with "Rollback" and "Keep Changes" buttons
   - "Rollback" action restores document to pre-replacement state
   - "Keep Changes" leaves document in partial state (user responsible for cleanup)

### Reliability Targets

- **Audit success rate**: >99% for documents within optimal/acceptable zones (0-5k layers)
- **Replacement success rate**: >95% for operations <1000 layers with automatic retry handling
- **Transient error recovery**: >90% of network/timeout errors resolved by automatic retry
- **Data integrity**: 100% (no corrupted document states due to partial operations)
- **Rollback success**: 100% (version history checkpoint always available when needed)

## Requirements _(mandatory)_

### Functional Requirements

#### Core Audit Engine

- **FR-001**: System MUST scan all pages in the current Figma document when audit is initiated
- **FR-002**: System MUST detect all text layers regardless of their location (top-level frames, groups, component instances, nested components)
- **FR-003**: System MUST identify text styles from local document sources and team library sources
- **FR-004**: System MUST categorize each text layer as "fully-styled" (all properties match assigned style), "partially-styled" (one or more properties differ from assigned style due to local overrides), or "unstyled" (no style assigned)
- **FR-005**: System MUST complete document audit in under 30 seconds for documents with up to 1000 text layers
- **FR-006**: System MUST provide real-time progress indication during audit following the defined state model (validating → scanning → processing) with appropriate UI feedback for each state
- **FR-007**: System MUST allow users to cancel an in-progress audit operation during scanning or processing states, transitioning to cancelled state
- **FR-007a**: System MUST detect when the Figma document has been modified after an audit completes. Modifications that trigger invalidation include: text layer add/delete, text layer property changes, text style add/delete/modify, design token add/delete/modify. Changes to non-text elements (shapes, images, frames without text children) do NOT trigger invalidation. Both user edits and collaborative changes from other users trigger invalidation equally.
- **FR-007b**: System MUST invalidate cached audit results when document modifications are detected
- **FR-007c**: System MUST display a warning message when showing potentially stale audit data: "Document has been modified. Audit results may be outdated. Please re-run the audit."
- **FR-007d**: System MUST provide a prominent "Re-run Audit" button when audit results are invalidated
- **FR-007e**: System MUST detect total text layer count before starting audit (during validating state)
- **FR-007f**: System MUST display warning dialog when layer count exceeds 10,000: "This document contains [X] text layers. Audit may take several minutes and UI may become less responsive. Continue?" with Continue/Cancel options
- **FR-007g**: System MUST block audit operation when layer count exceeds 50,000 with error message: "This document contains [X] text layers, which exceeds the maximum supported limit of 50,000. Please reduce document scope or audit smaller sections separately."
- **FR-007h**: System MUST complete audit in under 30 seconds for documents with 0-1,000 text layers under typical hardware conditions

#### Style Metadata & Metrics

- **FR-008**: System MUST capture for each text style: style name, full hierarchical path (e.g., "Typography/Heading/H1"), source library name or "Local", and usage count
- **FR-009**: System MUST track page distribution showing which pages contain text layers using each style
- **FR-010**: System MUST identify component context for each text layer: usage in main components, usage in component instances, usage in plain frames/groups
- **FR-011**: System MUST count component override instances where style properties have been modified
- **FR-012**: System MUST detect parent-child relationships in style hierarchy based on naming conventions (e.g., "Typography/Heading" is parent of "Typography/Heading/H1")

#### Unstyled Text Detection

- **FR-013**: System MUST identify all text layers without any style assignment
- **FR-014**: System MUST group unstyled text in a dedicated "Needs Styling" section at the same level as library style groups
- **FR-015**: System MUST provide count and page location information for each unstyled text layer

#### Design Token Analysis

- **FR-016**: System MUST detect design tokens (variables) used in text layer properties (color tokens)
- **FR-017**: System MUST capture for each token: token name, current value, token type (color/typography/spacing/semantic)
- **FR-018**: System MUST track usage count per token showing how many text layers reference it
- **FR-019**: System MUST identify mixed usage patterns where text layers use both styles and tokens (e.g., style for typography, token for color)
- **FR-020**: System MUST capture token collection and mode information for each detected token

#### User Interface - Style Dashboard

- **FR-021**: System MUST display audit results in a tree view grouped by library → hierarchy
- **FR-022**: System MUST provide toggle to switch grouping between "by library" and "by hierarchy" views
- **FR-023**: System MUST show usage count badge next to each style in the tree
- **FR-024**: System MUST provide expandable list of pages for each style showing where it's used
- **FR-025**: System MUST include search capability to filter styles by name
- **FR-026**: System MUST include filter options for: by library, by usage count (unused=0 usages, rarely used=1-5 usages, frequently used=>20 usages, or custom numeric range), by assignment status (fully-styled, partially-styled)
- **FR-027**: System MUST visually distinguish between local styles and library styles with clear labeling

#### User Interface - Token View

- **FR-028**: System MUST provide a separate view/tab for design tokens distinct from the style view
- **FR-029**: System MUST group tokens by collection in the token view
- **FR-030**: System MUST display token coverage metrics (see Metrics Definitions) alongside token adoption metrics to provide complete visibility into token system health

#### User Interface - Detail Panel

- **FR-031**: System MUST display a detail panel when a style is selected showing complete list of layers using that style
- **FR-031a**: System MUST support detail panel display for styles with up to 5,000 layer usages using virtualized rendering to maintain render time <500ms and smooth scrolling at 60fps per Constitution Principle II (Performance & Responsiveness)
- **FR-032**: System MUST group detail panel layers by page and component
- **FR-033**: System MUST provide click-to-navigate functionality to focus each layer in the Figma canvas
- **FR-034**: System MUST provide quick actions in the detail panel: Replace and Navigate options

#### Style Management - Replacement

- **FR-035**: System MUST allow users to select one or more source styles for replacement
- **FR-036**: System MUST provide a cross-library style picker showing all available styles from local and team library sources
- **FR-037**: System MUST display a preview showing affected layer count before executing replacement
- **FR-038**: System MUST create an automatic version history checkpoint before executing any style replacement operation (creating_checkpoint state)
- **FR-039**: System MUST show confirmation dialog with format: "Replace [Style A] from [Library X] with [Style B] from [Library Y] in [N] text layers?"
- **FR-040**: System MUST preserve component override states during style replacement operations
- **FR-041**: System MUST provide real-time progress indication during replacement following the state model (validating → creating_checkpoint → processing → complete/error) with appropriate UI feedback for each state
- **FR-041a**: System MUST NOT allow cancellation of replacement operations once the creating_checkpoint state is reached (operations must complete or fail to maintain data integrity)
- **FR-041b**: System MUST process replacement operations in batches using adaptive batch sizing: start at 100 layers/batch, reduce to 25 on errors, increase back to 100 after 5 consecutive successes
- **FR-041c**: System MUST retry failed batches up to 3 times with exponential backoff (1s, 2s, 4s) for transient errors (network timeout, API unavailability, rate limiting)
- **FR-041d**: System MUST classify errors as transient, persistent, validation, or partial failure according to the defined error taxonomy
- **FR-041e**: System MUST stop processing and initiate rollback workflow for persistent errors (permissions, invalid IDs, authentication) after retry attempts exhausted
- **FR-041f**: System MUST continue processing remaining batches when individual layers fail within a batch, collecting failed layer IDs and reasons for final error report
- **FR-041g**: System MUST present rollback confirmation dialog after persistent failures: "Operation failed. Rollback to version checkpoint '[timestamp]'?" with Rollback/Keep Changes options
- **FR-041h**: System MUST display completion summary for partial successes: "Replacement completed with warnings: [X] of [Y] layers updated. [Z] layers failed: [reasons]. View failed layers?"

#### Token Management - Replacement

- **FR-042**: System MUST allow users to select source tokens for replacement
- **FR-043**: System MUST provide a token picker showing all available tokens from document collections
- **FR-044**: System MUST apply the same safety model to token replacement as style replacement (version checkpoint, confirmation, progress indication)
- **FR-045**: System MUST preserve component override states during token replacement operations

#### Analytics Dashboard

- **FR-046**: System MUST display style adoption rate calculated as percentage of styled versus unstyled text layers
- **FR-047**: System MUST display library usage distribution showing percentage of layers using each library source
- **FR-048**: System MUST display token coverage percentage showing what proportion of available design tokens are actively used in the document (see Metrics Definitions for distinction from token adoption)
- **FR-049**: System MUST display top 10 most-used styles ranked by usage count
- **FR-050**: System MUST display count of text layers in "Needs Styling" category

#### Export & Reporting

- **FR-051**: System MUST generate PDF report containing: executive summary metrics, style inventory by library, adoption visualizations, timestamp, and document metadata
- **FR-052**: System MUST generate CSV export containing one row per text layer with columns: layer ID, style name, style source, token name (if applicable), page name, component context, assignment status
- **FR-053**: System MUST include document metadata in exports: file name, total page count, total text layer count, audit timestamp

### Analytics Metrics Definitions

The Analytics Dashboard displays several key metrics to provide visibility into design system health. Clear definitions are essential to avoid confusion between related but distinct metrics.

#### Style Adoption Rate

**Definition**: Percentage of text layers that are "fully-styled" (have an assigned style that matches all their text properties).

**Formula**: (Fully-styled layers / Total text layers) × 100%

**Example**: In a document with 100 text layers, if 75 layers are fully-styled, then Style Adoption Rate = 75%

**What it indicates**:

- Measures how well the document conforms to the style system
- High adoption (80%+) indicates strong design system implementation
- Low adoption (<50%) suggests many layers need styling or override content

**Related metric**: Assignment Status Breakdown (fully-styled vs partially-styled vs unstyled) shows the detailed breakdown

#### Token Coverage Rate

**Definition**: Percentage of design tokens that are actively used in at least one text layer in the document.

**Formula**: (Number of unique tokens used / Total number of tokens) × 100%

**Example**: If your design system has 50 design tokens but only 30 are referenced by text layers, then Token Coverage = 60%

**What it indicates**:

- Identifies which tokens in your design system are actively utilized
- High coverage (80%+) indicates tokens are well-integrated
- Low coverage (<50%) suggests token system may have unused/orphaned tokens, or incomplete token adoption
- Helps identify tokens available for consolidation or removal

**Related metric**: Token Adoption Rate (shown separately) measures what % of layers USE tokens, which is different from coverage of which tokens are used

**Key difference from Token Adoption**:

- **Token Coverage** = "Of the tokens we have, how many are used?" (token-centric view)
- **Token Adoption** = "Of the layers we have, how many use tokens?" (layer-centric view)

#### Library Usage Distribution

**Definition**: Percentage breakdown of text layers by library source (local styles vs each team library).

**Formula**: (Layers using Library X / Total text layers) × 100% for each library

**Example**: 60% from Local, 25% from Design System Library, 15% from Brand Library

**What it indicates**:

- Shows which sources contribute most to the document's styling
- Imbalance may suggest incomplete library migration or adoption

#### Top 10 Most-Used Styles

**Definition**: Ranked list of the 10 styles with the highest usage count (most layers using them).

**Ranked by**: Usage count (layers using that style)

**What it indicates**:

- Identifies core styles that are critical to the design system
- High-usage styles should be carefully maintained (changes affect many layers)
- Inverted pyramid pattern (few styles with high usage) indicates healthy consolidation

#### Unstyled Layer Count

**Definition**: Total number of text layers without any assigned style.

**What it indicates**:

- Identifies styling gaps
- Every unstyled layer is a candidate for style assignment
- Forms the "Needs Styling" section in the tree view

### Key Entities

- **Text Layer**: Represents a text node in the Figma document; attributes include layer ID, text content preview, page location, component context (main/instance/none), style assignment status (fully-styled/partially-styled/unstyled), assigned style reference, token references, visibility, opacity
- **Text Style**: Represents a text style from local or library source; attributes include style ID, style name, hierarchical path, source library name (or "Local"), usage count, page distribution, component usage breakdown, parent-child hierarchy relationships
- **Design Token**: Represents a variable used in text properties; attributes include token ID, token name, token type, current value, collection name, mode information, usage count, associated text layers
- **Audit Result**: Represents the complete output of a document scan; attributes include timestamp, document file name, total text layer count, total style count, total token count, style inventory, token inventory, analytics metrics, categorized text layers
- **Library Source**: Represents the origin of a style; attributes include source type (local/team library), library name, library status (enabled/disabled), total styles from this source, usage statistics

## Success Criteria _(mandatory)_

### Measurable Outcomes

- **SC-001**: Users can complete a full document style audit in under 30 seconds for documents with 100+ pages and 1000+ text layers
- **SC-002**: Library migration operations reduce time-to-complete from days (manual review and replacement) to minutes (bulk replacement), representing a 90% reduction in manual effort
- **SC-003**: Users achieve 100% style coverage visibility, meaning every text layer in the document is identified and categorized by style assignment status with zero false negatives
- **SC-004**: Zero deprecated styles are missed during audits, meaning all style instances are accurately detected regardless of component nesting or page location
- **SC-005**: Style replacement operations complete with 95% success rate without requiring rollback or manual cleanup
- **SC-006**: Design system teams can generate executive reports in under 60 seconds showing style adoption metrics suitable for stakeholder presentations
- **SC-007**: Users can identify all unstyled text layers in a document within 30 seconds of audit completion
- **SC-008**: Token usage visibility is available for 100% of design tokens used in text properties with accurate usage counts
- **SC-009**: 90% of users successfully complete their first style replacement operation on first attempt without errors or confusion
- **SC-010**: All bulk operations (audit, replacement) handle 500+ affected layers without timeout or performance degradation

## Assumptions _(mandatory)_

### Scope Assumptions for MVP v1.0

- **MVP focuses on text styles only**: Color styles and effect styles are explicitly deferred to v1.1 and v1.2 respectively, allowing faster delivery of core value proposition
- **Published component library styles deferred**: MVP supports local and team library styles only; published component library style detection will be added in v1.1 based on user demand
- **Token chain visualization deferred**: Basic token detection (name, value, usage count) is included in MVP; token chain/alias visualization (e.g., text.primary → brand.blue → #0066CC) is deferred to v1.1
- **Suggested style matching deferred**: Unstyled text detection and grouping is included; automatic style matching suggestions based on property similarity are deferred to v1.1
- **Single-document scope**: MVP operates on the current document only; multi-file batch processing is deferred to v2.0+
- **Manual export process**: PDF and CSV exports are user-initiated; scheduled audits with automated reporting are deferred to v2.0+

### Technical Assumptions

- **Figma API stability**: Assumes Figma Plugin API provides stable access to text nodes, style metadata, library information, token/variable data, and version history
- **Team library access**: Assumes enabled team libraries are accessible during audit; unavailable libraries will show warning but won't block audit
- **Performance targets based on typical hardware**: 30-second audit target assumes standard development machine; lower-end hardware may experience slightly longer times
- **Version history API available**: Assumes Figma provides programmatic access to create version history checkpoints before bulk operations
- **Style naming conventions**: Parent-child style hierarchy detection assumes forward-slash separated naming (e.g., "Category/Subcategory/Style")
- **Component override preservation**: Assumes Figma API allows modification of text layer styles without breaking component instance relationships

### User Behavior Assumptions

- **Users create version history before bulk changes**: While system creates automatic checkpoints, users are expected to maintain regular version history as standard practice
- **Design system maturity**: Primary users are teams with established design systems and multiple style libraries; early-stage design system adoption may have different usage patterns
- **Style migration workflows**: Assumes users need to replace entire styles in bulk rather than individual layer-by-layer updates
- **Token adoption varies**: Some organizations have high token adoption while others use direct values; feature must work regardless of token maturity level

## Out of Scope _(mandatory)_

### Explicitly Excluded from MVP v1.0

- **Font-specific auditing features**: All capabilities from the previous font audit tool (font family/weight/size analysis independent of styles) are removed in the pivot to style governance
- **Color and effect styles**: Detection and management of color styles and effect styles are deferred to v1.1 and v1.2
- **Published component library support**: Only local and team libraries are supported in MVP
- **Token chain visualization**: Advanced token alias chains and semantic token relationships deferred to v1.1
- **Suggested style matching**: Automatic suggestions for applying styles to unstyled text based on property matching
- **Bulk styling of unstyled text**: Ability to select multiple unstyled layers and apply a style in one operation (deferred to v1.1)
- **Multi-file batch processing**: Running audits across multiple Figma files in a single operation
- **Scheduled/automated audits**: Background audits that run on a schedule or detect changes over time
- **Style deprecation workflow automation**: Automated tagging and tracking of deprecated styles with migration workflows
- **Team-wide adoption dashboards**: Aggregated metrics across multiple files or team members
- **Figma REST API integration**: Plugin operates entirely within Figma Plugin API; external CI/CD integration deferred to v2.0+
- **Style guide auto-generation**: Automatic creation of style guide documentation from audit data
- **Migration rule templates**: Saved/reusable replacement rules for common migration patterns
- **Historical adoption tracking**: Time-series data showing style adoption changes over weeks/months
- **AI-powered style consolidation**: Machine learning suggestions for merging similar styles
- **Permission/role-based access**: All users with plugin access have full functionality; role restrictions deferred to enterprise version

### Known Limitations

- **Mixed text styles in single layer**: When a text layer has multiple styles applied to character ranges, system reports style at first character position as the primary style
- **Network-dependent library access**: Library style detection requires active network connection to access team libraries
- **Large document performance**: Documents with 5000+ text layers may exceed 30-second audit target on lower-end hardware
- **Style property comparison granularity**: Partial style matching (e.g., only font-family matches but size differs) is detected but detailed property-level diff is not shown in MVP
- **Undo granularity**: Version history checkpoint allows document-level rollback; individual layer-level undo of bulk operations not supported

## Dependencies _(mandatory)_

### External Dependencies

- **Figma Plugin API**: Complete dependency on Figma's plugin API for document traversal, node access, style enumeration, library access, token/variable API, and version history
- **Team Library Availability**: Team library style detection requires libraries to be enabled in the document and accessible over network
- **Browser PDF Generation**: PDF export relies on browser's PDF rendering capabilities or requires third-party PDF generation library

### Internal Dependencies

- **Existing Code to be Replaced**: Current font audit implementation (scanning, UI, state management) will be refactored/replaced for style governance workflows
- **Message Passing Infrastructure**: Existing dual-context architecture (main/UI) will be preserved and extended for style management messages
- **Build System**: Current Vite-based build configuration will be maintained

### Design System Dependencies

- **Assumes Standard Naming Conventions**: Hierarchical style organization detection works best with forward-slash separated naming (e.g., "Typography/Heading/H1")
- **Library Organization Patterns**: Feature assumes styles are organized by library source with logical grouping

## Risks & Mitigations _(optional)_

### Technical Risks

- **Risk**: Large file performance - documents with 5000+ text layers may exceed performance targets
  - **Mitigation**: Implement pagination and progressive loading; process pages in batches with yield points to prevent UI blocking

- **Risk**: Library conflicts - same style name exists in multiple libraries causing ambiguity
  - **Mitigation**: Clear library source labeling in UI; confirmation dialogs always show full library path for source and target

- **Risk**: Destructive changes - bulk replacement errors could affect hundreds of layers
  - **Mitigation**: Mandatory version history checkpoint before any bulk operation; clear confirmation dialogs with affected layer count preview

- **Risk**: Figma API limitations or breaking changes
  - **Mitigation**: Graceful degradation with clear user messaging when API capabilities are unavailable; avoid undocumented API features

### User Experience Risks

- **Risk**: Complex UI - tree view with hundreds of styles could be overwhelming
  - **Mitigation**: Progressive disclosure with collapsed tree by default; robust search and filtering to narrow results

- **Risk**: Unclear replacement impact - users may not understand component override implications
  - **Mitigation**: Clear documentation and confirmation dialogs explaining override preservation; preview affected layer count before operation

### Workflow Risks

- **Risk**: Interrupted operations - network issues or browser crashes during bulk replacement
  - **Mitigation**: Version history checkpoint allows rollback; operation progress tracking with resume capability consideration for future versions

## Change Log

**v1.0** - Initial specification for design system style governance feature. Complete strategic pivot from font auditing to style and token management focused on design system team workflows. MVP scope: local + team library styles, basic token detection, style/token replacement, analytics, and reporting.
