# Quick Start Guide: Font Audit Plugin Development

**Feature Branch**: `001-font-audit` | **Date**: 2025-11-16 | **Plan**: [plan.md](./plan.md)

## Overview

This guide walks you through setting up the development environment for the Font Audit Plugin, from initial project creation through testing and building. Optimized for a single developer using AI coding assistants to generate implementation code.

**Target Audience**: Inexperienced designer leveraging AI-based coding tools (e.g., Claude Code, GitHub Copilot, Cursor)

**Development Philosophy**: Small, testable increments with strong typing and clear separation of concerns

---

## Prerequisites

### Required Software

1. **Node.js 18+** (LTS recommended)
   ```bash
   node --version  # Should be 18.x or higher
   ```
   Install from: https://nodejs.org/

2. **pnpm** (Package manager)
   ```bash
   npm install -g pnpm
   pnpm --version  # Should be 8.x or higher
   ```

3. **Figma Desktop App** (for testing)
   - Download from: https://www.figma.com/downloads/
   - Required for plugin development and testing
   - Web version does NOT support local plugin development

4. **Code Editor** (VS Code recommended)
   - Install TypeScript extension
   - Install ESLint extension
   - Install Prettier extension

### Optional Tools

- **Git** (for version control)
- **Figma account** with edit access to test files

---

## Project Initialization

### Step 1: Scaffold with create-figma-plugin

The project uses the `create-figma-plugin` toolkit for initial scaffolding, then customizes it.

```bash
# Navigate to project root
cd figma-fontscope

# Initialize Figma plugin project
npx create-figma-plugin figma-fontscope

# Answer prompts:
# - Plugin name: Figma Font Audit Pro
# - Plugin ID: (auto-generated)
# - Template: Default (TypeScript + React)
```

**What this creates**:
- Basic dual-context structure (main + UI)
- TypeScript configuration
- Build scripts using Vite
- Figma plugin manifest template

### Step 2: Install Additional Dependencies

```bash
# Navigate into project
cd figma-fontscope

# Install UI dependencies
pnpm add react react-dom
pnpm add -D @types/react @types/react-dom

# Install styling
pnpm add tailwindcss postcss autoprefixer
pnpm add @figma/plugin-ds @figma/plugin-ds-tokens

# Install PDF export (lazy-loaded)
pnpm add jspdf

# Install CSV export
pnpm add papaparse
pnpm add -D @types/papaparse

# Install testing framework
pnpm add -D vitest @vitest/ui
pnpm add -D @testing-library/react @testing-library/jest-dom

# Install development tools
pnpm add -D eslint prettier eslint-config-prettier
pnpm add -D typescript @types/node
```

### Step 3: Configure Tailwind CSS

```bash
# Initialize Tailwind
npx tailwindcss init -p
```

Update `tailwind.config.js`:

```javascript
/** @type {import('tailwindcss').Config} */
export default {
  content: [
    "./src/ui/**/*.{js,ts,jsx,tsx}",
  ],
  theme: {
    extend: {
      colors: {
        // Figma Plugin DS tokens
        'figma-bg': 'var(--figma-color-bg)',
        'figma-text': 'var(--figma-color-text)',
        'figma-border': 'var(--figma-color-border)',
        // Add more tokens as needed
      }
    },
  },
  plugins: [],
}
```

Create `src/ui/styles/globals.css`:

```css
@tailwind base;
@tailwind components;
@tailwind utilities;

/* Import Figma Plugin DS tokens */
@import '@figma/plugin-ds/figma-plugin-ds.css';
```

### Step 4: Configure Vite

Update `vite.config.ts` for dual-context build:

```typescript
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import { viteSingleFile } from 'vite-plugin-singlefile';
import path from 'path';

export default defineConfig({
  plugins: [react(), viteSingleFile()],
  resolve: {
    alias: {
      '@': path.resolve(__dirname, './src'),
    },
  },
  build: {
    rollupOptions: {
      input: {
        main: 'src/main/code.ts',
        ui: 'src/ui/index.html',
      },
      output: {
        entryFileNames: '[name].js',
      },
    },
  },
});
```

### Step 5: Configure TypeScript

Update `tsconfig.json` for strict mode:

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "ESNext",
    "lib": ["ES2020", "DOM"],
    "jsx": "react-jsx",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "moduleResolution": "bundler",
    "resolveJsonModule": true,
    "isolatedModules": true,
    "noEmit": true,
    "baseUrl": ".",
    "paths": {
      "@/*": ["./src/*"]
    },
    "types": ["@figma/plugin-typings", "vite/client"]
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

### Step 6: Configure Vitest

Create `vitest.config.ts`:

```typescript
import { defineConfig } from 'vitest/config';
import react from '@vitejs/plugin-react';
import path from 'path';

export default defineConfig({
  plugins: [react()],
  test: {
    globals: true,
    environment: 'jsdom',
    setupFiles: './tests/setup.ts',
  },
  resolve: {
    alias: {
      '@': path.resolve(__dirname, './src'),
    },
  },
});
```

Create `tests/setup.ts`:

```typescript
import '@testing-library/jest-dom';

// Mock Figma API for tests
global.figma = {
  currentPage: {},
  ui: {
    postMessage: vi.fn(),
  },
  // Add more mocks as needed
} as any;
```

### Step 7: Update package.json Scripts

```json
{
  "scripts": {
    "dev": "vite build --watch",
    "build": "vite build",
    "test": "vitest",
    "test:ui": "vitest --ui",
    "lint": "eslint src --ext .ts,.tsx",
    "format": "prettier --write \"src/**/*.{ts,tsx,css}\""
  }
}
```

---

## Project Structure Setup

Create the directory structure outlined in [plan.md](./plan.md):

```bash
# Create main context directories
mkdir -p src/main/audit
mkdir -p src/main/utils

# Create UI context directories
mkdir -p src/ui/components
mkdir -p src/ui/hooks
mkdir -p src/ui/styles

# Create shared directory
mkdir -p src/shared

# Create export directory
mkdir -p src/export

# Create test directories
mkdir -p tests/audit
mkdir -p tests/utils
```

---

## Development Workflow

### Daily Development Cycle

1. **Start Watch Mode**
   ```bash
   pnpm dev
   ```
   This watches for file changes and rebuilds automatically.

2. **Load Plugin in Figma**
   - Open Figma Desktop
   - Go to Plugins → Development → Import plugin from manifest
   - Select `manifest.json` from project root
   - Plugin appears in Plugins menu

3. **Make Code Changes**
   - Edit files in `src/`
   - Watch mode rebuilds automatically
   - Reload plugin in Figma: Plugins → Development → Reload plugin

4. **View Console Logs**
   - Main context logs: Plugins → Development → Show console
   - UI context logs: Right-click plugin UI → Inspect element → Console

5. **Run Tests**
   ```bash
   pnpm test              # Run all tests
   pnpm test:ui           # Run tests with UI
   pnpm test -- --watch   # Watch mode
   ```

### Incremental Implementation Strategy

**Phase 1: Core Audit Engine (Main Context)**

Start with the audit engine in isolation before building UI:

1. **Text Node Traversal** (`src/main/audit/traversal.ts`)
   - Write `findAllTextNodes()` function
   - Test with mock Figma file (100 text layers)
   - Unit test: verify 100% discovery

2. **Text Property Extraction** (`src/main/audit/text-analyzer.ts`)
   - Write `extractTextProperties()` function
   - Handle `figma.mixed` properties
   - Unit test: verify all properties captured

3. **Component Hierarchy** (`src/main/utils/hierarchy.ts`)
   - Write `buildHierarchyPath()` function
   - Test with nested components
   - Unit test: verify breadcrumb accuracy

4. **Text Style Detection** (`src/main/audit/style-matcher.ts`)
   - Write `detectStyleAssignment()` function
   - Write `calculateSimilarity()` function
   - Unit test: verify 80% threshold logic

**Phase 2: PostMessage Protocol**

5. **Main Context Handler** (`src/main/code.ts`)
   - Implement message listener
   - Wire up audit engine
   - Manual test: verify AUDIT_COMPLETE message

6. **UI Context Handler** (`src/ui/hooks/useMessageHandler.ts`)
   - Implement React hook
   - Test with mock messages
   - Unit test: verify state updates

**Phase 3: React UI**

7. **Summary Dashboard** (`src/ui/components/SummaryDashboard.tsx`)
   - Display aggregate metrics
   - Manual test: verify metrics match audit result

8. **Results List** (`src/ui/components/AuditResults.tsx`)
   - Display text layers with grouping
   - Implement search/filter (<200ms requirement)
   - Manual test: verify performance

9. **Click-to-Navigate** (`src/ui/components/LayerItem.tsx`)
   - Wire up NAVIGATE_TO_LAYER message
   - Manual test: verify focus behavior

**Phase 4: Export Features**

10. **PDF Export** (`src/export/pdf-generator.ts`)
    - Lazy-load jsPDF
    - Generate 7-section report
    - Manual test: verify PDF structure

11. **CSV Export** (`src/export/csv-generator.ts`)
    - Use Papa Parse
    - Manual test: verify CSV format

---

## Testing Strategy

### Unit Tests (Vitest)

Test pure functions in isolation:

```typescript
// tests/audit/style-matcher.test.ts

import { describe, it, expect } from 'vitest';
import { calculateSimilarity } from '@/main/audit/style-matcher';

describe('calculateSimilarity', () => {
  it('should return 1.0 for identical properties', () => {
    const props = {
      fontFamily: 'Inter',
      fontSize: 16,
      fontWeight: 400,
      lineHeight: { value: 24, unit: 'PIXELS' },
      color: { r: 0, g: 0, b: 0, a: 1 }
    };

    const score = calculateSimilarity(props, props);
    expect(score).toBe(1.0);
  });

  it('should return 0.30 for matching font family only', () => {
    const text = { fontFamily: 'Inter', fontSize: 16, /* ... */ };
    const style = { fontFamily: 'Inter', fontSize: 20, /* ... */ };

    const score = calculateSimilarity(text, style);
    expect(score).toBeCloseTo(0.30, 2);
  });
});
```

**What to Unit Test**:
- Similarity calculation (all property combinations)
- Hierarchy path building (various nesting levels)
- Color normalization (hex, rgba conversions)
- Line height normalization (PIXELS, PERCENT, AUTO)
- Property extraction (handling `figma.mixed`)

### Integration Tests (Manual in Figma)

Test with real Figma files:

**Test File 1: Basic Audit (50 layers)**
- Various fonts (Inter, Roboto, Arial)
- Mix of styled and unstyled text
- Plain frames + component instances
- Hidden layers

**Test File 2: Large Audit (1000 layers)**
- Performance test: verify <10s completion
- Progress indicator test: verify updates every 500ms

**Test File 3: Edge Cases**
- Text with mixed fonts (multiple fonts in one text block)
- Detached text styles
- External library styles
- Nested components (5+ levels deep)

**Test File 4: Validation**
- 15,000 text layers (should trigger error)

### Performance Testing

```bash
# Create test file with 1000 text layers
# Run audit with console.time()
console.time('audit-1000-layers');
// ... run audit ...
console.timeEnd('audit-1000-layers');
// Should log: audit-1000-layers: <10000ms
```

---

## Build and Distribution

### Development Build

```bash
pnpm dev        # Watch mode for development
```

Output: `dist/code.js` and `dist/ui.html` (auto-reload in Figma)

### Production Build

```bash
pnpm build      # Optimized production build
```

Output: `dist/code.js` and `dist/ui.html` (minified, tree-shaken)

### Plugin Manifest

Ensure `manifest.json` is configured:

```json
{
  "name": "Figma Font Audit Pro",
  "id": "YOUR_PLUGIN_ID",
  "api": "1.0.0",
  "main": "dist/code.js",
  "ui": "dist/ui.html",
  "editorType": ["figma"],
  "permissions": ["currentuser"]
}
```

### Publishing to Figma Community (Future)

1. Test thoroughly with production build
2. Create plugin listing in Figma
3. Upload plugin files
4. Submit for review

---

## Troubleshooting

### Common Issues

**Issue**: "Cannot find module '@figma/plugin-typings'"
```bash
# Solution: Install typings
pnpm add -D @figma/plugin-typings
```

**Issue**: Plugin UI is blank in Figma
```bash
# Solution: Check browser console for errors
# Right-click plugin UI → Inspect element → Console
# Common cause: Missing globals.css import in App.tsx
```

**Issue**: "figma is not defined" in main context
```bash
# Solution: Ensure main code runs in Figma sandbox context
# Check manifest.json: "main": "dist/code.js"
```

**Issue**: Vite build fails with "Cannot resolve @/*"
```bash
# Solution: Verify tsconfig.json and vite.config.ts have matching paths
```

**Issue**: Tests fail with "figma is not defined"
```bash
# Solution: Ensure tests/setup.ts mocks figma global
# Import setup.ts in vitest.config.ts
```

---

## AI Coding Assistant Tips

### Generating Implementation Code

When using AI coding assistants (Claude Code, GitHub Copilot, Cursor), provide clear context:

**Good Prompt**:
```
Generate the calculateFontFamilySimilarity function in src/main/audit/style-matcher.ts.

Input: Two font family strings (e.g., "Inter", "Roboto")
Output: Number from 0-1 (exact match = 1.0, different = 0.0)
Logic: Case-insensitive exact match only

TypeScript, strict mode, include JSDoc comment.
```

**Less Effective Prompt**:
```
Write a function to compare fonts
```

### Iterative Development

1. **Generate small functions** (5-20 lines each)
2. **Test each function** before moving to next
3. **Refactor** once tests pass
4. **Ask AI to write tests** for generated code

**Example Workflow**:
```
You: Generate extractFontFamily function
AI: [generates code]
You: Generate unit test for extractFontFamily
AI: [generates test]
You: [run test, fix issues]
You: Generate extractFontSize function
...
```

---

## Next Steps

After completing setup:

1. ✅ Verify all dependencies installed: `pnpm install`
2. ✅ Run development build: `pnpm dev`
3. ✅ Load plugin in Figma Desktop
4. ✅ Create test Figma file with 50 text layers
5. ✅ Implement Phase 1 (Core Audit Engine) incrementally
6. ✅ Write unit tests for each function
7. ✅ Test with real Figma files
8. ✅ Proceed to Phase 2 (PostMessage Protocol)

**Remember**: Build incrementally, test frequently, and leverage AI assistants for code generation while maintaining clear architecture.

---

## Resources

- **Figma Plugin API Docs**: https://www.figma.com/plugin-docs/
- **create-figma-plugin**: https://github.com/yuanqing/create-figma-plugin
- **Figma Plugin DS**: https://www.figma.com/community/file/928108847914589057
- **Tailwind CSS**: https://tailwindcss.com/docs
- **Vitest**: https://vitest.dev/
- **React Docs**: https://react.dev/

For questions, refer to:
- [plan.md](./plan.md) - Overall implementation plan
- [research.md](./research.md) - Technical research and patterns
- [data-model.md](./data-model.md) - Data structures and validation
- [contracts/plugin-protocol.md](./contracts/plugin-protocol.md) - PostMessage protocol
