# Replacement Engine API Contract

**Feature**: `002-style-governance`
**Created**: 2025-11-20
**Purpose**: Define the ReplacementEngine interface for style and token bulk replacement operations

## Overview

The Replacement Engine is responsible for:
- Safe bulk replacement of styles and tokens
- Version history checkpoint creation
- Adaptive batch processing (100→25→100 algorithm)
- Error recovery with exponential backoff retry
- Rollback support via version history

---

## ReplacementEngine Interface

```typescript
interface ReplacementEngine {
  // Core Methods
  replaceStyle(options: StyleReplacementOptions): Promise<ReplacementResult>;
  replaceToken(options: TokenReplacementOptions): Promise<ReplacementResult>;

  // State Machine
  getState(): ReplacementState;
  canCancelReplacement(): boolean; // Always false after checkpoint created

  // Progress Callbacks
  onProgress(callback: ReplacementProgressCallback): void;
  onStateChange(callback: ReplacementStateChangeCallback): void;

  // Lifecycle
  dispose(): void;
}
```

---

## Types

### ReplacementState

```typescript
type ReplacementState =
  | 'idle'                 // No replacement running
  | 'validating'           // Checking source/target validity
  | 'creating_checkpoint'  // Creating version history savepoint
  | 'processing'           // Applying changes in batches
  | 'complete'             // Finished successfully
  | 'error';               // Failed with error

// Note: No 'cancelled' state - replacements cannot be cancelled after checkpoint
```

### StyleReplacementOptions

```typescript
interface StyleReplacementOptions {
  sourceStyleId: string;           // Style to replace
  targetStyleId: string;           // Replacement style
  affectedLayerIds: string[];      // Specific layers to update
  preserveOverrides?: boolean;     // Default: true
  skipComponentInstances?: boolean; // Default: false
  progressCallback?: ReplacementProgressCallback;
}
```

### TokenReplacementOptions

```typescript
interface TokenReplacementOptions {
  sourceTokenId: string;           // Token to replace
  targetTokenId: string;           // Replacement token
  affectedLayerIds: string[];      // Specific layers to update
  propertyTypes?: string[];        // e.g., ['fills', 'strokes']
  progressCallback?: ReplacementProgressCallback;
}
```

### ReplacementResult

```typescript
interface ReplacementResult {
  success: boolean;
  layersUpdated: number;           // Successfully updated
  layersFailed: number;            // Failed to update
  failedLayers: FailedLayer[];     // Details of failures
  checkpointTitle: string;         // Version history checkpoint
  duration: number;                // Time taken in ms
  hasWarnings: boolean;            // True if partial failures
}

interface FailedLayer {
  layerId: string;
  layerName: string;
  reason: string;
  retryCount: number;
}
```

### ReplacementProgressCallback

```typescript
type ReplacementProgressCallback = (progress: ReplacementProgress) => void;

interface ReplacementProgress {
  state: ReplacementState;
  percentage: number;              // 0-100
  currentBatch: number;
  totalBatches: number;
  currentBatchSize: number;        // Adaptive size (25-100)
  layersProcessed: number;
  failedLayers: number;
  checkpointTitle?: string;        // Available after checkpoint created
}
```

---

## Method Specifications

### replaceStyle()

Replaces all instances of a source style with a target style.

**Signature**:
```typescript
async replaceStyle(options: StyleReplacementOptions): Promise<ReplacementResult>
```

**Preconditions**:
- Current state must be `idle`
- Source and target styles must exist
- Source ≠ Target
- affectedLayerIds.length > 0

**State Transitions**:
```
idle → validating → creating_checkpoint → processing → complete
          ↓                ↓                  ↓
        error            error              error
```

**Steps**:

1. **Validation** (`validating`):
   - Verify source style exists
   - Verify target style exists
   - Check source ≠ target
   - Validate layer IDs
   - Check user has edit permissions

2. **Checkpoint Creation** (`creating_checkpoint`):
   - Call `figma.saveVersionHistoryAsync(title)`
   - Title format: "Style Replacement - [timestamp]"
   - Store checkpoint title for rollback

3. **Processing** (`processing`):
   - Use BatchProcessor with adaptive sizing
   - Start at 100 layers/batch
   - On error: reduce to 25 layers/batch
   - After 5 successes: increase back to 100
   - Retry failed batches 3x with exponential backoff (1s, 2s, 4s)
   - Collect failed layers for report

4. **Completion** (`complete`):
   - Return ReplacementResult
   - Emit completion message to UI

**Error Handling**:
- **Transient errors** (network, timeout): Retry 3x, reduce batch size
- **Persistent errors** (permissions, invalid ID): Fail immediately, offer rollback
- **Partial failures**: Continue operation, report failed layers at end

**Example**:
```typescript
const engine = new ReplacementEngine();

try {
  const result = await engine.replaceStyle({
    sourceStyleId: 'S:old-heading-style',
    targetStyleId: 'S:new-heading-style',
    affectedLayerIds: ['1:23', '1:45', '1:67'],
    preserveOverrides: true,
  });

  if (result.hasWarnings) {
    console.log(`Updated ${result.layersUpdated}, failed ${result.layersFailed}`);
  }
} catch (error) {
  console.error('Replacement failed, rollback to:', error.checkpointTitle);
}
```

---

### replaceToken()

Replaces all instances of a source token with a target token.

**Signature**:
```typescript
async replaceToken(options: TokenReplacementOptions): Promise<ReplacementResult>
```

**Behavior**: Same as replaceStyle() but uses `setBoundVariable()` API

---

## Batch Processor

Handles adaptive batch sizing and retry logic.

```typescript
interface BatchProcessor {
  processBatches(
    layers: TextNode[],
    operation: (batch: TextNode[]) => Promise<void>
  ): AsyncIterator<BatchResult>;
}

interface BatchResult {
  batchNumber: number;
  batchSize: number;
  layersProcessed: number;
  layersFailed: number;
  errors: Error[];
}

// Adaptive Algorithm:
// 1. Start: 100 layers/batch
// 2. On error: Reduce to 25 layers/batch, retry batch
// 3. After 5 consecutive successes: Increase by 25 (max 100)
```

**Example**:
```typescript
const processor = new BatchProcessor();

for await (const result of processor.processBatches(affectedLayers, async (batch) => {
  for (const layer of batch) {
    layer.textStyleId = targetStyleId;
  }
})) {
  onProgress({
    currentBatch: result.batchNumber,
    layersProcessed: result.layersProcessed,
    ...
  });
}
```

---

## Checkpoint Manager

Handles version history creation.

```typescript
interface CheckpointManager {
  createCheckpoint(operationName: string): Promise<string>;
}

// Implementation
async function createCheckpoint(operationName: string): Promise<string> {
  const timestamp = new Date().toISOString().slice(0, 19).replace('T', ' ');
  const title = `${operationName} - ${timestamp}`;

  await figma.saveVersionHistoryAsync(title);

  return title;
}
```

---

## Error Recovery

### Error Classification

```typescript
type ErrorType =
  | 'transient'    // Network, timeout, rate limit → Retry
  | 'persistent'   // Permission, invalid ID → Fail + rollback
  | 'validation'   // Invalid input → Prevent start
  | 'partial';     // Individual layer failures → Mark + continue

function classifyError(error: Error): ErrorType {
  if (error.message.includes('timeout') || error.message.includes('429')) {
    return 'transient';
  } else if (error.message.includes('permission') || error.message.includes('not found')) {
    return 'persistent';
  } else if (error.message.includes('locked')) {
    return 'partial';
  }
  return 'persistent'; // Default to safest option
}
```

### Retry Strategy

```typescript
interface RetryStrategy {
  maxRetries: number;              // 3
  backoffDelays: number[];         // [1000, 2000, 4000] ms
  shouldRetry(error: Error): boolean;
}

async function retryWithBackoff<T>(
  operation: () => Promise<T>,
  strategy: RetryStrategy
): Promise<T> {
  for (let attempt = 0; attempt <= strategy.maxRetries; attempt++) {
    try {
      return await operation();
    } catch (error) {
      if (attempt === strategy.maxRetries || !strategy.shouldRetry(error)) {
        throw error;
      }
      await sleep(strategy.backoffDelays[attempt]);
    }
  }
  throw new Error('Retry exhausted');
}
```

---

## Performance Targets

| Metric | Target | Notes |
|--------|--------|-------|
| Validation | <1s | Quick checks |
| Checkpoint creation | 1-3s | Depends on document size |
| Processing (1,000 layers) | 10-20s | ~100 layers/second optimal |
| Processing (1,000 layers, degraded) | 40-60s | ~25 layers/second after errors |
| Batch frequency | 1-4s | Depends on batch size |

---

## Testing Contract

```typescript
describe('ReplacementEngine', () => {
  test('should validate source and target before starting', async () => {
    const engine = new ReplacementEngine();

    await expect(
      engine.replaceStyle({
        sourceStyleId: 'invalid',
        targetStyleId: 'valid',
        affectedLayerIds: ['1:1'],
      })
    ).rejects.toThrow('Source style not found');
  });

  test('should create checkpoint before processing', async () => {
    const engine = new ReplacementEngine();
    const checkpoints: string[] = [];

    engine.onStateChange((_, newState) => {
      if (newState === 'creating_checkpoint') {
        checkpoints.push('checkpoint created');
      }
    });

    await engine.replaceStyle(validOptions);

    expect(checkpoints).toHaveLength(1);
  });

  test('should reduce batch size on error', async () => {
    const batchSizes: number[] = [];

    engine.onProgress((progress) => {
      batchSizes.push(progress.currentBatchSize);
    });

    // First batch fails → second batch should be 25 instead of 100
    expect(batchSizes[0]).toBe(100);
    expect(batchSizes[1]).toBe(25); // After error
  });
});
```

---

## Version History

- **v1.0** (2025-11-20): Initial Replacement Engine API
  - Style and token replacement
  - Adaptive batching (100→25→100)
  - Version checkpoints
  - Exponential backoff retry
  - No cancellation (safety constraint)
