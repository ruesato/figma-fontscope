# Data Model: Font Audit Plugin

**Feature Branch**: `001-font-audit` | **Date**: 2025-11-16 | **Plan**: [plan.md](./plan.md)

## Overview

This document defines the data structures used throughout the Font Audit Plugin, including entity definitions, validation rules, type specifications, and relationships. All entities are designed for in-memory processing only (no persistent storage per NFR-001).

## Core Entities

### 1. TextLayerData

Represents a single discovered text node with complete metadata for analysis and display.

**TypeScript Definition:**

```typescript
interface TextLayerData {
  // Identity
  id: string;                    // Figma node ID (unique identifier)
  content: string;               // Text content (first 50 chars for preview)

  // Font Properties
  fontFamily: string;            // Font family name (e.g., "Inter")
  fontSize: number;              // Font size in pixels
  fontWeight: number;            // Font weight (100-900)
  lineHeight: LineHeight;        // Line height (pixels, %, or "AUTO")

  // Visual Properties
  color: RGBA;                   // Text color {r, g, b, a}
  opacity: number;               // Layer opacity (0-1)
  visible: boolean;              // Visibility state

  // Context
  componentContext: ComponentContext;    // Component hierarchy info
  styleAssignment: StyleAssignment;      // Text style application details
  matchSuggestions?: StyleMatchSuggestion[];  // Close style matches (only for unstyled text)
}
```

**Field Specifications:**

| Field | Type | Required | Validation | Source |
|-------|------|----------|------------|--------|
| `id` | string | Yes | Non-empty, valid Figma node ID | `node.id` |
| `content` | string | Yes | Max 50 chars, truncated with "..." | `node.characters.substring(0, 50)` |
| `fontFamily` | string | Yes | Non-empty string | `node.fontName.family` |
| `fontSize` | number | Yes | Positive number | `node.fontSize` |
| `fontWeight` | number | Yes | 100-900 range | `node.fontName.weight` |
| `lineHeight` | LineHeight | Yes | See LineHeight type | `node.lineHeight` |
| `color` | RGBA | Yes | Valid RGBA object | `node.fills[0].color` if solid |
| `opacity` | number | Yes | 0-1 range | `node.opacity` |
| `visible` | boolean | Yes | Boolean | `node.visible` |
| `componentContext` | ComponentContext | Yes | Valid ComponentContext | Derived from hierarchy |
| `styleAssignment` | StyleAssignment | Yes | Valid StyleAssignment | Derived from text style |
| `matchSuggestions` | StyleMatchSuggestion[] | No | Only if unstyled and matches exist | Calculated similarity |

**Validation Rules:**

- If `node.characters` contains mixed fonts (detected via `figma.mixed`), capture first font and set `fontFamily` to `"Mixed Fonts"`
- If `node.fills` is empty or non-solid, set `color` to `{r: 0, g: 0, b: 0, a: 0}` with special flag
- `matchSuggestions` array MUST be empty if `styleAssignment.assignmentStatus` is not `'unstyled'`
- All numeric fields rounded to 2 decimal places for display, but stored with full precision

**Relationships:**

- Contains one `ComponentContext` (embedded)
- Contains one `StyleAssignment` (embedded)
- May contain 0-5 `StyleMatchSuggestion` objects (array)

---

### 2. StyleAssignment

Captures text style application status and property match details.

**TypeScript Definition:**

```typescript
interface StyleAssignment {
  assignmentStatus: 'fully-styled' | 'partially-styled' | 'unstyled';
  styleId?: string;              // Figma style ID (required if styled)
  styleName?: string;            // Full style name (e.g., "Body/Large")
  libraryName?: string;          // Library source (e.g., "Core Design System")
  propertyMatches?: PropertyMatchMap;  // Which properties match/differ
}

interface PropertyMatchMap {
  fontFamily: boolean;
  fontSize: boolean;
  fontWeight: boolean;
  lineHeight: boolean;
  color: boolean;
}
```

**Field Specifications:**

| Field | Type | Required | Validation | Source |
|-------|------|----------|------------|--------|
| `assignmentStatus` | enum | Yes | One of 3 values | Derived from style detection |
| `styleId` | string | Conditional | Required if styled/partially-styled | `node.getRangeTextStyleId(0, 1)` |
| `styleName` | string | Conditional | Required if styleId present | `figma.getStyleById(styleId).name` |
| `libraryName` | string | Conditional | Required if styleId present | `figma.getStyleById(styleId).remote?.key` |
| `propertyMatches` | PropertyMatchMap | Conditional | Required if partially-styled | Property comparison logic |

**Assignment Status Rules:**

1. **`fully-styled`**: Text has a text style applied (`styleId` present) AND all style properties match exactly
2. **`partially-styled`**: Text has a text style applied (`styleId` present) BUT some properties differ (local overrides)
3. **`unstyled`**: Text has no text style applied (`styleId` is null or empty)

**Validation Rules:**

- If `assignmentStatus === 'unstyled'`, all optional fields MUST be `undefined`
- If `assignmentStatus === 'fully-styled'`, `propertyMatches` MUST show all `true` values
- If `assignmentStatus === 'partially-styled'`, `propertyMatches` MUST have at least one `false` value
- `libraryName` is `"Local Styles"` if style is not from a library (no `remote.key`)

**Relationships:**

- Embedded in `TextLayerData`
- References a Figma text style (external to data model)

---

### 3. StyleMatchSuggestion

Represents a recommended text style for unstyled text based on similarity calculation.

**TypeScript Definition:**

```typescript
interface StyleMatchSuggestion {
  suggestedStyleId: string;      // Figma style ID
  suggestedStyleName: string;    // Full style name
  libraryName: string;           // Library source
  similarityScore: number;       // 0.80-1.00 (80%-100%)
  matchingProperties: string[];  // Properties that match
  differingProperties: DifferingProperty[];  // Properties that differ
}

interface DifferingProperty {
  property: 'fontFamily' | 'fontSize' | 'fontWeight' | 'lineHeight' | 'color';
  textValue: string;             // Current text property value
  styleValue: string;            // Style property value
}
```

**Field Specifications:**

| Field | Type | Required | Validation | Source |
|-------|------|----------|------------|--------|
| `suggestedStyleId` | string | Yes | Valid Figma style ID | Style library |
| `suggestedStyleName` | string | Yes | Non-empty string | `figma.getStyleById(id).name` |
| `libraryName` | string | Yes | Non-empty string | Library metadata |
| `similarityScore` | number | Yes | 0.80-1.00 range | Weighted calculation |
| `matchingProperties` | string[] | Yes | Non-empty array | Property comparison |
| `differingProperties` | DifferingProperty[] | Yes | May be empty if 100% match | Property comparison |

**Similarity Calculation:**

Weighted formula (see [research.md](./research.md) for implementation details):
- Font family: 30%
- Font size: 30%
- Font weight: 20%
- Line height: 15%
- Color: 5%

**Validation Rules:**

- `similarityScore` MUST be ≥ 0.80 (80% threshold for suggestions)
- `matchingProperties` + `differingProperties` MUST cover all 5 properties
- Suggestions sorted by `similarityScore` descending (highest first)
- Maximum 5 suggestions per unstyled text layer

**Relationships:**

- Array embedded in `TextLayerData.matchSuggestions`
- References a Figma text style (external to data model)

---

### 4. ComponentContext

Captures component hierarchy and instance metadata.

**TypeScript Definition:**

```typescript
interface ComponentContext {
  componentType: 'main-component' | 'instance' | 'plain';
  hierarchyPath: string[];       // Breadcrumb path (e.g., ["Page", "Card", "Button", "Label"])
  overrideStatus: 'default' | 'overridden' | 'detached';
}
```

**Field Specifications:**

| Field | Type | Required | Validation | Source |
|-------|------|----------|------------|--------|
| `componentType` | enum | Yes | One of 3 values | Node type detection |
| `hierarchyPath` | string[] | Yes | Non-empty array (min 1 element) | Hierarchy traversal |
| `overrideStatus` | enum | Yes | One of 3 values | Instance property check |

**Component Type Rules:**

1. **`main-component`**: Node is a ComponentNode (master component definition)
2. **`instance`**: Node is an InstanceNode (component instance)
3. **`plain`**: Node is in a regular frame/group (not component-related)

**Override Status Rules:**

1. **`default`**: Instance text matches main component (no overrides)
2. **`overridden`**: Instance text has local property overrides
3. **`detached`**: Instance is detached from main component

**Hierarchy Path Construction:**

- For plain text: `["Page Name", "Frame Name", "Text Layer Name"]`
- For component text: `["Page Name", "Component Name", "Nested Component", "Text Layer Name"]`
- Traverse parent nodes until reaching PageNode
- Reverse order for display (root → leaf)

**Validation Rules:**

- `hierarchyPath[0]` is always the page name
- `hierarchyPath[hierarchyPath.length - 1]` is the text layer name
- If `componentType === 'plain'`, `overrideStatus` MUST be `'default'`

**Relationships:**

- Embedded in `TextLayerData`

---

### 5. AuditResult

Complete audit output containing all discovered text layers and summary metrics.

**TypeScript Definition:**

```typescript
interface AuditResult {
  textLayers: TextLayerData[];   // All discovered text nodes
  summary: AuditSummary;         // Aggregate metrics
  timestamp: string;             // ISO 8601 format
  fileName: string;              // Figma file name
}
```

**Field Specifications:**

| Field | Type | Required | Validation | Source |
|-------|------|----------|------------|--------|
| `textLayers` | TextLayerData[] | Yes | Array (may be empty) | Audit traversal |
| `summary` | AuditSummary | Yes | Valid AuditSummary | Calculated from textLayers |
| `timestamp` | string | Yes | ISO 8601 format | `new Date().toISOString()` |
| `fileName` | string | Yes | Non-empty string | `figma.root.name` |

**Validation Rules:**

- `textLayers.length` MUST match `summary.totalTextLayers`
- `timestamp` format: `YYYY-MM-DDTHH:mm:ss.sssZ`
- If `textLayers` is empty, display empty state message

**Relationships:**

- Contains array of `TextLayerData` (0-10,000 items)
- Contains one `AuditSummary` (embedded)

---

### 6. AuditSummary

Aggregate metrics calculated from audit results.

**TypeScript Definition:**

```typescript
interface AuditSummary {
  totalTextLayers: number;       // Total text nodes discovered
  uniqueFontFamilies: number;    // Distinct font families
  styleCoveragePercent: number;  // % of text with styles applied (0-100)
  librariesInUse: string[];      // Library names
  potentialMatchesCount: number; // Unstyled text with ≥80% matches
  hiddenLayersCount: number;     // Invisible text layers
}
```

**Field Specifications:**

| Field | Type | Required | Validation | Source |
|-------|------|----------|------------|--------|
| `totalTextLayers` | number | Yes | Non-negative integer | `textLayers.length` |
| `uniqueFontFamilies` | number | Yes | Positive integer | `new Set(textLayers.map(t => t.fontFamily)).size` |
| `styleCoveragePercent` | number | Yes | 0-100 range | Calculated from assignmentStatus |
| `librariesInUse` | string[] | Yes | Array of unique library names | From styleAssignment |
| `potentialMatchesCount` | number | Yes | Non-negative integer | Count unstyled with matchSuggestions |
| `hiddenLayersCount` | number | Yes | Non-negative integer | Count where visible === false |

**Calculation Rules:**

```typescript
// Style coverage percentage
const styledCount = textLayers.filter(
  t => t.styleAssignment.assignmentStatus !== 'unstyled'
).length;
const styleCoveragePercent = (styledCount / totalTextLayers) * 100;

// Potential matches count
const potentialMatchesCount = textLayers.filter(
  t => t.matchSuggestions && t.matchSuggestions.length > 0
).length;

// Hidden layers count
const hiddenLayersCount = textLayers.filter(t => !t.visible).length;
```

**Validation Rules:**

- All counts MUST be ≥ 0
- `styleCoveragePercent` rounded to 1 decimal place for display
- `librariesInUse` includes "Local Styles" if local styles are used
- Empty arrays are valid (e.g., no libraries in use)

**Relationships:**

- Embedded in `AuditResult`

---

## Supporting Types

### LineHeight

Figma line height can be pixels, percentage, or auto.

```typescript
type LineHeight =
  | { value: number; unit: 'PIXELS' }
  | { value: number; unit: 'PERCENT' }
  | { unit: 'AUTO' };
```

**Normalization for Similarity:**

- `PIXELS`: Use raw value
- `PERCENT`: Convert to pixels using `(percent / 100) * fontSize`
- `AUTO`: Treat as 120% of fontSize (Figma default)

---

### RGBA

Standard RGBA color representation.

```typescript
interface RGBA {
  r: number;  // 0-1 range
  g: number;  // 0-1 range
  b: number;  // 0-1 range
  a: number;  // 0-1 range (opacity)
}
```

**Display Format:**

- Convert to hex: `#RRGGBB` (if a === 1)
- Convert to rgba: `rgba(R, G, B, A)` (if a < 1)

---

## Message Protocol Types

### UI → Main Messages

```typescript
type UIToMainMessage =
  | { type: 'RUN_AUDIT'; scope: 'page' | 'selection' }
  | { type: 'NAVIGATE_TO_LAYER'; layerId: string }
  | { type: 'CANCEL_AUDIT' };
```

### Main → UI Messages

```typescript
type MainToUIMessage =
  | { type: 'AUDIT_STARTED' }
  | { type: 'AUDIT_PROGRESS'; progress: number; current: number; total: number }
  | { type: 'AUDIT_COMPLETE'; result: AuditResult }
  | { type: 'AUDIT_ERROR'; error: string; errorType: 'VALIDATION' | 'API' | 'UNKNOWN' }
  | { type: 'NAVIGATE_SUCCESS' }
  | { type: 'NAVIGATE_ERROR'; error: string };
```

See [contracts/plugin-protocol.md](./contracts/plugin-protocol.md) for complete message flow specifications.

---

## Export Format Types

### PDF Export Structure

```typescript
interface PDFExportData {
  summary: AuditSummary;
  fontInventory: FontInventoryEntry[];
  styleUsage: StyleUsageEntry[];
  unstyledText: TextLayerData[];
  partialMatches: TextLayerData[];
  componentBreakdown: ComponentBreakdownEntry[];
  hiddenText: TextLayerData[];
}

interface FontInventoryEntry {
  fontFamily: string;
  weights: number[];
  sizes: number[];
  layerCount: number;
}

interface StyleUsageEntry {
  libraryName: string;
  styleName: string;
  usageCount: number;
}

interface ComponentBreakdownEntry {
  componentType: 'main-component' | 'instance' | 'plain';
  count: number;
  percentage: number;
}
```

### CSV Export Structure

Flat table format with one row per text layer:

```typescript
interface CSVRow {
  'Layer ID': string;
  'Content': string;
  'Font Family': string;
  'Font Size': number;
  'Font Weight': number;
  'Line Height': string;
  'Color': string;
  'Opacity': number;
  'Visible': boolean;
  'Component Type': string;
  'Hierarchy Path': string;
  'Override Status': string;
  'Style Assignment': string;
  'Style Name': string;
  'Library Name': string;
  'Close Match': string;
  'Similarity': string;
}
```

---

## Error Handling Types

```typescript
interface AuditError {
  nodeId?: string;              // Failed node ID (if applicable)
  errorType: 'VALIDATION' | 'API' | 'UNKNOWN';
  message: string;
  retryAttempts?: number;       // Number of retries attempted
}
```

**Error Types:**

- `VALIDATION`: Pre-audit validation failed (e.g., >10,000 layers)
- `API`: Figma API error (e.g., font loading failure)
- `UNKNOWN`: Unexpected error

---

## Data Flow Summary

```
1. User triggers audit → RUN_AUDIT message
2. Main context:
   - Traverse nodes → TextLayerData[]
   - Calculate similarity → StyleMatchSuggestion[]
   - Aggregate metrics → AuditSummary
   - Build AuditResult
3. Main → UI: AUDIT_COMPLETE with AuditResult
4. UI context:
   - Display results (hierarchical views)
   - Enable search/filter (in-memory operations)
   - Export to PDF/CSV (PDFExportData/CSVRow)
```

---

## Validation Checklist

Before implementation, verify:

- [ ] All required fields have clear validation rules
- [ ] Relationships between entities are well-defined
- [ ] TypeScript types match Figma Plugin API types
- [ ] Similarity calculation inputs are normalized (0-1 scale)
- [ ] Message protocol types are exhaustive (no missing cases)
- [ ] Error handling covers all edge cases
- [ ] Export formats contain all audit data
- [ ] No persistent storage (in-memory only per NFR-001)
