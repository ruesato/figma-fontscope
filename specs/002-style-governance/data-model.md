# Data Model: Design System Style Governance

**Feature**: `002-style-governance`
**Created**: 2025-11-20
**Purpose**: Define core entities, attributes, relationships, and state machines for the style governance feature

## Overview

This document defines the data structures and relationships used throughout the plugin. All entities are designed to work with the Figma Plugin API while maintaining clean separation between audit data, UI state, and export formats.

---

## Core Entities

### 1. TextLayer

Represents a text node in the Figma document with style and token metadata.

**TypeScript Signature**:

```typescript
interface TextLayer {
  // Identity
  id: string; // Figma node ID
  name: string; // Layer name from Figma

  // Content
  textContent: string; // Text content preview (first 50 chars)
  characters: number; // Total character count

  // Location
  pageId: string; // Parent page ID
  pageName: string; // Parent page name
  parentType: 'MAIN_COMPONENT' | 'INSTANCE' | 'FRAME' | 'GROUP';
  componentPath?: string; // Full component hierarchy if in component

  // Style Assignment
  assignmentStatus: 'fully-styled' | 'partially-styled' | 'unstyled';
  styleId?: string; // Assigned text style ID (if any)
  styleName?: string; // Resolved style name
  styleSource?: string; // Library name or "Local"

  // Token Usage
  tokens: TokenBinding[]; // Design tokens applied to this layer

  // Visual Properties
  visible: boolean; // Layer visibility state
  opacity: number; // Layer opacity (0-1)

  // Override Status (for component instances)
  hasOverrides: boolean; // True if style properties locally overridden
  overriddenProperties?: string[]; // List of overridden property names
}
```

**Figma API Mapping**:

- Source: `TextNode` from Figma Plugin API
- Key properties: `id`, `name`, `characters`, `textStyleId`, `parent`, `visible`, `opacity`
- Parent traversal: Walk up node tree to find page and component context

**Relationships**:

- **BelongsTo** TextStyle (via styleId)
- **BelongsTo** Page (via pageId)
- **HasMany** TokenBinding (via tokens array)

---

### 2. TextStyle

Represents a text style from local document or library source with usage metrics.

**TypeScript Signature**:

```typescript
interface TextStyle {
  // Identity
  id: string; // Figma style ID
  name: string; // Style name (e.g., "Heading/H1")
  key: string; // Unique key for stable references

  // Hierarchy
  hierarchyPath: string[]; // Path components ["Typography", "Heading", "H1"]
  parentStyleId?: string; // Parent style in hierarchy (if exists)
  childStyleIds: string[]; // Child styles in hierarchy

  // Source
  sourceType: 'local' | 'team_library' | 'published_component';
  libraryName: string; // Library name or "Local"
  libraryId?: string; // Library ID (if from library)

  // Usage Metrics
  usageCount: number; // Total layers using this style
  pageDistribution: PageUsage[]; // Usage breakdown by page
  componentUsage: ComponentUsage; // Usage in components vs plain layers

  // Status
  isDeprecated: boolean; // Marked as deprecated
  lastModified?: Date; // Last modification timestamp (if available)
}

interface PageUsage {
  pageId: string;
  pageName: string;
  layerCount: number;
}

interface ComponentUsage {
  mainComponentCount: number; // Usage in main components
  instanceCount: number; // Usage in instances
  plainLayerCount: number; // Usage in plain frames/groups
  overrideCount: number; // Instances with overridden properties
}
```

**Figma API Mapping**:

- Source: `TextStyle` from `figma.getLocalTextStyles()` or `figma.importStyleByKeyAsync()`
- Hierarchy: Derived from name parsing (split by "/")
- Library info: From `style.remote` property

**Relationships**:

- **HasMany** TextLayer (via layers using this styleId)
- **BelongsTo** LibrarySource (via libraryId)
- **HasMany** TextStyle as children (hierarchy)
- **BelongsTo** TextStyle as parent (hierarchy)

---

### 3. DesignToken

Represents a design variable (token) used in text layer properties.

**TypeScript Signature**:

```typescript
interface DesignToken {
  // Identity
  id: string; // Figma variable ID
  name: string; // Token name (e.g., "text.primary")
  key: string; // Unique stable key

  // Type & Value
  type: 'COLOR' | 'STRING' | 'FLOAT' | 'BOOLEAN';
  resolvedType: string; // Human-readable type
  currentValue: any; // Resolved value in current mode

  // Collection & Mode
  collectionId: string; // Parent collection ID
  collectionName: string; // Collection display name
  modeId: string; // Active mode ID
  modeName: string; // Mode display name
  valuesByMode: Record<string, any>; // All mode values

  // Token Chain (for aliases)
  isAlias: boolean; // True if references another token
  aliasedTokenId?: string; // Source token ID if alias
  aliasChain?: string[]; // Full chain (e.g., ["text.primary", "brand.blue", "#0066CC"])

  // Usage Metrics
  usageCount: number; // Layers using this token
  layerIds: string[]; // Layer IDs using this token
  propertyTypes: string[]; // Which properties use it (e.g., ["fills", "fontFamily"])
}
```

**Figma API Mapping**:

- Source: `Variable` from `figma.variables.getLocalVariablesAsync()`
- Binding: Detected via `TextNode.boundVariables`
- Collection: From `figma.variables.getVariableCollectionByIdAsync()`
- Mode: From collection's current mode

**Relationships**:

- **BelongsTo** VariableCollection (via collectionId)
- **HasMany** TextLayer (via layerIds)
- **RefersTo** DesignToken (via aliasedTokenId for aliases)

---

### 4. AuditResult

Represents the complete output of a document scan with all metrics and inventories.

**TypeScript Signature**:

```typescript
interface AuditResult {
  // Metadata
  timestamp: Date; // When audit was performed
  documentName: string; // Figma file name
  documentId: string; // Figma file ID

  // Scope
  totalPages: number; // Pages scanned
  totalTextLayers: number; // Text layers found

  // Inventories
  styles: TextStyle[]; // All detected styles
  tokens: DesignToken[]; // All detected tokens
  layers: TextLayer[]; // All text layers
  libraries: LibrarySource[]; // All library sources

  // Hierarchy
  styleHierarchy: StyleHierarchyNode[]; // Computed hierarchy tree

  // Categorization
  styledLayers: TextLayer[]; // Layers with style assigned
  unstyledLayers: TextLayer[]; // Layers without style

  // Analytics
  metrics: AuditMetrics; // Computed metrics

  // Status
  isStale: boolean; // True if document modified since audit
  auditDuration: number; // Time taken in milliseconds
}

interface StyleHierarchyNode {
  styleName: string; // Full style name
  styleId?: string; // Style ID (if real style, not intermediate node)
  parentName?: string; // Parent node name
  children: StyleHierarchyNode[]; // Child nodes
  usageCount: number; // Total usage (including children)
}

interface AuditMetrics {
  // Style Adoption
  styleAdoptionRate: number; // % of layers with style (0-100)
  fullyStyledCount: number; // Layers matching style exactly
  partiallyStyledCount: number; // Layers with overrides
  unstyledCount: number; // Layers without style

  // Library Distribution
  libraryDistribution: Record<string, number>; // Library name → layer count

  // Token Metrics
  tokenAdoptionRate: number; // % of layers using tokens (0-100)
  tokenCoverageRate: number; // % of design tokens that are actively used (0-100) - see spec Metrics Definitions for distinction
  tokenUsageCount: number; // Total token usages across all layers
  mixedUsageCount: number; // Layers with both style and tokens

  // Top Styles
  topStyles: Array<{ styleId: string; styleName: string; usageCount: number }>;
  deprecatedStyleCount: number; // Count of deprecated style instances
}
```

**Figma API Mapping**:

- Source: Aggregated from traversal of `figma.root.children` (pages)
- Metadata: From `figma.currentPage`, `figma.editorType`
- Duration: Measured via performance.now()

**Relationships**:

- **HasMany** TextStyle (via styles array)
- **HasMany** DesignToken (via tokens array)
- **HasMany** TextLayer (via layers array)
- **HasMany** LibrarySource (via libraries array)

---

### 5. LibrarySource

Represents the origin of a text style (local document or external library).

**TypeScript Signature**:

```typescript
interface LibrarySource {
  // Identity
  id: string; // Library ID or "local"
  name: string; // Display name (e.g., "Design System v2" or "Local")
  type: 'local' | 'team_library' | 'published_component';

  // Status
  isEnabled: boolean; // True if library currently enabled
  isAvailable: boolean; // True if network-accessible

  // Content
  styleCount: number; // Total styles from this source
  styleIds: string[]; // Style IDs from this source

  // Usage
  totalUsageCount: number; // Total layers using styles from this source
  usagePercentage: number; // % of total styled layers (0-100)
}
```

**Figma API Mapping**:

- Source: From `figma.teamLibrary.getAvailableLibraryVariablesAsync()` or style.remote
- Enabled status: From enabled libraries list
- Availability: From network access checks

**Relationships**:

- **HasMany** TextStyle (via styleIds)

---

## Supporting Types

### TokenBinding

Represents a single token binding on a text layer.

```typescript
interface TokenBinding {
  property: 'fills' | 'fontFamily' | 'fontSize' | 'lineHeight' | 'letterSpacing';
  tokenId: string; // Variable ID
  tokenName: string; // Resolved token name
  tokenValue: any; // Resolved value
}
```

---

## State Machines

### AuditState

Represents the state of an audit operation.

```typescript
type AuditState =
  | 'idle' // No audit running; UI shows "Run Audit"
  | 'validating' // Checking document accessibility, counting layers
  | 'scanning' // Traversing pages, discovering text nodes
  | 'processing' // Extracting metadata, building inventory
  | 'complete' // Audit finished successfully
  | 'error' // Audit failed with error
  | 'cancelled'; // User cancelled operation

// Valid Transitions
const AUDIT_TRANSITIONS: Record<AuditState, AuditState[]> = {
  idle: ['validating'],
  validating: ['scanning', 'error'],
  scanning: ['processing', 'error', 'cancelled'],
  processing: ['complete', 'error', 'cancelled'],
  complete: ['idle'],
  error: ['idle'],
  cancelled: ['idle'],
};
```

**UI Mapping**:

- `idle`: Show "Run Audit" button
- `validating`: Show indeterminate spinner "Validating..."
- `scanning`: Show progress bar "Scanning page X of Y"
- `processing`: Show progress bar "Processing layer X of Y"
- `complete`: Show results with success checkmark
- `error`: Show error message with "Retry" button
- `cancelled`: Brief "Cancelled" message, return to idle

---

### ReplacementState

Represents the state of a style/token replacement operation.

```typescript
type ReplacementState =
  | 'idle' // No replacement running
  | 'validating' // Checking source/target validity, counting layers
  | 'creating_checkpoint' // Creating version history savepoint
  | 'processing' // Applying changes in batches
  | 'complete' // Replacement finished successfully
  | 'error'; // Replacement failed with error

// Valid Transitions (no 'cancelled' - replacements cannot be cancelled after checkpoint)
const REPLACEMENT_TRANSITIONS: Record<ReplacementState, ReplacementState[]> = {
  idle: ['validating'],
  validating: ['creating_checkpoint', 'error'],
  creating_checkpoint: ['processing', 'error'],
  processing: ['complete', 'error'],
  complete: ['idle'],
  error: ['idle'],
};
```

**UI Mapping**:

- `idle`: Show "Replace" button in detail panel
- `validating`: Show indeterminate spinner "Validating..."
- `creating_checkpoint`: Show indeterminate spinner "Creating checkpoint..."
- `processing`: Show progress bar "Replacing batch X of Y"
- `complete`: Show success message, invalidate audit
- `error`: Show error with rollback option

---

### BatchProcessorState

Tracks adaptive batch sizing for replacement operations.

```typescript
interface BatchProcessorState {
  currentBatchSize: number; // Current batch size (25-100)
  consecutiveSuccesses: number; // Count of successful batches since last resize
  totalLayersProcessed: number; // Total layers updated so far
  totalLayersToProcess: number; // Total layers in operation
  failedLayers: FailedLayer[]; // Layers that failed to update
}

interface FailedLayer {
  layerId: string;
  layerName: string;
  reason: string; // Error message
  retryCount: number; // Times retried
}

// Batch Size Algorithm
// - Start: 100 layers/batch
// - On error: Reduce to 25 layers/batch
// - After 5 consecutive successes: Increase by 25 (max 100)
```

---

## Relationship Diagram

```
LibrarySource
  ↓ (1:N)
TextStyle
  ↓ (1:N)         ↓ (N:M - hierarchy)
TextLayer  ←—————— TextStyle (parent/child)
  ↓ (1:N)
TokenBinding
  ↓ (N:1)
DesignToken

AuditResult
  ↓ (aggregates)
  ├── TextStyle[]
  ├── DesignToken[]
  ├── TextLayer[]
  ├── LibrarySource[]
  └── AuditMetrics
```

---

## Design Decisions

### Why TextLayer includes redundant data (styleName, styleSource)?

**Rationale**: Denormalization for performance. Avoids N lookups during export and rendering. AuditResult is ephemeral (not persisted), so redundancy acceptable.

### Why separate TokenBinding from DesignToken?

**Rationale**: A token can be used by multiple layers on different properties. TokenBinding represents the specific usage instance, DesignToken represents the token definition.

### Why styleHierarchy separate from styles array?

**Rationale**: Hierarchy is computed (derived from names), not stored in Figma. Separating allows efficient tree rendering without recomputing on every render.

### Why track parentType and componentPath on TextLayer?

**Rationale**: Enables "component vs plain layer" breakdown in analytics. Critical for understanding adoption patterns in design systems.

### Why no persistence layer?

**Rationale**: Figma document is source of truth. Audit results are ephemeral snapshots. Re-running audit is fast (<30s for 1000 layers), so caching unnecessary.

---

## Version History

- **v1.0** (2025-11-20): Initial data model for MVP
  - 5 core entities
  - 2 state machines (Audit, Replacement)
  - Support for styles + tokens
  - Hierarchy detection
  - Token chain placeholders (full implementation deferred to v1.1)
