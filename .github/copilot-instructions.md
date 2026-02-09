# Copilot Instructions for gcode_tpgen

## Project Overview
Browser-based G-code test pattern generator for CNC machines, plotters, and MPCNC systems. Generates calibration patterns (rulers, acceleration tests, Z-leveling, surfacing) and renders text using bezier curves. Pure client-side JavaScript with no build process.

## Architecture

### Core Components
- **[index.html](index.html)**: Main UI with jQuery-based form handling and G-code generation logic (1147 lines)
- **[char_render.js](char_render.js)**: OCR-B font vector data and text-to-G-code rendering using G5 bezier splines (204 lines)
- **[checker.html](checker.html)**: Standalone G-code file analyzer/validator with statistics and warnings (482 lines)

### Key Data Flow
1. User selects pattern mode (radio buttons) → `show_options()` dynamically reveals relevant UI controls
2. "Generate" button → `generate()` function builds G-code string array via pattern-specific functions
3. Output appended to `<pre id="output">` via `output_append()` helper
4. No server communication - all generation happens in browser DOM

## Pattern Generation Patterns

### G-code Output Structure
All generation functions follow this convention:
```javascript
function pattern_name(...params, zup, zdn, rapid, vertical, drawspeed) {
  output_append("G0 X... Y... " + zup + " F" + rapid);      // rapid move with pen up
  output_append("G1 " + zdn + " F" + vertical);              // lower pen
  output_append("G1 X... Y... " + zdn + " F" + drawspeed);  // draw move
  output_append("G0 " + zup + " F" + vertical);              // raise pen
}
```
- `zup`/`zdn` are pre-formatted strings like `" Z0.5"`, `" Z-0.5"`
- All coordinates use `.toFixed(3)` or `.toFixed(4)` for precision
- Feedrates are integers (mm/min)

### Pattern-Specific Functions
- **Rulers**: `x_zig()`, `x_zag()`, `y_zig()`, `y_zag()` - draw tick marks at 1mm intervals
- **Z-tests**: `z_test()` - draws "X" patterns with ramping Z to amplify surface height errors (1:10 slope)
- **Dense segments**: `dense_x_segments()` - tests parser speed with many tiny G1 moves
- **Acceleration**: Uses `M201` (max accel), `M204` (default accel), bracketed by `M501` (restore)
- **Surfacing**: `surfacing()` + `surfacing_perim()` - unidirectional cutting with climb milling orientation
- **Text**: `render_text()` in [char_render.js](char_render.js) - converts OCR-B font to G5 bezier curves

## Critical Conventions

### UI State Management
- jQuery `.show()`/`.hide()` on `parent().parent()` targets `<tr>` rows (input → td → tr)
- Mode descriptions use `$("div[id=mode_desc]")` selectors
- Dynamic label updates: `$("td[name=z_pen_down_txt]")[0].innerHTML = ...` (surfacing vs. drawing terminology)

### Coordinate System Assumptions
- Origin (0,0) is typically starting position after optional `G92 X0 Y0 Z0`
- Positive X/Y extents only (no negative coordinates in standard patterns)
- Z: positive is "up" (pen raised), negative is "down" (cutting/drawing)

### Text Rendering Requirements
- **Requires firmware G5 support** (cubic bezier splines) - NOT enabled in default MPCNC firmware
- Font data structure in `ocrb` object: `chars[]` with `ch` and `cmds` (M/L/C commands)
- Scaling: `native_pitch` [2.55, 4.5] mm, output pitch calculated from extent / text dimensions
- Commands: `M` (move), `L` (line), `C` (bezier with control points I,J,P,Q)

### Hog-Out Mode Specifics
- Creates "top-hat" profile at slow speed, then fast cross-cut to measure deflection
- Purpose: experimentally determine max safe feedrate for material/bit combination
- Hides X/Y extent inputs (pattern has fixed dimensions)

## Development Workflow

### Testing Patterns
1. Open [index.html](index.html) in browser (no server needed)
2. Select mode, adjust parameters, click Generate
3. Copy G-code from `<pre>` output to clipboard
4. Test with CNC controller (Grbl, Marlin, etc.) or simulator

### Validating G-code
1. Open [checker.html](checker.html) in separate tab
2. Upload generated `.gcode` file
3. Reviews: missing feedrates, moves before G92/G28, parentheses (unsupported comments), X/Y/Z limits
4. Displays table with warnings/info (color-coded: yellow=warning, green=good)

### Adding New Patterns
1. Add radio button in mode list: `<input type="radio" name="mode" value="new_pattern">`
2. Create description div: `<div id="new_pattern_desc"><h1>Pattern Name</h1>...</div>`
3. Update `show_options()` to show/hide relevant controls and description
4. Add pattern generation logic in `generate()` function switch statement
5. Follow existing coordinate/feedrate conventions

## Common Pitfalls

### Surfacing vs. Drawing Modes
Different terminology/defaults apply when `mode == "surfacing" || mode == "hog"`:
- Labels change: "Z level for pen-down" → "Z height of cut (usually negative)"
- "Draw Feedrate" → "Cutting Feedrate"
- Much slower speeds required (defaults likely unsafe for real cutting)

### Acceleration Test Caveats
- Assumes junction deviation (not classic jerk) in firmware
- Uses `M501` to restore EEPROM settings after test (may not work on all controllers)
- Rapid feedrate becomes the test speed (not just rapid positioning)

### Dense Segments Bandwidth Test
- "Save bandwidth" mode omits redundant Z/Y/F parameters
- "Draw diagonally" uses `Math.sqrt(0.5)` for 45° segment length adjustment
- Can generate massive G-code files (100+ lines per mm of Y extent)

### Text Generation
- Text gets stretched/squished to fill X extent × Y extent rectangle
- Check comments in output for "x pitch" and "y/x aspect ratio" warnings
- Font is fixed-width only (OCR-B) - proportional fonts not supported
- Missing characters return empty G-code array (silently skip)

## File Organization
- No build system, bundler, or package.json - just HTML/JS/CSS
- jQuery loaded from CDN: `https://code.jquery.com/jquery-1.9.1.min.js`
- Single repository focus - see GitHub at `https://github.com/vector76/gcode_tpgen`
