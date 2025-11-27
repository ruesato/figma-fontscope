# Investigation Summary: Token Coverage & Library Variables

**Investigation Dates**: November 26, 2025  
**Status**: COMPLETE - INVESTIGATION PHASE  
**Branch**: 002-style-governance  

---

## What We Investigated

**Question**: Are library tokens included in Token Coverage calculations, and why don't their values appear in the UI?

**Scope**: 
- Token Coverage metric definition
- Library variable value accessibility via Figma Plugin API
- Impact on existing functionality
- Potential enhancements for v1.1

---

## Key Findings

### 1. Token Coverage Includes Library Tokens ✅

**Confirmed via code analysis**:
- `tokenDetection.ts` Step 1: Collects local tokens via `figma.variables.getLocalVariablesAsync()`
- `tokenDetection.ts` Step 2: Collects library tokens via `figma.teamLibrary.getAvailableLibraryVariableCollectionsAsync()`
- Both sources are merged into same tokenMap for coverage calculation

**Impact**: Token Coverage metric correctly counts all available tokens (local + library)

### 2. Library Variable Values Are Not Accessible (API Limitation) ❌

**Root Cause**: Figma Plugin API design - `LibraryVariable` interface only exposes:
- ✅ `name` (token name)
- ✅ `resolvedType` (token type: COLOR, STRING, FLOAT, BOOLEAN)
- ❌ `valuesByMode` (actual token values) - NOT AVAILABLE
- ❌ Mode information - NOT AVAILABLE

**Impact on v1.0**:
- Token inventory works correctly (names, types available)
- Token usage tracking works correctly (via `boundVariables`)
- Token replacement works correctly (uses variable keys)
- **Only issue**: Library token values display as empty in UI

### 3. Workaround Exists for v1.1 Enhancement ✅

**Available API**: `figma.importVariableByKeyAsync(key)`
- Takes library variable key (from `LibraryVariable.key`)
- Returns full `Variable` object with all properties
- **Enables access** to `valuesByMode` and other metadata
- Documented in official Figma Plugin API v1.109.0+

**Use Case**: Optional v1.1 enhancement to display library token values

### 4. Token Binding Detection Not Affected ✅

**Current Implementation**: Uses `textNode.boundVariables` property
- Works independently of variable value access
- Functions correctly for both local and library tokens
- No changes needed to existing binding detection

---

## Documentation Updates (Completed)

### 1. spec.md - Token Coverage Rate Section

**Added explicit scope documentation**:
- Clarified that Token Coverage includes local AND team library tokens
- Added API references showing both sources
- Documented the known limitation regarding library variable values
- Noted this is a Figma Plugin API constraint

### 2. data-model.md - DesignToken Entity

**Added known limitation section**:
- Documented which token properties are available for local vs library variables
- Referenced implementation details in tokenDetection.ts
- Explained why library variable values are unavailable

### 3. New Documents Created

**VARIABLES_API_REFERENCE.md**:
- Complete Figma Plugin API documentation for variables
- Available APIs and their capabilities
- Code examples from current implementation
- Detailed limitations matrix

**LIBRARY_VARIABLES_INVESTIGATION.md**:
- Full investigation findings and analysis
- Technical deep-dive on APIs and workarounds
- v1.0 vs v1.1 comparison
- Recommendations for future enhancements

---

## Current Implementation Status

| Aspect | Status | Notes |
|--------|--------|-------|
| Token Coverage Calculation | ✅ Works | Includes local + library tokens |
| Token Inventory | ✅ Works | Names and types available for both sources |
| Token Usage Tracking | ✅ Works | Binding detection works for both sources |
| Token Replacement | ✅ Works | Variable keys work for both sources |
| Library Token Values | ⚠️ Limited | Metadata only; empty placeholders in UI |
| Documentation | ✅ Updated | Limitation now clearly documented |

---

## Findings Impact on Development

### No Changes Needed to v1.0
- Current implementation is correct
- Token Coverage metric functions as designed
- Library tokens are properly included
- Limitation is documented

### Future Enhancement Path (v1.1)
- Use `figma.importVariableByKeyAsync()` to access library token values
- Optional feature with graceful degradation
- Low risk, low effort enhancement
- Can be implemented independently of v1.0

---

## Architecture Decisions

### Why Library Tokens Have Empty Values in v1.0

**Decision**: Accept API limitation, document clearly for v1.0

**Rationale**:
1. Figma Plugin API design constraint (not our limitation)
2. Importing all library variables on startup has performance cost
3. v1.0 scope focused on token counting and replacement (values not critical)
4. v1.1 can add value display as optional enhancement

### Why Token Coverage Includes Library Tokens

**Decision**: Include all tokens (local + library) in coverage calculation

**Rationale**:
1. Design system teams want visibility into full system (not just local)
2. Library tokens are linked to document and can be referenced
3. Provides complete picture of token system health
4. Consistent with feature goal: "comprehensive design system governance"

---

## References

### Documentation
- `spec.md` (002-style-governance) - Updated Token Coverage definition
- `data-model.md` (002-style-governance) - Updated DesignToken entity
- `VARIABLES_API_REFERENCE.md` - Complete API reference
- `LIBRARY_VARIABLES_INVESTIGATION.md` - Full investigation report

### Code
- `src/main/utils/tokenDetection.ts` - Token detection implementation
- `src/shared/types.ts` - DesignToken interface

### API
- Figma Plugin API v1.109.0 type definitions
- `figma.variables.*` - Local variable APIs
- `figma.teamLibrary.*` - Library variable APIs

---

## Conclusion

The investigation confirms that:

1. ✅ Token Coverage correctly includes library tokens
2. ❌ Library token values are not accessible (Figma API limitation)
3. ✅ This limitation is properly documented
4. ✅ A workaround exists for v1.1 enhancement
5. ✅ v1.0 is ready for release with documentation updates

**No blocking issues identified. Ready to proceed with planned workflow.**

---

**Investigation Status**: ✅ COMPLETE  
**Ready for**: Code review and v1.0 release  
**Next Phase**: v1.1 feature planning for library token value display  

**Created**: November 26, 2025  
**Project**: figma-fontscope / 002-style-governance  
**Author**: OpenCode (Investigation Mode)
