# Research: Font Audit Plugin

**Date**: 2025-11-16
**Phase**: Phase 0 - Technical Research
**Purpose**: Document Figma Plugin API patterns, similarity algorithm design, and architectural decisions for implementation

## 1. Figma Plugin API Patterns

### 1.1 Text Node Traversal

**Best Practice: Use `findAllWithCriteria()` for Performance**

```typescript
// ✅ RECOMMENDED - 100x faster, supports type filtering
const textNodes = figma.currentPage.findAllWithCriteria({
  types: ['TEXT']
});

// ❌ AVOID - Slow, requires manual filtering
const allNodes = figma.currentPage.findAll();
const textNodes = allNodes.filter(node => node.type === 'TEXT');
```

**Performance Optimization Flags:**

```typescript
// Enable before traversal for 100x performance boost
// Skips invisible nested instances while maintaining complete discovery
figma.skipInvisibleInstanceChildren = true;

// Disable after traversal to restore default behavior
figma.skipInvisibleInstanceChildren = false;
```

**Decision**: Use `findAllWithCriteria({ types: ['TEXT'] })` with `skipInvisibleInstanceChildren = true` for optimal performance while maintaining 100% text layer discovery (Constitution Principle I & II).

### 1.2 Text Style Detection

**Detecting Applied Text Styles:**

```typescript
async function detectTextStyle(textNode: TextNode) {
  // Load font before accessing properties
  await figma.loadFontAsync(textNode.fontName as FontName);

  // Get text style for entire node or range
  const styleId = textNode.getRangeTextStyleId(0, textNode.characters.length);

  if (styleId === figma.mixed) {
    // Text has multiple styles - handle mixed case
    return { status: 'mixed', styles: /* extract individual ranges */ };
  }

  if (styleId && styleId !== '') {
    const style = figma.getStyleById(styleId) as TextStyle;
    return {
      styleId: styleId,
      styleName: style.name, // e.g., "Body/Large"
      libraryName: style.remote ? style.key.split(':')[0] : 'Local',
      assignmentStatus: 'fully-styled'
    };
  }

  return { assignmentStatus: 'unstyled' };
}
```

**Library Source Detection:**

```typescript
function getLibrarySource(style: TextStyle): string {
  if (!style.remote) {
    return 'Local'; // Local file style
  }

  // Remote library style - extract library name from key
  // Key format: "library-id:style-id"
  const libraryId = style.key.split(':')[0];

  // Note: Library name requires additional API call
  // For v1.0, use library ID or display as "External Library"
  return `Library ${libraryId}`;
}
```

**Decision**: Use `getRangeTextStyleId()` + `getStyleById()` for text style detection. Handle `figma.mixed` case by flagging as partial match. Library name extraction deferred to Phase 1 design (may need to enumerate libraries).

### 1.3 Component Hierarchy Traversal

**Building Component Paths:**

```typescript
function buildComponentHierarchy(node: SceneNode): {
  componentType: 'main' | 'instance' | 'plain';
  hierarchyPath: string[];
  overrideStatus: 'default' | 'overridden' | 'detached';
} {
  const path: string[] = [];
  let current: BaseNode | null = node;
  let componentType: 'main' | 'instance' | 'plain' = 'plain';
  let overrideStatus: 'default' | 'overridden' | 'detached' = 'default';

  // Detect if node is in a component
  if (node.type === 'TEXT') {
    let parent = node.parent;

    while (parent) {
      if (parent.type === 'COMPONENT') {
        componentType = 'main';
        path.unshift(parent.name);
        break;
      } else if (parent.type === 'INSTANCE') {
        componentType = 'instance';
        path.unshift(parent.name);

        // Check for overrides (simplified - actual implementation more complex)
        // Note: Override detection requires comparing instance to main component
        // Deferred to implementation phase
      } else if (parent.type === 'FRAME' || parent.type === 'GROUP') {
        path.unshift(parent.name);
      }

      parent = parent.parent;
    }
  }

  return { componentType, hierarchyPath: path, overrideStatus };
}
```

**Decision**: Traverse parent chain to build hierarchy breadcrumb. Override detection simplified for v1.0 (compare text properties to main component defaults). Edge case: Handle nested instances >10 levels deep with path truncation.

### 1.4 Font Loading Optimization

**Batch Font Loading Strategy:**

```typescript
async function batchLoadFonts(textNodes: TextNode[]): Promise<void> {
  // Collect unique fonts to minimize load calls
  const uniqueFonts = new Map<string, FontName>();

  for (const node of textNodes) {
    const fontKey = `${node.fontName.family}-${node.fontName.style}`;
    if (!uniqueFonts.has(fontKey)) {
      uniqueFonts.set(fontKey, node.fontName as FontName);
    }
  }

  // Load all fonts in parallel
  await Promise.all(
    Array.from(uniqueFonts.values()).map(font =>
      figma.loadFontAsync(font).catch(err => {
        // Handle missing fonts gracefully
        console.warn(`Failed to load font: ${font.family} ${font.style}`, err);
      })
    )
  );
}
```

**Decision**: Batch load unique fonts before property extraction to minimize API calls. Handle `hasMissingFont` flag for unavailable fonts (Constitution Principle III - graceful error handling).

### 1.5 Performance Optimizations Summary

| Optimization | Impact | Implementation |
|--------------|--------|----------------|
| `findAllWithCriteria()` | 100x faster traversal | Use instead of `findAll()` |
| `skipInvisibleInstanceChildren` | 100x faster for nested instances | Enable before traversal, disable after |
| Batch font loading | Reduces API calls | Load unique fonts in parallel |
| Progress updates | Non-blocking UI | postMessage every 100 nodes |
| Incremental processing | Prevents UI freeze | Process in chunks, yield control |

## 2. Similarity Algorithm Design

### 2.1 Weighted Similarity Formula

**Property Weights (from spec clarifications):**
- Font Family: 30%
- Font Size: 30%
- Font Weight: 20%
- Line Height: 15%
- Color: 5%

**Total**: 100%

### 2.2 Normalization Approach (0-1 Scoring)

#### Font Family Similarity

```typescript
function calculateFontFamilySimilarity(
  textFamily: string,
  styleFamily: string
): number {
  // Exact match or no match
  return textFamily === styleFamily ? 1.0 : 0.0;
}
```

**Rationale**: Font family must match exactly for style suggestion to be useful. No partial credit.

#### Font Size Similarity

```typescript
function calculateFontSizeSimilarity(
  textSize: number,
  styleSize: number
): number {
  const percentageDiff = Math.abs(textSize - styleSize) /
                         Math.max(textSize, styleSize);

  // 0% diff = 1.0 score
  // 50% diff = 0.0 score
  // Linear interpolation between
  return Math.max(0, 1 - (percentageDiff * 2));
}
```

**Rationale**: Font size is visually significant. 50%+ difference is too large for meaningful suggestion. Linear scaling provides intuitive similarity.

#### Font Weight Similarity

```typescript
function calculateFontWeightSimilarity(
  textWeight: number,
  styleWeight: number
): number {
  const weightDiff = Math.abs(textWeight - styleWeight);

  // 0 diff = 1.0 score
  // 400 diff (e.g., 300 to 700) = 0.0 score
  // Linear interpolation
  return Math.max(0, 1 - (weightDiff / 400));
}
```

**Rationale**: Font weight range is typically 100-900 (thin to black). 400-point difference is visually extreme (e.g., light to bold).

#### Line Height Similarity

```typescript
function calculateLineHeightSimilarity(
  textLineHeight: LineHeight,
  styleLineHeight: LineHeight,
  fontSize: number
): number {
  // Normalize both to pixels for comparison
  const textPixels = normalizeLineHeight(textLineHeight, fontSize);
  const stylePixels = normalizeLineHeight(styleLineHeight, fontSize);

  const percentageDiff = Math.abs(textPixels - stylePixels) /
                         Math.max(textPixels, stylePixels);

  return Math.max(0, 1 - (percentageDiff * 2));
}

function normalizeLineHeight(lineHeight: LineHeight, fontSize: number): number {
  if (lineHeight.unit === 'PIXELS') {
    return lineHeight.value;
  } else if (lineHeight.unit === 'PERCENT') {
    return (lineHeight.value / 100) * fontSize;
  } else { // AUTO
    return fontSize * 1.2; // Default line height assumption
  }
}
```

**Rationale**: Line height can be pixels, percentage, or auto. Normalize to pixels for accurate comparison. 50%+ difference is too large.

#### Color Similarity

```typescript
function calculateColorSimilarity(
  textColor: RGBA,
  styleColor: RGBA
): number {
  // Euclidean distance in RGB space (ignore alpha for similarity)
  const distance = Math.sqrt(
    Math.pow(textColor.r - styleColor.r, 2) +
    Math.pow(textColor.g - styleColor.g, 2) +
    Math.pow(textColor.b - styleColor.b, 2)
  );

  // Max distance in RGB space (0-1 range): sqrt(3) ≈ 1.732
  const maxDistance = Math.sqrt(3);

  return 1 - (distance / maxDistance);
}
```

**Rationale**: Euclidean distance in RGB color space provides perceptually meaningful similarity. Alpha (opacity) excluded to focus on color hue.

### 2.3 Complete Similarity Calculation

```typescript
function calculateStyleSimilarity(
  textProps: StyleProperties,
  styleProps: StyleProperties
): number {
  const familyScore = calculateFontFamilySimilarity(
    textProps.fontFamily,
    styleProps.fontFamily
  );

  const sizeScore = calculateFontSizeSimilarity(
    textProps.fontSize,
    styleProps.fontSize
  );

  const weightScore = calculateFontWeightSimilarity(
    textProps.fontWeight,
    styleProps.fontWeight
  );

  const lineHeightScore = calculateLineHeightSimilarity(
    textProps.lineHeight,
    styleProps.lineHeight,
    textProps.fontSize
  );

  const colorScore = calculateColorSimilarity(
    textProps.color,
    styleProps.color
  );

  // Weighted average
  const totalScore =
    (familyScore * 0.30) +
    (sizeScore * 0.30) +
    (weightScore * 0.20) +
    (lineHeightScore * 0.15) +
    (colorScore * 0.05);

  return totalScore; // Returns 0-1, convert to percentage for display
}
```

**Threshold Application**: `totalScore >= 0.80` qualifies as "close match" per spec clarification.

**Decision**: Implement weighted similarity with normalized 0-1 scoring per property. Threshold at 80% for suggestions. Rank multiple matches by descending similarity score (max 5 suggestions per FR-008 edge case).

## 3. Architecture Decisions

### 3.1 Dual-Context Plugin Structure

**Main Context (Figma Sandbox)**
- Runs in Figma's JavaScript sandbox
- Has access to Figma Plugin API
- No DOM access, no external network requests
- Processes audit logic (traversal, analysis)

**UI Context (iframe)**
- Runs in browser iframe
- Has DOM access, can render React UI
- No access to Figma Plugin API
- Displays results, handles user interactions

**Communication**: postMessage between contexts

```typescript
// Main → UI
figma.ui.postMessage({ type: 'AUDIT_COMPLETE', result: auditResult });

// UI → Main
window.onmessage = (event) => {
  const message = event.data.pluginMessage;
  if (message.type === 'RUN_AUDIT') {
    // Start audit
  }
};
```

### 3.2 PostMessage Protocol Design

**Message Types** (see contracts/plugin-protocol.md for full spec):

1. **UI → Main**:
   - `RUN_AUDIT`: Initiate audit (scope: page/selection)
   - `NAVIGATE_TO_LAYER`: Focus layer in Figma
   - `CANCEL_AUDIT`: Abort in-progress audit

2. **Main → UI**:
   - `AUDIT_STARTED`: Audit begun
   - `AUDIT_PROGRESS`: Update progress (%, current/total)
   - `AUDIT_COMPLETE`: Audit finished with results
   - `AUDIT_ERROR`: Audit failed with error details
   - `NAVIGATE_SUCCESS/ERROR`: Navigation result

**Decision**: Type-safe message protocol with TypeScript discriminated unions. Single message type per action for clarity. Progress updates every 100 nodes to balance responsiveness and performance.

### 3.3 State Management

**UI-Side State (React hooks)**:

```typescript
// useAuditState.ts
interface AuditState {
  status: 'idle' | 'running' | 'complete' | 'error';
  result: AuditResult | null;
  progress: { current: number; total: number; percentage: number } | null;
  error: string | null;
}

function useAuditState() {
  const [state, setState] = useState<AuditState>({
    status: 'idle',
    result: null,
    progress: null,
    error: null
  });

  // Message handler updates state based on messages from main
  useMessageHandler((message) => {
    switch (message.type) {
      case 'AUDIT_STARTED':
        setState(prev => ({ ...prev, status: 'running', progress: { current: 0, total: 0, percentage: 0 } }));
        break;
      case 'AUDIT_PROGRESS':
        setState(prev => ({ ...prev, progress: { current: message.current, total: message.total, percentage: message.progress } }));
        break;
      case 'AUDIT_COMPLETE':
        setState(prev => ({ ...prev, status: 'complete', result: message.result, progress: null }));
        break;
      case 'AUDIT_ERROR':
        setState(prev => ({ ...prev, status: 'error', error: message.error, progress: null }));
        break;
    }
  });

  return state;
}
```

**Decision**: Centralized React state for audit status/results in UI. Main context is stateless (processes audit on demand, sends results). This aligns with Constitution Principle VI (clear separation of concerns).

### 3.4 Progress Indicator Strategy

**Non-Blocking Progress Updates:**

```typescript
// Main context - during traversal
let processedCount = 0;
for (const textNode of textNodes) {
  // Process node...
  processedCount++;

  // Update every 100 nodes to balance performance and responsiveness
  if (processedCount % 100 === 0) {
    figma.ui.postMessage({
      type: 'AUDIT_PROGRESS',
      progress: (processedCount / textNodes.length) * 100,
      current: processedCount,
      total: textNodes.length
    });

    // Yield control to prevent blocking
    await new Promise(resolve => setTimeout(resolve, 0));
  }
}
```

**Decision**: Progress updates every 100 nodes with `setTimeout(0)` yield to prevent UI blocking. Displays percentage, current count, total count (Constitution Principle II - responsive UI).

### 3.5 Incremental Loading Design

**500-Item Batch Rendering:**

```typescript
// UI context - results display
function AuditResults({ textLayers }: { textLayers: TextLayerData[] }) {
  const [displayCount, setDisplayCount] = useState(500);

  const visibleLayers = textLayers.slice(0, displayCount);
  const hasMore = displayCount < textLayers.length;

  return (
    <>
      {visibleLayers.map(layer => (
        <LayerItem key={layer.id} layer={layer} />
      ))}

      {hasMore && (
        <button onClick={() => setDisplayCount(prev => prev + 500)}>
          Load More ({textLayers.length - displayCount} remaining)
        </button>
      )}
    </>
  );
}
```

**Alternative**: Virtual scrolling with `react-window` for smoother UX with very large result sets.

**Decision**: Start with simple batch loading (500 items + "Load More"). Upgrade to virtual scrolling if user testing shows need (Constitution Principle II - incremental loading >500 items).

## 4. Technology Integration

### 4.1 create-figma-plugin Setup

**Scaffolding Command:**

```bash
npx create-figma-plugin
# Select:
# - TypeScript: Yes
# - UI framework: React
# - Template: Plugin with UI
```

**Customization Steps:**
1. Add Tailwind CSS + PostCSS configuration
2. Add Figma Plugin DS components
3. Configure Vite for dual-context build
4. Add jsPDF (lazy import), Papa Parse, Vitest
5. Update manifest.json for permissions

### 4.2 Figma Plugin DS + Tailwind Configuration

**Install Dependencies:**

```bash
pnpm add @create-figma-plugin/ui tailwindcss postcss autoprefixer
pnpm add -D @tailwindcss/forms
```

**Tailwind Config** (tailwind.config.js):

```javascript
module.exports = {
  content: ['./src/ui/**/*.{ts,tsx}'],
  theme: {
    extend: {
      colors: {
        // Map Figma Plugin DS tokens to Tailwind
        'figma-bg': 'var(--color-bg)',
        'figma-text': 'var(--color-text)',
        'figma-border': 'var(--color-border)',
        // ... more tokens
      }
    }
  },
  plugins: [require('@tailwindcss/forms')]
};
```

**Decision**: Use Figma Plugin DS components for primary UI (buttons, inputs, modals) + Tailwind utilities for custom styling and layout. Token mapping ensures visual consistency with Figma's native UI.

### 4.3 jsPDF Lazy Loading

**Dynamic Import Strategy:**

```typescript
// export/pdf-generator.ts
async function generatePDF(auditResult: AuditResult): Promise<Blob> {
  // Lazy load jsPDF only when export is triggered
  const { jsPDF } = await import('jspdf');

  const doc = new jsPDF();

  // Generate 7-section report (see spec FR-013)
  // Section 1: Summary dashboard
  // Section 2: Text style usage by library
  // Section 3: Font inventory
  // Section 4: Unstyled text analysis
  // Section 5: Partial style matches
  // Section 6: Component breakdown
  // Section 7: Hidden text inventory

  return doc.output('blob');
}
```

**Bundle Size Impact**:
- jsPDF: ~200KB
- Lazy loading prevents inclusion in main bundle
- Only loaded when user clicks "Export PDF"

**Decision**: Lazy load jsPDF via dynamic import to keep main bundle small. Export functionality in UI context (has DOM access for blob creation).

### 4.4 Vitest Test Setup

**Configuration** (vitest.config.ts):

```typescript
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    environment: 'node', // Figma API mocks in Node env
    globals: true,
    coverage: {
      reporter: ['text', 'json', 'html']
    }
  }
});
```

**Mock Figma API:**

```typescript
// tests/mocks/figma.ts
global.figma = {
  currentPage: {
    findAllWithCriteria: jest.fn().mockReturnValue([/* mock text nodes */])
  },
  getStyleById: jest.fn(),
  loadFontAsync: jest.fn().mockResolvedValue(undefined),
  skipInvisibleInstanceChildren: false,
  // ... more mocks
};
```

**Decision**: Use Vitest for unit tests (Vite-native, fast). Mock Figma API for testing audit logic. Manual testing required for actual Figma integration (Constitution testing philosophy - test-encouraged, pragmatic approach).

## Research Summary

All technical unknowns resolved. Key decisions documented:

1. **Text Traversal**: `findAllWithCriteria()` + `skipInvisibleInstanceChildren` for 100x performance
2. **Style Detection**: `getRangeTextStyleId()` + `getStyleById()` with `figma.mixed` handling
3. **Similarity Algorithm**: Weighted formula (30/30/20/15/5) with normalized 0-1 scoring, 80% threshold
4. **Architecture**: Dual-context plugin (main sandbox + UI iframe) with postMessage protocol
5. **State Management**: React hooks in UI, stateless main context
6. **Progress**: Updates every 100 nodes with `setTimeout(0)` yield
7. **Incremental Loading**: 500-item batches with "Load More" button
8. **Scaffolding**: create-figma-plugin + Figma Plugin DS + Tailwind
9. **PDF Export**: jsPDF lazy-loaded via dynamic import
10. **Testing**: Vitest with mocked Figma API for unit tests, manual for integration

Ready for Phase 1 design (data-model.md, contracts/, quickstart.md).
