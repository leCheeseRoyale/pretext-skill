---
name: pretext
description: >
  Guide for building with @chenglou/pretext — a pure TypeScript text measurement & layout engine.
  TRIGGER when: user asks about text measurement, text layout, pretext, line breaking, text height prediction,
  shrinkwrap text, magazine layout, masonry text, chat bubble sizing, text virtualization, canvas text layout,
  overflow-wrap, multiline text measurement, or imports from '@chenglou/pretext'.
  DO NOT TRIGGER when: user is working with CSS-only text layout, DOM measurement, or unrelated text processing.
---

# Pretext: Pure TypeScript Text Measurement & Layout Engine

You are an expert in building with `@chenglou/pretext` — a lightweight, pure JavaScript/TypeScript library that delivers fast, accurate multiline text measurement and layout entirely in userland, without touching the DOM or CSS.

## What Pretext Is

Pretext uses the browser's native font engine (via a hidden canvas `measureText`) as ground truth, then implements its own high-performance measurement and line-breaking logic. This completely bypasses expensive DOM reflows (`getBoundingClientRect`, `offsetHeight`) while matching browser behavior across Chrome, Safari, Firefox, all languages, emojis, mixed bidirectional text, and platform-specific quirks.

**Key insight**: Text layout was the last major bottleneck preventing truly dynamic UIs. DOM read/write interleaving destroys performance and wrecks component boundaries. Pretext removes this bottleneck entirely.

## Installation

```bash
npm install @chenglou/pretext
# or
bun add @chenglou/pretext
```

## Architecture: Two-Phase Model

1. **`prepare()` phase** (one-time, ~19ms for 500 texts): Normalize whitespace, segment text via `Intl.Segmenter`, apply script-specific rules (CJK kinsoku, Arabic combining, URLs, etc.), measure segments via canvas, cache widths.
2. **`layout()` phase** (hot path, ~0.09ms for 500 texts): Pure arithmetic over cached widths. No DOM, no canvas calls, no string work, no allocations.

This means `prepare()` runs once per text+font combination, then `layout()` can be called thousands of times at different widths for virtually free.

## Complete API Reference

### Use Case 1: Height Prediction (Simple, Most Common)

```ts
import { prepare, layout } from '@chenglou/pretext'

// Step 1: Prepare text (one-time cost)
const prepared = prepare(
  'AGI 春天到了. بدأت الرحلة',
  '16px Inter',           // Must match your CSS font exactly
  { whiteSpace: 'normal' } // Optional: 'normal' (default) or 'pre-wrap'
)

// Step 2: Layout at any width (near-zero cost, call repeatedly)
const { height, lineCount } = layout(prepared, 320, 20)
// height = lineCount * lineHeight
```

**When to use**: Virtualizing lists, predicting container heights, accordion animations, masonry layouts — anywhere you need text height without DOM measurement.

### Use Case 2: Full Line Layout (Rich Rendering)

```ts
import { prepareWithSegments, layoutWithLines } from '@chenglou/pretext'

const prepared = prepareWithSegments(text, '16px Inter')
const { height, lineCount, lines } = layoutWithLines(prepared, maxWidth, lineHeight)

for (const line of lines) {
  // line.text   — full text content of the line
  // line.width  — measured pixel width
  // line.start  — { segmentIndex, graphemeIndex }
  // line.end    — exclusive end cursor
  ctx.fillText(line.text, x, y)
  y += lineHeight
}
```

**When to use**: Canvas/SVG/WebGL text rendering, custom text editors, rich text display.

### Use Case 3: Non-Materializing Line Walk (Shrinkwrap / Search)

```ts
import { prepareWithSegments, walkLineRanges } from '@chenglou/pretext'

const prepared = prepareWithSegments(text, font)

// Calls onLine per line WITHOUT building text strings — fastest for width search
const lineCount = walkLineRanges(prepared, maxWidth, (line) => {
  // line.width, line.start, line.end (no .text — use layoutWithLines if you need text)
})
```

**When to use**: Binary-searching for tightest shrinkwrap width, counting lines without materializing text.

### Use Case 4: Variable-Width Line Layout (Obstacle Flow)

```ts
import { prepareWithSegments, layoutNextLine } from '@chenglou/pretext'

const prepared = prepareWithSegments(text, font)
let cursor = { segmentIndex: 0, graphemeIndex: 0 }
let y = startY

while (true) {
  // Each line can have a DIFFERENT maxWidth
  const availableWidth = getWidthAtY(y) // e.g., narrower where an image sits
  const line = layoutNextLine(prepared, cursor, availableWidth)
  if (line === null) break

  ctx.fillText(line.text, getXAtY(y), y)
  cursor = line.end
  y += lineHeight
}
```

**When to use**: Magazine layouts flowing around images/shapes, multi-column text, editorial engines, obstacle-aware text routing.

### Helper Functions

```ts
import { clearCache, setLocale } from '@chenglou/pretext'

// Clear all internal caches (segment metrics, emoji correction, analysis caches)
// Useful when cycling through many fonts
clearCache()

// Set locale for Intl.Segmenter used in future prepare() calls
// Also clears caches. Does NOT mutate existing prepared handles
setLocale('ja-JP')
```

## Types

```ts
type PreparedText // Opaque handle from prepare()

type PreparedTextWithSegments = PreparedText & {
  segments: string[]           // e.g., ['hello', ' ', 'world']
  widths: number[]             // pixel width per segment
  kinds: SegmentBreakKind[]    // break behavior per segment
  breakableWidths: (number[] | null)[]  // grapheme widths for overflow-wrap
  discretionaryHyphenWidth: number
}

type LayoutResult = { lineCount: number; height: number }
type LayoutLinesResult = LayoutResult & { lines: LayoutLine[] }

type LayoutLine = {
  text: string
  width: number
  start: LayoutCursor
  end: LayoutCursor
}

type LayoutLineRange = {
  width: number
  start: LayoutCursor
  end: LayoutCursor
}

type LayoutCursor = {
  segmentIndex: number
  graphemeIndex: number
}

type SegmentBreakKind =
  | 'text' | 'space' | 'preserved-space' | 'tab'
  | 'glue' | 'zero-width-break' | 'soft-hyphen' | 'hard-break'
```

## Configuration Options

The `options` parameter in `prepare()` / `prepareWithSegments()`:

| Option | Values | Default | Description |
|---|---|---|---|
| `whiteSpace` | `'normal'`, `'pre-wrap'` | `'normal'` | `'normal'`: collapses whitespace, trims edges. `'pre-wrap'`: preserves spaces, tabs (with `tab-size: 8` stops), and `\n` hard breaks. |

**CSS target**: Pretext matches the behavior of:
```css
white-space: normal; /* or pre-wrap */
word-break: normal;
overflow-wrap: break-word;
line-break: auto;
```

It does NOT support `break-all`, `keep-all`, `strict`, `loose`, or `anywhere`.

## Common Patterns

### Pattern 1: Shrinkwrap (Tightest Bubble Width)

Binary search for the narrowest width that doesn't increase line count:

```ts
import { prepare, layout } from '@chenglou/pretext'

function shrinkwrap(text: string, font: string, maxWidth: number, lineHeight: number) {
  const prepared = prepare(text, font)
  const { lineCount } = layout(prepared, maxWidth, lineHeight)

  if (lineCount <= 1) {
    // Single line — shrinkwrap to actual text width
    // Use layoutWithLines to get the actual line width
    return // handle single-line case
  }

  let lo = 1
  let hi = Math.ceil(maxWidth)

  while (lo < hi) {
    const mid = Math.floor((lo + hi) / 2)
    if (layout(prepared, mid, lineHeight).lineCount <= lineCount) {
      hi = mid
    } else {
      lo = mid + 1
    }
  }

  return lo // tightest width that preserves line count
}
```

### Pattern 2: Virtualized List with Precomputed Heights

```ts
import { prepare, layout } from '@chenglou/pretext'

const font = '14px Inter'
const lineHeight = 20
const containerWidth = 400

// Prepare all items once
const items = texts.map(text => ({
  text,
  prepared: prepare(text, font),
}))

// Compute heights (near-zero cost per item)
const heights = items.map(item =>
  layout(item.prepared, containerWidth, lineHeight).height
)

// Build cumulative offset array for virtualization
const offsets = [0]
for (let i = 0; i < heights.length; i++) {
  offsets.push(offsets[i] + heights[i])
}

// On resize — just re-run layout() with new width (no re-prepare needed)
function onResize(newWidth: number) {
  for (let i = 0; i < items.length; i++) {
    heights[i] = layout(items[i].prepared, newWidth, lineHeight).height
  }
  // rebuild offsets...
}
```

### Pattern 3: Accordion with Animated Height

```ts
import { prepare, layout } from '@chenglou/pretext'

function AccordionPanel({ text, isOpen }) {
  const preparedRef = useRef(prepare(text, '16px Inter'))

  const targetHeight = isOpen
    ? layout(preparedRef.current, containerWidth, 24).height + padding * 2
    : 0

  // Animate to targetHeight — no DOM measurement needed
  return <div style={{ height: targetHeight, transition: 'height 0.3s' }}>
    <p>{text}</p>
  </div>
}
```

### Pattern 4: Text Flowing Around an Obstacle

```ts
import { prepareWithSegments, layoutNextLine } from '@chenglou/pretext'

function flowAroundImage(text: string, font: string, columnWidth: number,
  image: { x: number, width: number, top: number, bottom: number },
  lineHeight: number) {

  const prepared = prepareWithSegments(text, font)
  let cursor = { segmentIndex: 0, graphemeIndex: 0 }
  const lines: Array<{ text: string; x: number; y: number }> = []
  let y = 0

  while (true) {
    // Narrow the line width where the image sits
    let availableWidth = columnWidth
    let xOffset = 0

    if (y + lineHeight > image.top && y < image.bottom) {
      // Line overlaps with image — reduce available width
      availableWidth = columnWidth - image.width - 10 // 10px gap
      if (image.x === 0) xOffset = image.width + 10   // image on left
    }

    const line = layoutNextLine(prepared, cursor, availableWidth)
    if (line === null) break

    lines.push({ text: line.text, x: xOffset, y })
    cursor = line.end
    y += lineHeight
  }

  return lines
}
```

### Pattern 5: Multi-Column Flow (Cursor Handoff)

```ts
import { prepareWithSegments, layoutNextLine } from '@chenglou/pretext'

function multiColumnFlow(text: string, font: string, columns: number,
  columnWidth: number, columnHeight: number, lineHeight: number) {

  const prepared = prepareWithSegments(text, font)
  let cursor = { segmentIndex: 0, graphemeIndex: 0 }
  const result: Array<Array<{ text: string; y: number }>> = []

  for (let col = 0; col < columns; col++) {
    const colLines: Array<{ text: string; y: number }> = []
    let y = 0

    while (y + lineHeight <= columnHeight) {
      const line = layoutNextLine(prepared, cursor, columnWidth)
      if (line === null) break

      colLines.push({ text: line.text, y })
      cursor = line.end  // Hand off cursor to next column
      y += lineHeight
    }

    result.push(colLines)
    if (layoutNextLine(prepared, cursor, columnWidth) === null) break
  }

  return result
}
```

### Pattern 6: Canvas Text Rendering

```ts
import { prepareWithSegments, layoutWithLines } from '@chenglou/pretext'

function renderTextOnCanvas(ctx: CanvasRenderingContext2D, text: string,
  font: string, x: number, y: number, maxWidth: number, lineHeight: number) {

  ctx.font = font
  ctx.textBaseline = 'top'

  const prepared = prepareWithSegments(text, font)
  const { lines } = layoutWithLines(prepared, maxWidth, lineHeight)

  for (const line of lines) {
    ctx.fillText(line.text, x, y)
    y += lineHeight
  }
}
```

## Critical Rules & Gotchas

### MUST follow:
1. **Font string must match CSS exactly** — The `font` parameter uses canvas font shorthand (e.g., `'16px Inter'`, `'bold 14px "Helvetica Neue"'`). It MUST match your CSS `font` declaration or measurements will be wrong.
2. **`lineHeight` must match CSS** — Pass the same numeric value as your CSS `line-height`.
3. **Never use `system-ui` font** — Canvas and DOM resolve `system-ui` to different font variants on macOS. Always use named fonts.
4. **`prepare()` is the expensive call** — Call it once, cache the result. `layout()` is essentially free.
5. **Reuse prepared handles** — If text and font haven't changed, reuse the `PreparedText`. Only re-prepare when the text or font changes.

### AVOID:
1. **Don't re-prepare on every render** — `prepare()` costs ~0.04ms per text. Wasted if text hasn't changed.
2. **Don't use `layoutWithLines` when you only need height** — Use `layout()` instead; it's faster because it doesn't build line objects.
3. **Don't use `layoutWithLines` for width search** — Use `walkLineRanges()` instead; it skips string materialization.
4. **Don't expect `break-all` or `keep-all` behavior** — Pretext targets `word-break: normal` with `overflow-wrap: break-word` only.
5. **Don't forget to call `clearCache()` when cycling many fonts** — The internal cache grows unbounded otherwise.

### Browser-specific behavior (handled internally):
- **Safari/WebKit**: Uses `lineFitEpsilon = 1/64`, prefers prefix widths for breakable runs, prefers early soft-hyphen breaks.
- **Chrome/Firefox**: Uses `lineFitEpsilon = 0.005`.
- **Emoji on macOS**: Chrome/Firefox measure emoji wider in canvas than DOM at small sizes. Pretext auto-detects and corrects this.

## Performance Reference

| Operation | Time (500 texts) | Notes |
|---|---|---|
| `prepare()` batch | ~19ms | One-time cost. Script-heavy text (Arabic, CJK) can be slower |
| `layout()` batch | ~0.09ms | Pure arithmetic. ~0.0002ms per text |
| `layoutWithLines()` | ~0.05ms | Slightly more than layout() due to line object creation |
| `walkLineRanges()` | ~0.03ms | Fastest line-aware API (no string materialization) |
| `layoutNextLine()` | ~0.07ms | Per-call overhead for variable-width flexibility |
| DOM batch measurement | ~4-44ms | 500 reflows. Safari: up to 87ms |

## Choosing the Right API

```
Need just height/lineCount?
  → prepare() + layout()

Need line text and widths for rendering?
  → prepareWithSegments() + layoutWithLines()

Need to search for optimal width (shrinkwrap)?
  → prepareWithSegments() + walkLineRanges()

Need variable-width lines (obstacles, columns)?
  → prepareWithSegments() + layoutNextLine()
```

## Supported Text Features

- Full Unicode: CJK, Arabic/RTL, Thai, Myanmar, Devanagari, emojis
- Mixed bidirectional text (simplified bidi support)
- Soft hyphens (`\u00AD`) — invisible when unbroken, shows `-` at break
- Non-breaking spaces (NBSP, NNBSP, Word Joiner) — treated as non-breaking glue
- Zero-width space (ZWSP) — explicit break opportunity
- URL detection and smart breaking
- CJK kinsoku (line-start/end prohibition rules)
- Tab characters (with `tab-size: 8` stops in `pre-wrap` mode)

## When NOT to Use Pretext

- You need full CSS text layout features (`break-all`, `keep-all`, vertical text, etc.)
- You need font shaping or advanced OpenType features
- Server-side rendering without canvas (needs `OffscreenCanvas` or DOM `<canvas>`)
- You only have a single text element and performance isn't a concern — just use DOM measurement

## Development (Contributing)

```bash
bun install
bun start          # dev server at http://localhost:3000
bun run check      # typecheck + lint
bun test           # invariant test suite
bun run build:package  # emit dist/
```
