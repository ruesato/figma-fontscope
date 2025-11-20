# Audit Engine API Contract

**Feature**: `002-style-governance`
**Created**: 2025-11-20
**Purpose**: Define the AuditEngine interface and audit operation contracts

## Overview

The Audit Engine is responsible for:
- Scanning all pages in the Figma document
- Discovering and cataloging text layers
- Extracting style and token metadata
- Building comprehensive audit results
- Managing audit state machine

---

## AuditEngine Interface

```typescript
interface AuditEngine {
  // Core Methods
  runAudit(options?: AuditOptions): Promise<AuditResult>;
  cancelAudit(): void;
  getState(): AuditState;

  // State Machine
  transitionTo(newState: AuditState): boolean;
  canTransitionTo(newState: AuditState): boolean;

  // Progress Callbacks
  onProgress(callback: ProgressCallback): void;
  onStateChange(callback: StateChangeCallback): void;

  // Lifecycle
  dispose(): void;
}
```

---

## Types

### AuditOptions

Configuration for audit operations.

```typescript
interface AuditOptions {
  // Scope
  includeHiddenLayers?: boolean;   // Default: true
  includeTokens?: boolean;         // Default: true

  // Performance
  batchSize?: number;              // Layers per progress update (default: 50)

  // Validation
  skipSizeValidation?: boolean;    // Skip 5k/25k warnings (default: false)

  // Progress
  progressCallback?: ProgressCallback;
  stateChangeCallback?: StateChangeCallback;
}
```

### AuditState

Current state of the audit operation (see data-model.md for full state machine).

```typescript
type AuditState =
  | 'idle'
  | 'validating'
  | 'scanning'
  | 'processing'
  | 'complete'
  | 'error'
  | 'cancelled';
```

### ProgressCallback

Called during audit to report progress.

```typescript
type ProgressCallback = (progress: AuditProgress) => void;

interface AuditProgress {
  state: AuditState;
  percentage: number;              // 0-100
  currentStep: string;             // Human-readable description
  pagesScanned?: number;           // If in scanning state
  totalPages?: number;
  layersProcessed?: number;        // If in processing state
  totalLayers?: number;
}
```

### StateChangeCallback

Called when audit state changes.

```typescript
type StateChangeCallback = (oldState: AuditState, newState: AuditState) => void;
```

---

## Method Specifications

### runAudit()

Executes a complete document audit.

**Signature**:
```typescript
async runAudit(options?: AuditOptions): Promise<AuditResult>
```

**Preconditions**:
- Current state must be `idle`
- Figma document must be accessible
- No other audit currently running

**State Transitions**:
```
idle → validating → scanning → processing → complete
                  ↓           ↓           ↓
                error       error       error
                  ↓           ↓           ↓
                cancelled   cancelled
```

**Steps**:

1. **Validation Phase** (`validating`):
   - Check document accessibility
   - Count total text layers
   - Enforce size limits (warn at 5k, block at 25k)
   - Validate options

2. **Scanning Phase** (`scanning`):
   - Traverse all pages
   - Discover all text nodes
   - Extract basic node properties
   - Build page index

3. **Processing Phase** (`processing`):
   - Extract style metadata for each layer
   - Detect token bindings
   - Build style hierarchy
   - Categorize layers (styled/unstyled)
   - Compute analytics

4. **Completion** (`complete`):
   - Assemble final AuditResult
   - Return result

**Error Handling**:
- **ValidationError**: Thrown if document exceeds 25k layers or validation fails
- **ScanningError**: Thrown if page traversal fails
- **ProcessingError**: Thrown if metadata extraction fails
- All errors transition to `error` state

**Returns**: `Promise<AuditResult>` - Complete audit data

**Throws**: `AuditError` with details

**Example**:
```typescript
const engine = new AuditEngine();

engine.onProgress((progress) => {
  console.log(`${progress.currentStep}: ${progress.percentage}%`);
});

try {
  const result = await engine.runAudit({
    includeHiddenLayers: true,
    includeTokens: true,
  });

  console.log(`Found ${result.totalTextLayers} text layers`);
  console.log(`Style adoption: ${result.metrics.styleAdoptionRate}%`);
} catch (error) {
  if (error.code === 'DOCUMENT_TOO_LARGE') {
    console.error('Document exceeds 25,000 text layers');
  }
}
```

---

### cancelAudit()

Cancels an in-progress audit operation.

**Signature**:
```typescript
cancelAudit(): void
```

**Preconditions**:
- Current state must be `scanning` or `processing`
- Cannot cancel during `validating` (too fast)
- Cannot cancel during `complete` (already done)

**State Transitions**:
```
scanning → cancelled → idle
processing → cancelled → idle
```

**Behavior**:
- Stops page traversal immediately
- Stops metadata extraction
- Discards partial results
- Transitions to `cancelled` state, then `idle`

**Example**:
```typescript
const engine = new AuditEngine();

// Start audit
engine.runAudit().catch(() => {
  // Cancellation results in rejection
});

// User clicks cancel after 2 seconds
setTimeout(() => {
  engine.cancelAudit();
}, 2000);
```

---

### getState()

Returns current audit state.

**Signature**:
```typescript
getState(): AuditState
```

**Returns**: Current state ('idle', 'validating', 'scanning', etc.)

**Example**:
```typescript
if (engine.getState() === 'idle') {
  // Safe to start new audit
  await engine.runAudit();
}
```

---

### transitionTo()

Attempts to transition to a new state.

**Signature**:
```typescript
transitionTo(newState: AuditState): boolean
```

**Parameters**:
- `newState`: Target state

**Returns**: `true` if transition succeeded, `false` if invalid transition

**Validation**: Checks against AUDIT_TRANSITIONS map (see data-model.md)

**Example**:
```typescript
// Internal usage
if (engine.transitionTo('scanning')) {
  // Start scanning
} else {
  throw new Error('Invalid state transition');
}
```

---

### canTransitionTo()

Checks if transition to new state is valid.

**Signature**:
```typescript
canTransitionTo(newState: AuditState): boolean
```

**Parameters**:
- `newState`: Target state

**Returns**: `true` if transition is valid from current state

**Example**:
```typescript
if (engine.canTransitionTo('cancelled')) {
  // Show cancel button
} else {
  // Hide cancel button
}
```

---

### onProgress()

Registers a progress callback.

**Signature**:
```typescript
onProgress(callback: ProgressCallback): void
```

**Parameters**:
- `callback`: Function called on progress updates

**Frequency**: Every 50 layers processed (or every page for scanning)

**Example**:
```typescript
engine.onProgress((progress) => {
  updateProgressBar(progress.percentage);
  updateStatusText(progress.currentStep);
});
```

---

### onStateChange()

Registers a state change callback.

**Signature**:
```typescript
onStateChange(callback: StateChangeCallback): void
```

**Parameters**:
- `callback`: Function called on state changes

**Example**:
```typescript
engine.onStateChange((oldState, newState) => {
  console.log(`Audit: ${oldState} → ${newState}`);

  if (newState === 'scanning') {
    showProgressIndicator();
  } else if (newState === 'complete') {
    hideProgressIndicator();
  }
});
```

---

### dispose()

Cleans up resources and resets state.

**Signature**:
```typescript
dispose(): void
```

**Behavior**:
- Cancels any running audit
- Clears callbacks
- Resets to `idle` state

**Example**:
```typescript
// Cleanup before unmounting
useEffect(() => {
  const engine = new AuditEngine();

  return () => {
    engine.dispose();
  };
}, []);
```

---

## Supporting Components

### Validator

Validates document before audit starts.

```typescript
interface Validator {
  validate(): Promise<ValidationResult>;
}

interface ValidationResult {
  isValid: boolean;
  totalTextLayers: number;
  totalPages: number;
  warnings: ValidationWarning[];
  errors: ValidationError[];
}

interface ValidationWarning {
  code: 'LARGE_DOCUMENT' | 'MANY_PAGES';
  message: string;
  layerCount?: number;
}

interface ValidationError {
  code: 'DOCUMENT_TOO_LARGE' | 'NO_ACCESS' | 'NO_TEXT_LAYERS';
  message: string;
}
```

**Validation Rules**:
- **Warning** at >5,000 text layers: "This document contains [X] text layers. Audit may take several minutes."
- **Error** at >25,000 text layers: "This document contains [X] text layers, which exceeds the maximum supported limit of 25,000."
- **Error** if no text layers: "No text layers found in document"

---

### Scanner

Traverses document pages and discovers text nodes.

```typescript
interface Scanner {
  scan(pages: PageNode[]): AsyncIterator<TextNode>;
}

// Usage
for await (const textNode of scanner.scan(figma.root.children)) {
  // Process each text node
}
```

---

### Processor

Extracts metadata from discovered text nodes.

```typescript
interface Processor {
  process(textNode: TextNode): Promise<TextLayer>;
}

// Extracts:
// - Style assignment
// - Token bindings
// - Component context
// - Page location
```

---

## Error Types

```typescript
class AuditError extends Error {
  code: AuditErrorCode;
  state: AuditState;
  details?: any;
}

type AuditErrorCode =
  | 'VALIDATION_FAILED'
  | 'DOCUMENT_TOO_LARGE'
  | 'NO_TEXT_LAYERS'
  | 'SCANNING_FAILED'
  | 'PROCESSING_FAILED'
  | 'OPERATION_CANCELLED'
  | 'UNKNOWN_ERROR';
```

---

## Performance Targets

| Metric | Target | Notes |
|--------|--------|-------|
| Validation | <1s | Quick size check |
| Scanning (1,000 layers) | 5-10s | Page traversal |
| Processing (1,000 layers) | 10-20s | Metadata extraction |
| **Total (1,000 layers)** | **<30s** | End-to-end |
| Scanning (5,000 layers) | 30-60s | Warning zone |
| Processing (5,000 layers) | 60-120s | Warning zone |
| **Total (5,000 layers)** | **<2min** | Acceptable |
| Progress updates | Every 50 layers | Or per page |

---

## Testing Contract

### Unit Tests

```typescript
describe('AuditEngine', () => {
  test('should start in idle state', () => {
    const engine = new AuditEngine();
    expect(engine.getState()).toBe('idle');
  });

  test('should transition through states correctly', async () => {
    const engine = new AuditEngine();
    const states: AuditState[] = [];

    engine.onStateChange((_, newState) => states.push(newState));

    await engine.runAudit();

    expect(states).toEqual([
      'validating',
      'scanning',
      'processing',
      'complete'
    ]);
  });

  test('should reject invalid state transitions', () => {
    const engine = new AuditEngine();
    expect(engine.canTransitionTo('processing')).toBe(false); // Can't skip states
  });

  test('should cancel audit when requested', async () => {
    const engine = new AuditEngine();

    const auditPromise = engine.runAudit();

    setTimeout(() => engine.cancelAudit(), 100);

    await expect(auditPromise).rejects.toThrow('Operation cancelled');
    expect(engine.getState()).toBe('idle');
  });

  test('should emit progress updates', async () => {
    const engine = new AuditEngine();
    const progressUpdates: number[] = [];

    engine.onProgress((progress) => {
      progressUpdates.push(progress.percentage);
    });

    await engine.runAudit();

    expect(progressUpdates.length).toBeGreaterThan(0);
    expect(progressUpdates[progressUpdates.length - 1]).toBe(100);
  });
});
```

### Integration Tests

```typescript
describe('AuditEngine Integration', () => {
  test('should audit real document and return valid results', async () => {
    // Setup: Create test document with known structure
    const page = figma.createPage();
    const text1 = figma.createText();
    const text2 = figma.createText();

    const engine = new AuditEngine();
    const result = await engine.runAudit();

    expect(result.totalTextLayers).toBe(2);
    expect(result.styles.length).toBeGreaterThanOrEqual(0);
  });

  test('should warn on large documents', async () => {
    // Setup: Document with 5,001 text layers
    const warnings: string[] = [];

    const engine = new AuditEngine();
    engine.onProgress((progress) => {
      if (progress.warnings) warnings.push(...progress.warnings);
    });

    await engine.runAudit();

    expect(warnings).toContain(expect.stringMatching(/5,001.*text layers/));
  });
});
```

---

## Version History

- **v1.0** (2025-11-20): Initial Audit Engine API
  - 7-state machine (idle, validating, scanning, processing, complete, error, cancelled)
  - Size validation (5k warning, 25k hard limit)
  - Progress callbacks
  - Cancellation support
