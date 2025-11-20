# Message Protocol Contract

**Feature**: `002-style-governance`
**Created**: 2025-11-20
**Purpose**: Define the message passing protocol between UI (iframe) and Main (sandbox) contexts

## Overview

Figma plugins run in dual contexts:
- **Main Context** (sandbox): Full Figma Plugin API access, no DOM/React
- **UI Context** (iframe): React/DOM for UI, limited Figma access

Communication happens via `postMessage` between these contexts. This document defines all message types and their payloads.

---

## Message Format

All messages follow this base structure:

```typescript
interface BaseMessage {
  type: string;        // Message type identifier
  payload?: any;       // Optional message data
  timestamp: number;   // Message creation time (Date.now())
}
```

---

## UI → Main Messages

Messages sent from React UI to the main Figma plugin context.

### Audit Messages

#### RUN_AUDIT

Start a new audit operation.

```typescript
interface RunAuditMessage {
  type: 'RUN_AUDIT';
  payload: {
    options?: {
      includeHiddenLayers?: boolean;  // Default: true
      includeTokens?: boolean;        // Default: true
    };
  };
}
```

**Trigger**: User clicks "Run Audit" button
**Response**: Main sends `AUDIT_STARTED` → `AUDIT_PROGRESS`* → `AUDIT_COMPLETE` or `AUDIT_ERROR`

---

#### CANCEL_AUDIT

Cancel an in-progress audit operation.

```typescript
interface CancelAuditMessage {
  type: 'CANCEL_AUDIT';
  payload: null;
}
```

**Trigger**: User clicks "Cancel" button during scan/process
**Response**: Main sends `AUDIT_CANCELLED`
**Constraint**: Only valid during `scanning` or `processing` states

---

### Replacement Messages

#### REPLACE_STYLE

Replace all instances of a source style with a target style.

```typescript
interface ReplaceStyleMessage {
  type: 'REPLACE_STYLE';
  payload: {
    sourceStyleId: string;           // Style to replace
    targetStyleId: string;           // Replacement style
    affectedLayerIds: string[];      // Layers to update
    options?: {
      preserveOverrides?: boolean;   // Default: true
      skipComponentInstances?: boolean; // Default: false
    };
  };
}
```

**Trigger**: User confirms replacement dialog
**Response**: Main sends `REPLACEMENT_STARTED` → `REPLACEMENT_CHECKPOINT_CREATED` → `REPLACEMENT_PROGRESS`* → `REPLACEMENT_COMPLETE` or `REPLACEMENT_ERROR`

---

#### REPLACE_TOKEN

Replace all instances of a source token with a target token.

```typescript
interface ReplaceTokenMessage {
  type: 'REPLACE_TOKEN';
  payload: {
    sourceTokenId: string;           // Token to replace
    targetTokenId: string;           // Replacement token
    affectedLayerIds: string[];      // Layers to update
    propertyTypes?: string[];        // Which properties to update (e.g., ['fills'])
  };
}
```

**Trigger**: User confirms token replacement dialog
**Response**: Same as REPLACE_STYLE (reuses replacement engine)

---

### Export Messages

#### EXPORT_PDF

Generate and download a PDF report.

```typescript
interface ExportPDFMessage {
  type: 'EXPORT_PDF';
  payload: {
    auditResult: AuditResult;        // Full audit data
    options?: {
      includeCharts?: boolean;       // Default: false (text-based for MVP)
      includeDetailedTables?: boolean; // Default: true
      pageOrientation?: 'portrait' | 'landscape'; // Default: 'portrait'
    };
  };
}
```

**Trigger**: User clicks "Export PDF Report" button
**Response**: Main sends `EXPORT_PDF_STARTED` → `EXPORT_PDF_COMPLETE` with blob URL

---

#### EXPORT_CSV

Generate and download a CSV data export.

```typescript
interface ExportCSVMessage {
  type: 'EXPORT_CSV';
  payload: {
    auditResult: AuditResult;        // Full audit data
    options?: {
      includeHeaders?: boolean;      // Default: true
      delimiter?: string;            // Default: ','
    };
  };
}
```

**Trigger**: User clicks "Export CSV Data" button
**Response**: Main sends `EXPORT_CSV_STARTED` → `EXPORT_CSV_COMPLETE` with blob URL

---

### Navigation Messages

#### NAVIGATE_TO_LAYER

Focus a specific layer in the Figma canvas.

```typescript
interface NavigateToLayerMessage {
  type: 'NAVIGATE_TO_LAYER';
  payload: {
    layerId: string;                 // Target layer ID
    zoom?: boolean;                  // Zoom to fit (default: true)
  };
}
```

**Trigger**: User clicks layer in detail panel
**Response**: Main sends `NAVIGATION_COMPLETE` or `NAVIGATION_ERROR`

---

## Main → UI Messages

Messages sent from the main Figma plugin context to the React UI.

### Audit Status Messages

#### AUDIT_STARTED

Audit operation has begun.

```typescript
interface AuditStartedMessage {
  type: 'AUDIT_STARTED';
  payload: {
    state: 'validating';
    totalPages?: number;             // If known from validation
  };
}
```

---

#### AUDIT_PROGRESS

Progress update during audit.

```typescript
interface AuditProgressMessage {
  type: 'AUDIT_PROGRESS';
  payload: {
    state: 'scanning' | 'processing';
    progress: number;                // 0-100 percentage
    currentStep: string;             // e.g., "Scanning page 5 of 10"
    pagesScanned?: number;           // If in scanning state
    layersProcessed?: number;        // If in processing state
  };
}
```

**Frequency**: Every 50 layers processed (or every page for scanning)

---

#### AUDIT_COMPLETE

Audit finished successfully.

```typescript
interface AuditCompleteMessage {
  type: 'AUDIT_COMPLETE';
  payload: {
    result: AuditResult;             // Complete audit data
    duration: number;                // Time taken in ms
  };
}
```

---

#### AUDIT_ERROR

Audit failed with error.

```typescript
interface AuditErrorMessage {
  type: 'AUDIT_ERROR';
  payload: {
    error: string;                   // Error message
    errorType: 'validation' | 'scanning' | 'processing' | 'unknown';
    canRetry: boolean;               // True if retryable error
    details?: string;                // Stack trace or additional info
  };
}
```

---

#### AUDIT_CANCELLED

Audit was cancelled by user.

```typescript
interface AuditCancelledMessage {
  type: 'AUDIT_CANCELLED';
  payload: {
    partialResults?: Partial<AuditResult>; // If any data collected
  };
}
```

---

#### AUDIT_INVALIDATED

Document changed, audit results now stale.

```typescript
interface AuditInvalidatedMessage {
  type: 'AUDIT_INVALIDATED';
  payload: {
    reason: 'document_modified' | 'style_changed' | 'token_changed';
    changeDetails?: string;          // Description of what changed
  };
}
```

**Trigger**: `documentchange` event detected in main context

---

### Replacement Status Messages

#### REPLACEMENT_STARTED

Replacement operation has begun.

```typescript
interface ReplacementStartedMessage {
  type: 'REPLACEMENT_STARTED';
  payload: {
    operationType: 'style' | 'token';
    state: 'validating';
    sourceId: string;
    targetId: string;
    affectedLayerCount: number;
  };
}
```

---

#### REPLACEMENT_CHECKPOINT_CREATED

Version history checkpoint created successfully.

```typescript
interface ReplacementCheckpointCreatedMessage {
  type: 'REPLACEMENT_CHECKPOINT_CREATED';
  payload: {
    checkpointTitle: string;         // e.g., "Style Replacement - 2025-11-20 14:32:15"
    timestamp: Date;
  };
}
```

---

#### REPLACEMENT_PROGRESS

Progress update during replacement.

```typescript
interface ReplacementProgressMessage {
  type: 'REPLACEMENT_PROGRESS';
  payload: {
    state: 'processing';
    progress: number;                // 0-100 percentage
    currentBatch: number;            // Current batch number
    totalBatches: number;            // Total batches
    currentBatchSize: number;        // Layers in current batch
    layersProcessed: number;         // Total layers updated so far
    failedLayers: number;            // Layers that failed
  };
}
```

**Frequency**: After each batch completes

---

#### REPLACEMENT_COMPLETE

Replacement finished successfully.

```typescript
interface ReplacementCompleteMessage {
  type: 'REPLACEMENT_COMPLETE';
  payload: {
    operationType: 'style' | 'token';
    layersUpdated: number;           // Successful updates
    failedLayers?: FailedLayer[];    // Layers that couldn't be updated
    duration: number;                // Time taken in ms
    hasWarnings: boolean;            // True if partial failures occurred
  };
}

interface FailedLayer {
  layerId: string;
  layerName: string;
  reason: string;
}
```

---

#### REPLACEMENT_ERROR

Replacement failed with error.

```typescript
interface ReplacementErrorMessage {
  type: 'REPLACEMENT_ERROR';
  payload: {
    operationType: 'style' | 'token';
    error: string;                   // Error message
    errorType: 'validation' | 'checkpoint' | 'processing' | 'permission';
    checkpointTitle?: string;        // Checkpoint to rollback to
    canRollback: boolean;            // True if rollback available
    details?: string;
  };
}
```

---

### Export Status Messages

#### EXPORT_PDF_STARTED

PDF generation started.

```typescript
interface ExportPDFStartedMessage {
  type: 'EXPORT_PDF_STARTED';
  payload: null;
}
```

---

#### EXPORT_PDF_COMPLETE

PDF generation finished.

```typescript
interface ExportPDFCompleteMessage {
  type: 'EXPORT_PDF_COMPLETE';
  payload: {
    blobUrl: string;                 // Blob URL for download
    filename: string;                // Suggested filename
    fileSize: number;                // Size in bytes
  };
}
```

---

#### EXPORT_CSV_STARTED

CSV generation started.

```typescript
interface ExportCSVStartedMessage {
  type: 'EXPORT_CSV_STARTED';
  payload: null;
}
```

---

#### EXPORT_CSV_COMPLETE

CSV generation finished.

```typescript
interface ExportCSVCompleteMessage {
  type: 'EXPORT_CSV_COMPLETE';
  payload: {
    blobUrl: string;                 // Blob URL for download
    filename: string;                // Suggested filename
    fileSize: number;                // Size in bytes
    rowCount: number;                // Number of data rows
  };
}
```

---

#### EXPORT_ERROR

Export operation failed.

```typescript
interface ExportErrorMessage {
  type: 'EXPORT_ERROR';
  payload: {
    exportType: 'pdf' | 'csv';
    error: string;                   // Error message
    details?: string;
  };
}
```

---

### Navigation Status Messages

#### NAVIGATION_COMPLETE

Layer navigation succeeded.

```typescript
interface NavigationCompleteMessage {
  type: 'NAVIGATION_COMPLETE';
  payload: {
    layerId: string;
  };
}
```

---

#### NAVIGATION_ERROR

Layer navigation failed.

```typescript
interface NavigationErrorMessage {
  type: 'NAVIGATION_ERROR';
  payload: {
    layerId: string;
    error: string;                   // e.g., "Layer not found" or "Layer deleted"
  };
}
```

---

## Message Flow Examples

### Complete Audit Flow

```
UI:   RUN_AUDIT →
Main:               ← AUDIT_STARTED (state: validating)
Main:               ← AUDIT_PROGRESS (state: scanning, progress: 20%)
Main:               ← AUDIT_PROGRESS (state: scanning, progress: 50%)
Main:               ← AUDIT_PROGRESS (state: scanning, progress: 100%)
Main:               ← AUDIT_PROGRESS (state: processing, progress: 30%)
Main:               ← AUDIT_PROGRESS (state: processing, progress: 60%)
Main:               ← AUDIT_PROGRESS (state: processing, progress: 100%)
Main:               ← AUDIT_COMPLETE (result: {...}, duration: 2834)
```

### Replacement with Partial Failure

```
UI:   REPLACE_STYLE →
Main:               ← REPLACEMENT_STARTED (state: validating)
Main:               ← REPLACEMENT_CHECKPOINT_CREATED (title: "Style Replacement...")
Main:               ← REPLACEMENT_PROGRESS (batch 1/4, 100 layers)
Main:               ← REPLACEMENT_PROGRESS (batch 2/4, 100 layers)
Main:               ← REPLACEMENT_PROGRESS (batch 3/4, 25 layers) [batch size reduced after error]
Main:               ← REPLACEMENT_PROGRESS (batch 4/4, 25 layers)
Main:               ← REPLACEMENT_COMPLETE (updated: 245, failed: 5, hasWarnings: true)
```

### Document Invalidation

```
[User runs audit successfully]
Main:               ← AUDIT_COMPLETE (...)
[User adds text layer in Figma]
Main:               ← AUDIT_INVALIDATED (reason: document_modified)
[UI shows warning banner]
```

---

## Implementation Notes

### Message Handler Pattern (UI)

```typescript
// src/ui/hooks/useMessageHandler.ts
useEffect(() => {
  window.onmessage = (event) => {
    const message = event.data.pluginMessage;

    switch (message.type) {
      case 'AUDIT_STARTED':
        setAuditState('validating');
        break;
      case 'AUDIT_PROGRESS':
        setProgress(message.payload.progress);
        setAuditState(message.payload.state);
        break;
      case 'AUDIT_COMPLETE':
        setAuditResult(message.payload.result);
        setAuditState('complete');
        break;
      // ... other handlers
    }
  };
}, []);
```

### Message Sender Pattern (Main)

```typescript
// src/main/code.ts
figma.ui.postMessage({
  type: 'AUDIT_PROGRESS',
  payload: {
    state: 'scanning',
    progress: Math.round((pagesScanned / totalPages) * 100),
    currentStep: `Scanning page ${pagesScanned} of ${totalPages}`,
    pagesScanned,
  },
  timestamp: Date.now(),
});
```

---

## Version History

- **v1.0** (2025-11-20): Initial message protocol
  - Audit messages (run, cancel, progress, complete, error, invalidated)
  - Replacement messages (style, token, checkpoint, progress, complete, error)
  - Export messages (PDF, CSV, start, complete, error)
  - Navigation messages (navigate, complete, error)
