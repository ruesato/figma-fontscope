# Export Engine API Contract

**Feature**: `002-style-governance`
**Created**: 2025-11-20
**Purpose**: Define the ExportEngine interface for PDF and CSV report generation

## Overview

The Export Engine is responsible for:
- Generating PDF executive reports with tables and metrics
- Generating CSV data exports for analysis
- Formatting audit data for stakeholder consumption
- Creating downloadable blob URLs

---

## ExportEngine Interface

```typescript
interface ExportEngine {
  // Core Methods
  generatePDF(auditResult: AuditResult, options?: PDFOptions): Promise<ExportResult>;
  generateCSV(auditResult: AuditResult, options?: CSVOptions): Promise<ExportResult>;

  // Lifecycle
  dispose(): void;
}
```

---

## Types

### ExportResult

```typescript
interface ExportResult {
  success: boolean;
  blobUrl: string;                 // Blob URL for download
  filename: string;                // Suggested filename
  fileSize: number;                // Size in bytes
  mimeType: string;                // 'application/pdf' or 'text/csv'
  metadata: ExportMetadata;
}

interface ExportMetadata {
  documentName: string;
  auditTimestamp: Date;
  totalTextLayers: number;
  totalStyles: number;
  totalTokens: number;
  exportTimestamp: Date;
  pluginVersion: string;
}
```

### PDFOptions

```typescript
interface PDFOptions {
  // Content
  includeCharts?: boolean;         // Default: false (text-based for MVP)
  includeDetailedTables?: boolean; // Default: true
  includeTokenData?: boolean;      // Default: true

  // Format
  pageOrientation?: 'portrait' | 'landscape'; // Default: 'portrait'
  pageSize?: 'a4' | 'letter';      // Default: 'a4'
  fontSize?: number;               // Default: 10

  // Branding
  title?: string;                  // Default: "Style Audit Report"
  subtitle?: string;               // Default: document name
}
```

### CSVOptions

```typescript
interface CSVOptions {
  // Format
  includeHeaders?: boolean;        // Default: true
  delimiter?: string;              // Default: ','
  quoteChar?: string;              // Default: '"'

  // Content
  includeTokenData?: boolean;      // Default: true
  includeComponentContext?: boolean; // Default: true
}
```

---

## PDF Report Structure

### Section 1: Executive Summary

**Content**:
- Document metadata (name, timestamp, page count)
- Key metrics (total layers, style adoption rate, token coverage)
- Top findings (most used styles, deprecated styles)

**Format**:
```
========================================
  Style Audit Report
  [Document Name]
========================================

Audit Date: 2025-11-20 14:32:15
Total Pages: 42
Total Text Layers: 1,247

========================================
  Executive Summary
========================================

Style Adoption Rate: 87.3%
  - Fully Styled: 1,089 layers (87.3%)
  - Partially Styled: 132 layers (10.6%)
  - Unstyled: 26 layers (2.1%)

Token Coverage: 45.2%
  - Layers with Tokens: 564 (45.2%)
  - Layers without Tokens: 683 (54.8%)

Library Distribution:
  - Local: 234 layers (18.8%)
  - Design System v2: 855 layers (68.5%)
  - Legacy Library: 158 layers (12.7%)
```

### Section 2: Style Inventory

**Content**:
- Table of all styles grouped by library
- Usage counts, page distribution

**Format** (jsPDF-AutoTable):
```
Library: Design System v2
────────────────────────────────────────────────────────────
Style Name                     | Usage | Pages
────────────────────────────────────────────────────────────
Typography/Heading/H1          |   87  | 12, 15, 18, ...
Typography/Heading/H2          |  156  | 5, 8, 12, ...
Typography/Body/Regular        |  412  | All pages
────────────────────────────────────────────────────────────
```

### Section 3: Adoption Visualizations

**Content** (MVP: Text-based, v1.1: Charts):
```
Style Adoption Rate:
█████████████████░░░ 87.3%

Token Coverage:
█████████░░░░░░░░░░░ 45.2%

Library Distribution:
Design System v2   ████████████████████████████ 68.5%
Local              ██████                        18.8%
Legacy Library     ████                          12.7%
```

### Section 4: Metadata

**Content**:
- Plugin version
- Export timestamp
- Row counts
- Data completeness indicators

---

## CSV Export Schema

### Row Format

One row per text layer with all metadata.

**Columns**:
```csv
Layer ID, Layer Name, Text Content, Page Name, Style Name, Style Source, Token Name, Token Value, Component Context, Assignment Status, Visible, Opacity, Has Overrides
```

**Example**:
```csv
"1:234","Heading","Design System Guide","Cover","Typography/Heading/H1","Design System v2","","","FRAME","fully-styled",true,1.0,false
"1:456","Body Text","This is the introduction","Page 1","Typography/Body/Regular","Local","text.primary","#0066CC","INSTANCE","partially-styled",true,1.0,true
"1:789","Caption","Note: See details","Page 2","","","","","FRAME","unstyled",true,0.8,false
```

### Header Rows

Include document metadata before data rows:

```csv
"Document Name:","Design System Guide"
"Audit Date:","2025-11-20 14:32:15"
"Total Layers:","1,247"
"Total Styles:","47"
"Total Tokens:","12"
"Export Date:","2025-11-20 15:45:30"
""
Layer ID, Layer Name, ...
```

---

## Method Specifications

### generatePDF()

Generates a PDF report from audit results.

**Signature**:
```typescript
async generatePDF(auditResult: AuditResult, options?: PDFOptions): Promise<ExportResult>
```

**Steps**:

1. **Initialize jsPDF**:
```typescript
import jsPDF from 'jspdf';
import autoTable from 'jspdf-autotable';

const doc = new jsPDF({
  orientation: options.pageOrientation || 'portrait',
  unit: 'mm',
  format: options.pageSize || 'a4',
});
```

2. **Generate Executive Summary**:
   - Add title and metadata
   - Add key metrics with text formatting
   - Add text-based visualizations

3. **Generate Style Inventory Tables**:
   - Group styles by library
   - Use `autoTable()` for formatted tables
   - Include usage counts and page lists

4. **Generate Token Tables** (if includeTokenData):
   - Table of all tokens with usage
   - Group by collection

5. **Add Metadata Footer**:
   - Plugin version
   - Export timestamp
   - Page numbers

6. **Create Blob URL**:
```typescript
const pdfBlob = doc.output('blob');
const blobUrl = URL.createObjectURL(pdfBlob);
```

**Performance**:
- Target: <5 seconds for 25k layer audit
- Target file size: <5MB

**Example**:
```typescript
const engine = new ExportEngine();

const result = await engine.generatePDF(auditResult, {
  includeCharts: false,
  includeDetailedTables: true,
  title: 'Q4 2025 Style Audit',
});

// Trigger download
const link = document.createElement('a');
link.href = result.blobUrl;
link.download = result.filename;
link.click();
```

---

### generateCSV()

Generates a CSV data export from audit results.

**Signature**:
```typescript
async generateCSV(auditResult: AuditResult, options?: CSVOptions): Promise<ExportResult>
```

**Steps**:

1. **Prepare Header Rows**:
```typescript
const headerRows = [
  ['Document Name:', auditResult.documentName],
  ['Audit Date:', auditResult.timestamp.toISOString()],
  ['Total Layers:', auditResult.totalTextLayers.toString()],
  [''],
];
```

2. **Prepare Data Rows**:
```typescript
const dataRows = auditResult.layers.map(layer => [
  layer.id,
  layer.name,
  layer.textContent.slice(0, 50), // Truncate long text
  layer.pageName,
  layer.styleName || '',
  layer.styleSource || '',
  layer.tokens.map(t => t.tokenName).join('; '),
  layer.tokens.map(t => t.tokenValue).join('; '),
  layer.parentType,
  layer.assignmentStatus,
  layer.visible.toString(),
  layer.opacity.toString(),
  layer.hasOverrides.toString(),
]);
```

3. **Generate CSV** (using PapaParse):
```typescript
import Papa from 'papaparse';

const csv = Papa.unparse({
  fields: [
    'Layer ID', 'Layer Name', 'Text Content', 'Page Name',
    'Style Name', 'Style Source', 'Token Names', 'Token Values',
    'Component Context', 'Assignment Status', 'Visible', 'Opacity', 'Has Overrides'
  ],
  data: [...headerRows, ...dataRows],
}, {
  delimiter: options.delimiter || ',',
  quotes: true,
  header: options.includeHeaders !== false,
});
```

4. **Create Blob URL**:
```typescript
const csvBlob = new Blob([csv], { type: 'text/csv;charset=utf-8;' });
const blobUrl = URL.createObjectURL(csvBlob);
```

**Performance**:
- Target: <3 seconds for 25k layers
- Target file size: <10MB (text layers with metadata)

**Example**:
```typescript
const result = await engine.generateCSV(auditResult, {
  delimiter: ',',
  includeTokenData: true,
});

// Download CSV
const link = document.createElement('a');
link.href = result.blobUrl;
link.download = result.filename; // e.g., "style-audit-2025-11-20.csv"
link.click();
```

---

## Filename Generation

```typescript
function generateFilename(type: 'pdf' | 'csv', documentName: string): string {
  const date = new Date().toISOString().slice(0, 10); // YYYY-MM-DD
  const sanitizedName = documentName.replace(/[^a-z0-9]/gi, '-').toLowerCase();

  return `${sanitizedName}-style-audit-${date}.${type}`;
}

// Examples:
// "design-system-guide-style-audit-2025-11-20.pdf"
// "product-screens-style-audit-2025-11-20.csv"
```

---

## Testing Contract

```typescript
describe('ExportEngine', () => {
  test('should generate valid PDF', async () => {
    const engine = new ExportEngine();
    const result = await engine.generatePDF(mockAuditResult);

    expect(result.success).toBe(true);
    expect(result.mimeType).toBe('application/pdf');
    expect(result.fileSize).toBeGreaterThan(0);
    expect(result.filename).toMatch(/\.pdf$/);
  });

  test('should generate valid CSV', async () => {
    const engine = new ExportEngine();
    const result = await engine.generateCSV(mockAuditResult);

    expect(result.success).toBe(true);
    expect(result.mimeType).toBe('text/csv');
    expect(result.filename).toMatch(/\.csv$/);
  });

  test('PDF should include all required sections', async () => {
    const result = await engine.generatePDF(mockAuditResult);

    // Verify content (would need to parse PDF)
    expect(result.metadata.totalTextLayers).toBe(mockAuditResult.totalTextLayers);
  });

  test('CSV should have correct number of rows', async () => {
    const result = await engine.generateCSV(mockAuditResult);

    // Parse CSV and count rows
    const csv = await fetch(result.blobUrl).then(r => r.text());
    const parsed = Papa.parse(csv, { header: true });

    expect(parsed.data.length).toBe(mockAuditResult.totalTextLayers);
  });

  test('should handle large datasets efficiently', async () => {
    const largeAuditResult = createMockAuditResult(25000); // 25k layers

    const start = performance.now();
    const result = await engine.generatePDF(largeAuditResult);
    const duration = performance.now() - start;

    expect(duration).toBeLessThan(5000); // <5 seconds
    expect(result.fileSize).toBeLessThan(5 * 1024 * 1024); // <5MB
  });
});
```

---

## Version History

- **v1.0** (2025-11-20): Initial Export Engine API
  - PDF generation with jsPDF + AutoTable
  - CSV generation with PapaParse
  - Text-based visualizations (charts deferred to v1.1)
  - Document metadata in all exports
