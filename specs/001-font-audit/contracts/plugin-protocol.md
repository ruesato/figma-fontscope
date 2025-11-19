# Plugin Communication Protocol

**Feature Branch**: `001-font-audit` | **Date**: 2025-11-16 | **Plan**: [../plan.md](../plan.md)

## Overview

This document specifies the postMessage-based communication protocol between the Figma plugin's main context (sandbox) and UI context (iframe). The protocol enables audit orchestration, progress reporting, navigation, and error handling across the dual-context architecture.

## Architecture Context

```
┌─────────────────────────────────────────────────────────────┐
│  Figma Plugin                                               │
│                                                             │
│  ┌──────────────────────┐         ┌────────────────────┐   │
│  │  Main Context        │         │  UI Context        │   │
│  │  (Sandbox)           │◄───────►│  (Iframe)          │   │
│  │                      │ postMsg │                    │   │
│  │  - Figma API access  │         │  - React UI        │   │
│  │  - Audit engine      │         │  - User input      │   │
│  │  - Node traversal    │         │  - Results display │   │
│  │  - Navigation        │         │  - Export logic    │   │
│  └──────────────────────┘         └────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

**Communication Method**: `figma.ui.postMessage()` (main → UI) and `parent.postMessage()` (UI → main)

**Message Format**: All messages are TypeScript objects with a `type` discriminator field.

---

## Message Type Definitions

### UI → Main Messages

Messages sent from the UI context to the main context (user actions).

```typescript
type UIToMainMessage =
  | RunAuditMessage
  | NavigateToLayerMessage
  | CancelAuditMessage;

interface RunAuditMessage {
  type: 'RUN_AUDIT';
  scope: 'page' | 'selection';
}

interface NavigateToLayerMessage {
  type: 'NAVIGATE_TO_LAYER';
  layerId: string;
}

interface CancelAuditMessage {
  type: 'CANCEL_AUDIT';
}
```

**Handler Location**: `src/main/code.ts` (main entry point)

---

### Main → UI Messages

Messages sent from the main context to the UI context (audit status updates and results).

```typescript
type MainToUIMessage =
  | AuditStartedMessage
  | AuditProgressMessage
  | AuditCompleteMessage
  | AuditErrorMessage
  | NavigateSuccessMessage
  | NavigateErrorMessage;

interface AuditStartedMessage {
  type: 'AUDIT_STARTED';
}

interface AuditProgressMessage {
  type: 'AUDIT_PROGRESS';
  progress: number;        // 0-100 percentage
  current: number;         // Current node index
  total: number;           // Total nodes to process
}

interface AuditCompleteMessage {
  type: 'AUDIT_COMPLETE';
  result: AuditResult;     // See data-model.md
}

interface AuditErrorMessage {
  type: 'AUDIT_ERROR';
  error: string;           // User-friendly error message
  errorType: 'VALIDATION' | 'API' | 'UNKNOWN';
}

interface NavigateSuccessMessage {
  type: 'NAVIGATE_SUCCESS';
}

interface NavigateErrorMessage {
  type: 'NAVIGATE_ERROR';
  error: string;           // User-friendly error message
}
```

**Handler Location**: `src/ui/App.tsx` or `src/ui/hooks/useMessageHandler.ts`

---

## Message Flow Diagrams

### Happy Path: Page Audit

```
User                UI Context              Main Context            Figma API
 │                       │                       │                      │
 │ Click "Run Audit"     │                       │                      │
 ├──────────────────────►│                       │                      │
 │                       │ RUN_AUDIT (page)      │                      │
 │                       ├──────────────────────►│                      │
 │                       │                       │ Validate scope       │
 │                       │                       │                      │
 │                       │ AUDIT_STARTED         │                      │
 │                       │◄──────────────────────┤                      │
 │ Show spinner          │                       │ findAllWithCriteria  │
 │◄──────────────────────┤                       ├─────────────────────►│
 │                       │                       │ TextNode[]           │
 │                       │                       │◄─────────────────────┤
 │                       │                       │                      │
 │                       │ AUDIT_PROGRESS (33%)  │ Process nodes        │
 │                       │◄──────────────────────┤ 0-100                │
 │ Update progress bar   │                       │                      │
 │◄──────────────────────┤                       │                      │
 │                       │ AUDIT_PROGRESS (66%)  │ Process nodes        │
 │                       │◄──────────────────────┤ 100-200              │
 │                       │                       │                      │
 │                       │ AUDIT_COMPLETE        │ Build AuditResult    │
 │                       │◄──────────────────────┤                      │
 │ Display results       │                       │                      │
 │◄──────────────────────┤                       │                      │
 │                       │                       │                      │
```

**Timing Requirements**:
- `AUDIT_STARTED` sent immediately (<10ms after RUN_AUDIT received)
- `AUDIT_PROGRESS` sent every 100 nodes processed OR every 500ms (whichever comes first)
- `AUDIT_COMPLETE` sent once all processing complete (target: <10s for 1000 layers)

---

### Happy Path: Click-to-Navigate

```
User                UI Context              Main Context            Figma API
 │                       │                       │                      │
 │ Click on layer item   │                       │                      │
 ├──────────────────────►│                       │                      │
 │                       │ NAVIGATE_TO_LAYER     │                      │
 │                       ├──────────────────────►│                      │
 │                       │                       │ Get node by ID       │
 │                       │                       ├─────────────────────►│
 │                       │                       │ TextNode             │
 │                       │                       │◄─────────────────────┤
 │                       │                       │ Set selection        │
 │                       │                       ├─────────────────────►│
 │                       │                       │ Zoom into view       │
 │                       │                       ├─────────────────────►│
 │                       │                       │                      │
 │                       │ NAVIGATE_SUCCESS      │                      │
 │                       │◄──────────────────────┤                      │
 │ Visual feedback (✓)   │                       │                      │
 │◄──────────────────────┤                       │                      │
 │                       │                       │                      │
```

**Timing Requirements**:
- Navigation must complete in <200ms (constitution Principle IV)
- Visual feedback shown immediately on NAVIGATE_SUCCESS

---

### Error Path: Validation Failure

```
User                UI Context              Main Context
 │                       │                       │
 │ Click "Run Audit"     │                       │
 ├──────────────────────►│                       │
 │                       │ RUN_AUDIT (selection) │
 │                       ├──────────────────────►│
 │                       │                       │ Count nodes
 │                       │                       │ 12,345 > 10,000!
 │                       │                       │
 │                       │ AUDIT_ERROR           │
 │                       │ (VALIDATION)          │
 │                       │◄──────────────────────┤
 │ Show error dialog     │                       │
 │◄──────────────────────┤                       │
 │ "File too large..."   │                       │
 │                       │                       │
```

**Error Message Format**:

```typescript
{
  type: 'AUDIT_ERROR',
  error: 'File too large (12,345 layers detected). Maximum supported: 10,000 layers. Please audit a smaller selection or split into multiple files.',
  errorType: 'VALIDATION'
}
```

---

### Error Path: API Failure with Retry

```
User                UI Context              Main Context            Figma API
 │                       │                       │                      │
 │                       │ (Audit in progress)   │ loadFontAsync()      │
 │                       │                       ├─────────────────────►│
 │                       │                       │ ERROR                │
 │                       │                       │◄─────────────────────┤
 │                       │                       │ Retry 1 (wait 1s)    │
 │                       │                       ├─────────────────────►│
 │                       │                       │ ERROR                │
 │                       │                       │◄─────────────────────┤
 │                       │                       │ Retry 2 (wait 2s)    │
 │                       │                       ├─────────────────────►│
 │                       │                       │ ERROR                │
 │                       │                       │◄─────────────────────┤
 │                       │                       │ Retry 3 (wait 4s)    │
 │                       │                       ├─────────────────────►│
 │                       │                       │ ERROR                │
 │                       │                       │◄─────────────────────┤
 │                       │ AUDIT_ERROR           │ Mark node failed     │
 │                       │ (API)                 │ Continue audit       │
 │                       │◄──────────────────────┤                      │
 │ Show warning toast    │                       │                      │
 │◄──────────────────────┤                       │                      │
 │                       │                       │                      │
 │                       │ AUDIT_COMPLETE        │                      │
 │                       │ (partial results)     │                      │
 │                       │◄──────────────────────┤                      │
 │                       │                       │                      │
```

**Retry Logic** (per FR-014):
- 3 retry attempts with exponential backoff: 1s, 2s, 4s
- After all retries fail: mark node as "Unable to read layer properties"
- Continue audit with remaining nodes
- Include failed node count in AuditSummary

---

### Cancel Flow

```
User                UI Context              Main Context
 │                       │                       │
 │                       │ (Audit in progress)   │
 │                       │                       │ Processing nodes...
 │ Click "Cancel"        │                       │
 ├──────────────────────►│                       │
 │                       │ CANCEL_AUDIT          │
 │                       ├──────────────────────►│
 │                       │                       │ Set cancelFlag = true
 │                       │                       │ Stop processing
 │                       │                       │
 │                       │ AUDIT_ERROR           │
 │                       │ (UNKNOWN)             │
 │                       │◄──────────────────────┤
 │ Hide spinner          │                       │
 │◄──────────────────────┤                       │
 │                       │                       │
```

**Cancel Behavior**:
- Set `cancelFlag = true` in main context
- Check `cancelFlag` after each node processed
- Send `AUDIT_ERROR` with message "Audit cancelled by user"
- Clean up partial results (do not send AUDIT_COMPLETE)

---

## Implementation Details

### Main Context (Sandbox) Handler

```typescript
// src/main/code.ts

figma.ui.onmessage = async (msg: UIToMainMessage) => {
  switch (msg.type) {
    case 'RUN_AUDIT':
      await handleRunAudit(msg.scope);
      break;

    case 'NAVIGATE_TO_LAYER':
      await handleNavigateToLayer(msg.layerId);
      break;

    case 'CANCEL_AUDIT':
      handleCancelAudit();
      break;

    default:
      console.warn('Unknown message type:', (msg as any).type);
  }
};

async function handleRunAudit(scope: 'page' | 'selection') {
  try {
    // 1. Validation
    const nodes = scope === 'page'
      ? figma.currentPage.findAllWithCriteria({ types: ['TEXT'] })
      : figma.currentPage.selection.filter(n => n.type === 'TEXT');

    if (nodes.length > 10000) {
      figma.ui.postMessage({
        type: 'AUDIT_ERROR',
        error: `File too large (${nodes.length} layers detected). Maximum supported: 10,000 layers.`,
        errorType: 'VALIDATION'
      });
      return;
    }

    // 2. Start audit
    figma.ui.postMessage({ type: 'AUDIT_STARTED' });

    // 3. Process nodes with progress updates
    const textLayers: TextLayerData[] = [];
    for (let i = 0; i < nodes.length; i++) {
      if (cancelFlag) {
        figma.ui.postMessage({
          type: 'AUDIT_ERROR',
          error: 'Audit cancelled by user',
          errorType: 'UNKNOWN'
        });
        return;
      }

      const layerData = await processTextNode(nodes[i]);
      textLayers.push(layerData);

      // Progress update every 100 nodes or every 500ms
      if (i % 100 === 0 || Date.now() - lastProgressUpdate > 500) {
        figma.ui.postMessage({
          type: 'AUDIT_PROGRESS',
          progress: Math.round((i / nodes.length) * 100),
          current: i,
          total: nodes.length
        });
        lastProgressUpdate = Date.now();
      }
    }

    // 4. Complete audit
    const result = buildAuditResult(textLayers);
    figma.ui.postMessage({
      type: 'AUDIT_COMPLETE',
      result
    });

  } catch (error) {
    figma.ui.postMessage({
      type: 'AUDIT_ERROR',
      error: error.message || 'Unknown error occurred',
      errorType: 'UNKNOWN'
    });
  }
}

async function handleNavigateToLayer(layerId: string) {
  try {
    const node = await figma.getNodeByIdAsync(layerId);
    if (!node || node.type !== 'TEXT') {
      throw new Error('Layer not found or not a text layer');
    }

    // Focus layer
    figma.currentPage.selection = [node];
    figma.viewport.scrollAndZoomIntoView([node]);

    figma.ui.postMessage({ type: 'NAVIGATE_SUCCESS' });

  } catch (error) {
    figma.ui.postMessage({
      type: 'NAVIGATE_ERROR',
      error: error.message || 'Navigation failed'
    });
  }
}
```

---

### UI Context (React) Handler

```typescript
// src/ui/hooks/useMessageHandler.ts

import { useEffect, useState } from 'react';
import type { MainToUIMessage, AuditResult } from '@/shared/types';

export function useMessageHandler() {
  const [auditResult, setAuditResult] = useState<AuditResult | null>(null);
  const [isAuditing, setIsAuditing] = useState(false);
  const [progress, setProgress] = useState(0);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    const handleMessage = (event: MessageEvent<MainToUIMessage>) => {
      const msg = event.data;

      switch (msg.type) {
        case 'AUDIT_STARTED':
          setIsAuditing(true);
          setProgress(0);
          setError(null);
          break;

        case 'AUDIT_PROGRESS':
          setProgress(msg.progress);
          break;

        case 'AUDIT_COMPLETE':
          setAuditResult(msg.result);
          setIsAuditing(false);
          setProgress(100);
          break;

        case 'AUDIT_ERROR':
          setError(msg.error);
          setIsAuditing(false);
          console.error(`Audit error (${msg.errorType}):`, msg.error);
          break;

        case 'NAVIGATE_SUCCESS':
          // Optional: Show success toast
          break;

        case 'NAVIGATE_ERROR':
          setError(msg.error);
          break;

        default:
          console.warn('Unknown message type:', (msg as any).type);
      }
    };

    window.addEventListener('message', handleMessage);
    return () => window.removeEventListener('message', handleMessage);
  }, []);

  // Helper functions to send messages to main context
  const runAudit = (scope: 'page' | 'selection') => {
    parent.postMessage({ pluginMessage: { type: 'RUN_AUDIT', scope } }, '*');
  };

  const navigateToLayer = (layerId: string) => {
    parent.postMessage({ pluginMessage: { type: 'NAVIGATE_TO_LAYER', layerId } }, '*');
  };

  const cancelAudit = () => {
    parent.postMessage({ pluginMessage: { type: 'CANCEL_AUDIT' } }, '*');
  };

  return {
    auditResult,
    isAuditing,
    progress,
    error,
    runAudit,
    navigateToLayer,
    cancelAudit
  };
}
```

---

## Error Handling Strategy

### Error Types

1. **VALIDATION** - Pre-audit validation failure
   - Example: File exceeds 10,000 layer limit
   - Action: Display error dialog, do not start audit

2. **API** - Figma API failure after retries
   - Example: Font loading failure, node access error
   - Action: Mark specific node as failed, continue audit, show warning

3. **UNKNOWN** - Unexpected error or user cancellation
   - Example: Network error, corrupted data, user cancelled
   - Action: Display error message, stop audit

### User-Facing Error Messages

```typescript
// Good error messages (actionable, clear)
const errorMessages = {
  FILE_TOO_LARGE: 'File too large ({count} layers detected). Maximum supported: 10,000 layers. Please audit a smaller selection or split into multiple files.',

  NODE_READ_FAILED: 'Unable to read properties for {count} text layers. These layers are marked as "Unable to read" in results.',

  NAVIGATION_FAILED: 'Could not navigate to layer. It may have been deleted or is in a different page.',

  USER_CANCELLED: 'Audit cancelled by user.',

  UNKNOWN_ERROR: 'An unexpected error occurred. Please try again or contact support.'
};
```

---

## Testing Protocol

### Unit Tests (Vitest)

Test message handlers in isolation:

```typescript
// tests/main/message-handler.test.ts

describe('Main message handler', () => {
  it('should handle RUN_AUDIT message', async () => {
    const msg = { type: 'RUN_AUDIT', scope: 'page' };
    await handleRunAudit(msg.scope);

    expect(figma.ui.postMessage).toHaveBeenCalledWith({ type: 'AUDIT_STARTED' });
  });

  it('should validate layer count before audit', async () => {
    // Mock 15,000 text layers
    vi.spyOn(figma.currentPage, 'findAllWithCriteria').mockReturnValue(
      Array(15000).fill({ type: 'TEXT' })
    );

    const msg = { type: 'RUN_AUDIT', scope: 'page' };
    await handleRunAudit(msg.scope);

    expect(figma.ui.postMessage).toHaveBeenCalledWith(
      expect.objectContaining({
        type: 'AUDIT_ERROR',
        errorType: 'VALIDATION'
      })
    );
  });
});
```

### Integration Tests (Manual)

Test message flow in actual Figma plugin:

1. **Happy path audit**:
   - Run audit on page with 100 text layers
   - Verify AUDIT_STARTED → AUDIT_PROGRESS → AUDIT_COMPLETE sequence
   - Verify progress updates display correctly

2. **Navigation**:
   - Click layer in results
   - Verify Figma focuses correct layer
   - Verify NAVIGATE_SUCCESS received

3. **Error handling**:
   - Run audit on file with >10,000 layers
   - Verify AUDIT_ERROR displayed with user-friendly message

4. **Cancellation**:
   - Start audit, click cancel immediately
   - Verify audit stops and UI resets

---

## Performance Considerations

### Progress Update Throttling

Send progress updates at most every 500ms OR every 100 nodes (whichever comes first) to avoid UI thread congestion.

```typescript
let lastProgressUpdate = Date.now();

for (let i = 0; i < nodes.length; i++) {
  // ... process node ...

  if (i % 100 === 0 || Date.now() - lastProgressUpdate > 500) {
    figma.ui.postMessage({
      type: 'AUDIT_PROGRESS',
      progress: Math.round((i / nodes.length) * 100),
      current: i,
      total: nodes.length
    });
    lastProgressUpdate = Date.now();
  }
}
```

### Message Size Optimization

`AUDIT_COMPLETE` message contains full `AuditResult` which may be large (10,000 layers × 500 bytes ≈ 5MB). This is acceptable for in-memory processing but monitor performance:

- If postMessage latency >1s, consider chunking results
- If UI freezes on large results, implement incremental loading (batches of 500)

---

## Protocol Versioning

**Current Version**: 1.0.0 (initial release)

**Future Compatibility**: If protocol changes in v1.1+, include version field:

```typescript
interface VersionedMessage {
  protocolVersion: string;  // "1.0.0"
  type: string;
  // ... other fields
}
```

This allows UI to detect incompatible main context versions and display upgrade prompts.

---

## Summary

- **Unidirectional data flow**: UI sends commands, main sends results
- **Progress transparency**: Regular updates every 500ms or 100 nodes
- **Error resilience**: Retry logic, graceful degradation, clear error messages
- **Performance**: Throttled updates, <200ms navigation, <10s audits
- **Testability**: Clear handler functions, mockable message protocol
