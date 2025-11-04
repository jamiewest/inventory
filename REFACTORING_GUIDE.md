# Inventory Manager - Refactoring Guide & Architecture Documentation

## Overview
This is a single-file inventory management web application (~4000 lines) that uses cameras to scan barcodes for asset tracking. It's designed to work offline-first with IndexedDB storage and has fallback UI when Bootstrap CDN fails.

## Architecture Overview

### 1. **Core Components**
- **Data Layer**: IndexedDB (primary) with localStorage migration fallback
- **UI Layer**: Bootstrap 5 (with fallback styling if CDN fails)
- **Scanner**: Three detection methods (Native BarcodeDetector, ZXing, jsQR)
- **Storage Format**: CSV import/export, JSON backup

### 2. **Scanner Architecture** (Most Complex Part)

#### Detection Strategy (Multi-layered)
The scanner uses a **three-tier detection system** that runs in this priority:

1. **jsQR Detection** (lines 3233-3245)
   - Runs FIRST on every scan loop
   - Detects QR codes using canvas-based image processing
   - Does 12 attempts (3 crops × 4 rotations)
   - **Performance Issue**: This is SLOW and causes setTimeout violations
   - **Why it runs first**: Historical reliability over native APIs

2. **Native BarcodeDetector API** (lines 3254-3272)
   - Runs SECOND if available in browser
   - Fast, hardware-accelerated
   - Handles: CODE_128, CODE_39, EAN_13, EAN_8, UPC_A, UPC_E, QR_CODE
   - Only runs if `window.BarcodeDetector` exists

3. **ZXing Fallback** (lines 3490-3528)
   - ALWAYS starts in background via `startZXingLoop()` (line 3442)
   - Uses callback-based continuous scanning
   - **Critical**: Must use `decodeFromVideoDevice(null, video, callback)` NOT `decodeFromVideoElement`
   - Blocked from `scheduleScanLoop` by `zxingActive` flag (line 3219)

#### Scanner State Management
```javascript
// Scanner context (dual UI support)
scanContexts = {
  modal: { /* Bootstrap modal UI */ },
  fallback: { /* Fallback modal UI */ }
}

// Scanner state
scanActive = false      // Camera is running
scanPaused = false      // Detection temporarily paused (match found)
zxingActive = false     // ZXing is running (blocks scheduleScanLoop)
scanDetector = null     // Native BarcodeDetector instance
zxingReader = null      // ZXing reader instance
```

#### Critical Scanner Functions
- `startScanner(useFallback)` - Initialize camera and detection
- `stopScanner()` - Clean up all resources
- `scheduleScanLoop(delay)` - Timer-based detection loop (jsQR + BarcodeDetector)
- `startZXingLoop()` - Start continuous ZXing scanning
- `presentScanMatch(code)` - Handle detected barcode
- `detectQrFromFrame(video)` - CPU-intensive QR detection

## Safe Refactoring Opportunities

### Priority 1: High-Impact, Low-Risk

#### 1.1 Add Scanner Documentation Comments
**Location**: Lines 2747-2890
**Impact**: High (comprehension)
**Risk**: None (comments only)

```javascript
// ======= Barcode Scanner Module =======
//
// ARCHITECTURE: Three-tier detection system
// 1. jsQR (canvas-based QR detection) - runs first, CPU intensive
// 2. Native BarcodeDetector API - runs second if available
// 3. ZXing library - runs continuously in background as ultimate fallback
//
// CRITICAL: ZXing must use decodeFromVideoDevice(null, video, callback)
//           NOT decodeFromVideoElement - the former is continuous, latter is one-shot
//
// DUAL UI: Supports both Bootstrap modals and fallback modals
//          Context switching via scanContexts.modal / scanContexts.fallback
```

#### 1.2 Document Detection Flow
**Location**: Line 3217 (before `scheduleScanLoop`)
**Impact**: High
**Risk**: None

```javascript
/**
 * scheduleScanLoop - Timer-based barcode detection loop
 *
 * DETECTION ORDER:
 * 1. Process pending payloads queue
 * 2. Run jsQR detection on video frame (12 attempts: 3 crops × 4 rotations)
 * 3. Run native BarcodeDetector if available
 * 4. Schedule next loop iteration
 *
 * IMPORTANT:
 * - Exits early if zxingActive (ZXing handles detection instead)
 * - jsQR detection is SLOW (~50-200ms) causing setTimeout violations
 * - Pauses when match found (scanPaused = true)
 *
 * @param {number} delay - Milliseconds before next scan attempt
 */
function scheduleScanLoop(delay=300){
```

#### 1.3 Document Serial Number Extraction Logic
**Location**: Lines 3163-3201
**Impact**: Medium
**Risk**: None

```javascript
/**
 * extractSerialCandidates - Extract potential Dell service tags from text
 *
 * PATTERN: Exactly 7 alphanumeric characters
 * FILTERING: Must have at least 3 unique characters (excludes "1111111", "AAAAAAA")
 * USE CASE: Dell service tags typically omit confusing chars (I, O, Q)
 *
 * @param {string} text - Raw barcode/QR payload
 * @returns {string[]} Array of 7-char serial candidates
 */
function extractSerialCandidates(text){
```

```javascript
/**
 * normalizeSerialCandidate - Normalize serial numbers for Dell assets
 *
 * NORMALIZATION RULES:
 * - Converts confusing digits to letters when surrounded by letters
 * - 0 → O, 1 → I, 5 → S, 6 → G, 8 → B, 9 → G
 * - Example: "ABC1DEF" → "ABCIDEF" (1 between letters becomes I)
 * - Falls back to uppercase if not exactly 7 chars after normalization
 *
 * @param {string} code - Serial number candidate
 * @returns {string} Normalized serial or uppercase original
 */
function normalizeSerialCandidate(code){
```

### Priority 2: Performance Optimizations (Low Risk)

#### 2.1 Optimize jsQR Detection
**Location**: Lines 3064-3089 (detectQrFromFrame)
**Impact**: High (performance)
**Risk**: Low (can add feature flag)

**Current**: 12 attempts (3 crops × 4 rotations) = ~100-200ms
**Proposed**: Make configurable

```javascript
// Add configuration constant
const QR_DETECTION_MODE = {
  FAST: { crops: 1, rotations: 0 },      // 1 attempt
  BALANCED: { crops: 2, rotations: 2 },   // 6 attempts
  THOROUGH: { crops: 3, rotations: 4 }    // 12 attempts (current)
};
let currentQrMode = QR_DETECTION_MODE.THOROUGH;

// Modify detectQrFromFrame to use currentQrMode
// Add UI toggle to switch modes
```

#### 2.2 Debounce jsQR When BarcodeDetector Available
**Location**: Line 3233
**Impact**: High (performance)
**Risk**: Low

**Rationale**: If native BarcodeDetector works, jsQR is redundant overhead
**Proposed**: Only run jsQR every Nth iteration when BarcodeDetector is active

```javascript
let jsQrSkipCounter = 0;
const JSQR_SKIP_INTERVAL = 3; // Run jsQR every 3rd iteration when BarcodeDetector exists

// In scheduleScanLoop, before jsQR detection:
if(scanDetector && jsQrSkipCounter++ % JSQR_SKIP_INTERVAL !== 0){
  // Skip jsQR this iteration, let BarcodeDetector handle it
} else {
  // Run jsQR detection
  const qrPayload = detectQrFromFrame(scanCtx.video);
  // ...
}
```

### Priority 3: Code Organization (Medium Risk)

#### 3.1 Extract Scanner Module
**Impact**: High (maintainability)
**Risk**: Medium (lots of dependencies)

**Current**: Scanner code mixed inline (lines 2747-3800)
**Proposed**: Extract to separate `<script>` block or comment-delineated section

```javascript
// ============================================================================
// SCANNER MODULE START
// ============================================================================
// Dependencies: scanModalEl, scanVideo, etc. (DOM elements must be loaded)
// External functions used: findRowBySerial, markRowVerified, objToRow
// External state modified: None (scanner is self-contained)
//
// Public API:
// - openScannerModal() - Main entry point
// ============================================================================

// ... all scanner code ...

// ============================================================================
// SCANNER MODULE END
// ============================================================================
```

#### 3.2 Add Section Headers Throughout File
**Impact**: Medium (navigation)
**Risk**: None

```javascript
// Lines ~1020-1100: ===== DATA LAYER: IndexedDB & Storage =====
// Lines ~1194-1240: ===== DATA MODEL: Schema & Constants =====
// Lines ~1384-1850: ===== UI LAYER: View Asset Modal =====
// Lines ~1885-2057: ===== FEATURES: Notes & Metadata =====
// Lines ~2191-2370: ===== TABLE: Filtering, Sorting, Pagination =====
// Lines ~2318-2368: ===== DIFF: Compare Local vs Official Inventory =====
// Lines ~2486-2670: ===== MODAL: Add Asset Form =====
// Lines ~2747-3800: ===== SCANNER: Barcode & QR Detection =====
// Lines ~3855+:     ===== INITIALIZATION & Event Wiring =====
```

### Priority 4: State Management (Higher Risk)

#### 4.1 Consolidate Scanner State Object
**Location**: Lines 2861-2885
**Impact**: Medium (clarity)
**Risk**: Medium (must update all references)

**Current**: 20+ global variables
```javascript
let scanCtx = null;
let scanStream = null;
let scanDetector = null;
// ... 17 more ...
```

**Proposed**: Single state object
```javascript
const scannerState = {
  // Core
  ctx: null,
  stream: null,
  detector: null,
  active: false,
  paused: false,

  // Detection
  zxing: { reader: null, active: false },
  loopTimer: null,
  pendingPayloads: [],

  // Canvas contexts
  qr: { canvas: null, ctx: null, rotateCanvas: null, rotateCtx: null },
  ocr: { canvas: null, ctx: null },

  // Current scan
  row: null,
  code: "",
  ignoredCodes: new Set(),

  // Camera
  facingMode: 'environment',
  zoom: 0,
  capabilities: {},
  track: null,
  zoomRange: null,

  // UI
  lastContext: null
};
```

**Benefit**: Single import/export, easier testing, clear ownership
**Downside**: Must update ~50+ references throughout scanner code

## Testing Considerations

### Critical Scanner Paths to Test
1. **Native BarcodeDetector path** (Chrome/Edge on desktop)
   - Verify barcodes detected
   - Check no setTimeout violations

2. **ZXing fallback path** (Firefox, Safari)
   - Mock `BarcodeDetector` as undefined
   - Verify `decodeFromVideoDevice` is called with correct args

3. **jsQR QR code detection**
   - Test with QR codes containing URLs
   - Test serial number extraction from QR payload

4. **Camera switching**
   - Front/back camera toggle
   - Zoom controls
   - Verify no memory leaks on repeated open/close

5. **Serial normalization**
   - Test 7-char serials with numbers: "ABC1234"
   - Verify normalization: "1" → "I" when between letters
   - Test non-7-char barcodes (should pass through raw)

### Edge Cases to Document
1. **What happens with non-7-character barcodes?**
   - Falls back to raw barcode value (line 3260: `candidate = normalized || value.trim()`)

2. **Why does ZXing always start even with BarcodeDetector?**
   - Defense in depth - ZXing continues if BarcodeDetector fails mid-scan
   - ZXing blocked from scheduleScanLoop by `zxingActive` check (line 3219)

3. **Why is jsQR so slow?**
   - Canvas operations (drawImage, getImageData) are CPU-bound
   - 12 decode attempts with rotation transforms
   - Consider WebWorker for future optimization

## Common AI Assistant Pitfalls

### When Working on Scanner Code:

❌ **DON'T** change `decodeFromVideoDevice` to `decodeFromVideoElement`
   - The former is continuous, the latter is one-shot
   - This bug was already fixed (see git commit 61188ae)

❌ **DON'T** move jsQR detection after BarcodeDetector
   - Historical precedence shows jsQR runs first for reliability
   - While this seems inefficient, it may be intentional

❌ **DON'T** remove the `zxingActive` check in `scheduleScanLoop`
   - Prevents dual detection systems from running simultaneously
   - ZXing uses requestAnimationFrame, scheduleScanLoop uses setTimeout

❌ **DON'T** make ZXing conditional on `!scanDetector`
   - Current code runs ZXing always as ultimate fallback
   - Line 3442: `startZXingLoop()` is outside the conditional

✅ **DO** test on multiple browsers
   - Chrome: Has BarcodeDetector
   - Firefox: No BarcodeDetector, falls back to ZXing
   - Safari iOS: Camera permissions, facing mode

✅ **DO** preserve the dual UI system
   - Bootstrap modals (scanContexts.modal)
   - Fallback modals (scanContexts.fallback)
   - Critical for offline functionality

✅ **DO** check git history before major changes
   - This codebase has ~40 commits all labeled "updates"
   - Use `git show <commit>:file.html | grep` to find old implementations

## Quick Reference: Key Functions

### Scanner Lifecycle
```
User clicks "Scan barcode"
  → openScannerModal() (line 3613)
  → startScanner(useFallback) (line 3344)
  → Request camera permission
  → Start three detection systems:
     1. scheduleScanLoop(350) - Native BarcodeDetector + jsQR
     2. startZXingLoop() - ZXing continuous scanning
  → Detection finds barcode
  → presentScanMatch(code) (line 3282)
  → User confirms/ignores
  → handleScanConfirm() or handleScanIgnore()
  → stopScanner() when modal closes
```

### Data Flow
```
CSV Upload → parseCSV() → upgradeRow() → officialRows[]
User Input → addRow() → localBody (DOM) → readLocal() → saveLocalToIDB()
Scanner → presentScanMatch() → findRowBySerial() → markRowVerified()
Export → readLocal() → toCSV() / toMarkdown() → download
```

### State Persistence
```
IndexedDB (primary)
  ├─ 'localRows' → inventory data
  └─ 'statusOptions' → custom status values

localStorage (migration fallback)
  └─ 'inv.local.rows.v1' → legacy data

Memory (runtime)
  ├─ localBody (DOM table)
  ├─ officialRows[] (comparison baseline)
  └─ tableState{} (search, filter, sort, pagination)
```

## Recommended Next Steps

1. **Add Comments** (Priority 1)
   - Scanner architecture block comment
   - Function-level JSDoc comments
   - Section headers throughout file

2. **Performance Profiling** (Priority 2)
   - Measure jsQR detection time
   - Test with/without jsQR skip optimization
   - Compare setTimeout vs requestAnimationFrame for scheduling

3. **Extract Constants** (Priority 3)
   - Scanner timing values (350ms, 900ms, 200ms delays)
   - QR detection configuration
   - Serial number pattern (7 chars)

4. **Consider File Split** (Future)
   - scanner.js (lines 2747-3800)
   - data.js (IndexedDB, CSV parsing)
   - ui.js (modals, tables)
   - main.html (structure only)

## File Structure Summary

```
Lines    | Section
---------|--------------------------------------------------------
1-910    | HTML: Structure, styles, modals
910-1010 | JS: Bootstrap loading, routing
1010-1100| JS: IndexedDB & storage utilities
1100-1190| JS: CSV/data utilities
1190-1380| JS: Data model & constants
1380-1890| JS: View asset modal & UI
1890-2290| JS: Notes, metadata, table operations
2290-2480| JS: Status management, diff view
2480-2750| JS: Add asset modal
2750-3860| JS: SCANNER MODULE (most complex)
3860-4070| JS: Event wiring & initialization
```

## Conclusion

This is a well-structured single-file application with **good offline-first design** and **defensive scanning architecture**. The scanner's three-tier detection is intentional redundancy for maximum compatibility.

**Safe refactoring**: Add comments, extract constants, performance tune jsQR
**Risky refactoring**: State consolidation, file splitting, detection order changes

**When in doubt**: Check git history, test on multiple browsers, preserve dual UI system.
