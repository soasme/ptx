# PTX — Pixel Text Exchange Format
**Version 1.2.0**

PTX is a plain-text file format for representing and editing pixel art — static or animated, small or large. It is designed to be read and written by humans, coding models, and standard text tooling alike.

`.ptx` files are source code for pixel art. Git diffs, patches, code review, tests, linting, formatting, and coding agents all work on them natively.

---

## Design Principles

1. **Human-readable and writable.** No indent requirements. No mandatory quoting. Structure is conveyed by section headers, not nesting.
2. **Diff-friendly.** Every pixel, palette entry, layer, frame, and chunk is a line. Standard `diff`, `patch`, and `git` apply cleanly.
3. **Model-friendly.** A coding model never needs to reason about more than 32×32 = 1024 symbols at a time. Big sprites are chunked.
4. **Thinking-first.** A special thinking layer lets models sketch a low-res mental map of a large sprite. It is never rendered.
5. **Composable.** Layers, frames, and chunks are all first-class. Any combination is valid.

---

## File Structure

A `.ptx` file is a sequence of sections. Each section begins with a header line in square brackets. Sections may appear in any order, except that `[meta]` is conventionally first.

Blank lines and lines beginning with `#` are ignored (comments).

```
[meta]
[palette]
[frames]          # optional — omit for static art
[layer <name>]
[chunk <id> ...]
```

---

## `[meta]`

Describes the overall sprite.

```
[meta]
size 128x128
type animated       # static | animated
tile_size 32        # chunk grid size; must be 1–32 (default 32)
background transparent
```

| Key | Values | Default |
|---|---|---|
| `size` | `WxH` in pixels | required |
| `type` | `static` \| `animated` | `static` |
| `tile_size` | integer 1–32 | `32` |
| `background` | `transparent` or `#rrggbb` | `transparent` |

---

## `[palette]`

Maps single-character symbols to colors. Symbols are palette indices — a symbol means a color only because the palette says so.

```
[palette]
. transparent
K #000000
W #ffffff
R #ff3b30
r #9f1f1f
+ #ffd166
@ #5c7cfa
```

**Rules:**
- One entry per line: `<symbol> <color>`
- `<color>` is `transparent` or a CSS hex color (`#rrggbb` or `#rrggbbaa`)
- `.` (dot) conventionally means transparent; redefining it is allowed
- Symbol pool: `. a-z A-Z 0-9 @ % ^ * + = _ | ~ < > [ ] { } ? ! - / : ; ( ) $ &`
  This gives up to 84 distinct colors per file

---

## `[frames]`

Defines named frames and their display durations. Required for animated sprites; omit for static art.

```
[frames]
idle_1  120ms
walk_1   90ms
walk_2   90ms
```

Each line: `<frame_name> <duration>`. Duration is an integer followed by `ms`.

---

## `[layer <name>]`

Declares a named layer. Layer content is provided by the chunks that reference it.

### Layer attributes

```
[layer <name> order=<n> blend=<mode>]
```

| Attribute | Description | Default |
|---|---|---|
| `order` | Render order integer. Lower numbers render first (bottom). Higher numbers render on top. | Declaration order (0-based) |
| `blend` | Compositing mode for this layer onto the result below it. | `normal` |
| `frame` | (body line) Restrict this layer to specific frames. Repeatable. | all frames |

**`order`** is the authoritative render sequence. Declaration order in the file is the fallback when `order` is omitted. Explicit `order` values need not be contiguous — using multiples of 10 leaves room for insertions without renumbering.

```
[layer shadow  order=10 blend=multiply]
[layer fill    order=20 blend=normal]
[layer outline order=30 blend=normal]
[layer light   order=40 blend=screen]
[layer fx      order=50 blend=add]
```

Layers are composited bottom-up: `shadow` → `fill` → `outline` → `light` → `fx`.

### Blend modes

| Mode | Description |
|---|---|
| `normal` | Standard alpha compositing (Porter-Duff over). Default. |
| `multiply` | Multiplies pixel values. Darkens; transparent pixels pass through. |
| `screen` | Inverse multiply. Brightens; good for light/glow effects. |
| `overlay` | Multiply where base is dark, screen where base is light. |
| `add` | Additive blend. Pixels add up; clamps to white. Good for fire/sparks. |
| `subtract` | Subtracts layer from base. Clamps to black. |
| `replace` | Replaces every pixel in the layer region, ignoring alpha. |
| `erase` | Cuts the layer's painted pixels out of the layers below it. |

Blend modes apply per-pixel. Transparent pixels (`.` or `#rrggbb00`) in any layer are always a no-op regardless of blend mode.

### Frame restriction

Optional `frame` body lines restrict a layer to specific frames. Omitting all `frame` lines means the layer applies to every frame.

```
[layer outline order=30 blend=normal]
frame walk_1
frame walk_2
```

### Thinking Layer

A layer named `thinking` (or prefixed `thinking_`) is a **non-rendered** sketch layer. It exists only as a model aid for reasoning about overall shape or composition. It is stripped at export time and never participates in compositing, regardless of `order` or `blend`.

Thinking layers carry explicit metadata so parsers and models know exactly what the grid represents:

```
[layer thinking render=false scale=4 size=32x32 purpose=whole_sprite_silhouette]
```

| Attribute | Description | Default |
|---|---|---|
| `render` | Must be `false`. Marks this layer as non-rendered and export-stripped. | required |
| `scale` | Each symbol in this layer's chunks represents an `N×N` block of real pixels. | `1` |
| `size` | Grid dimensions for this layer's chunks in symbols (not pixels). `WxH` where `W = sprite_width / scale`, `H = sprite_height / scale`. | derived |
| `purpose` | Free-form label describing what this layer models. Parsers ignore it; models use it. | — |

**`scale` and real pixels:** A thinking layer with `scale=4` on a 128×128 sprite uses a 32×32 symbol grid. Each symbol covers a 4×4 pixel block. The layer gives a model a full-sprite overview in 1024 symbols instead of 16384.

**`size` is advisory but explicit.** If `size` is omitted, parsers derive it as `(sprite_width / scale) x (sprite_height / scale)`. If provided, it must match — a mismatch is a validation error.

**`render=false` is mandatory** on thinking layers. Parsers must reject any layer named `thinking` or `thinking_*` that omits `render=false`, to prevent accidental export.

**`purpose`** is a model-readable label. Suggested values: `whole_sprite_silhouette`, `chunk_layout`, `animation_key_pose`, `color_zones`. Any value is valid; it is never interpreted by parsers.

```
[layer thinking render=false scale=4 size=32x32 purpose=whole_sprite_silhouette]
# Each symbol = 4×4 real pixels. 32×32 grid covers full 128×128 sprite.
# K = dark mass, W = skin, . = empty

[chunk thinking_full x=0 y=0 w=32 h=32 layer=thinking]
................................
................................
.............KKKKKK.............
...........KKWWWWWWKK...........
...........KWWWWWWWWK...........
.............KKKKKK.............
...........KKKRRRRKKKK..........
.........KKKRRRRRRRRKKKK........
................................
................................
```

Thinking layers are skipped entirely in the render pipeline (step 4 of the render order summary).

### Render order summary

1. Start with the sprite background color from `[meta]`
2. Sort all non-thinking layers by `order` ascending
3. For each layer (bottom to top): composite its chunks onto the canvas using `blend`
4. Thinking layers are skipped entirely
5. Output the final composited canvas per frame

---

## `[chunk <id> ...]`

A chunk is a rectangular grid patch. All chunks have a maximum size of **32×32**. The minimum is **1×1**.

### Header syntax

```
[chunk <id> x=<n> y=<n> w=<n> h=<n>]
[chunk <id> x=<n> y=<n> w=<n> h=<n> name=<label>]
[chunk <id> x=<n> y=<n> w=<n> h=<n> name=<label> bg=<color>]
[chunk <id> x=<n> y=<n> w=<n> h=<n> layer=<layer> frame=<frame>]
```

| Attribute | Description |
|---|---|
| `id` | Chunk identifier, e.g. `A1`, `B2`, `torso`, `eye_L` |
| `x`, `y` | Top-left pixel coordinate within the sprite |
| `w`, `h` | Width and height in pixels (1–32) |
| `name` | Optional semantic label (e.g. `head`, `torso`, `left_arm`) |
| `bg` | Optional background fill color for this chunk (hex or `transparent`) |
| `layer` | Layer this chunk belongs to (default: first declared layer, or `base`) |
| `frame` | Frame this chunk belongs to (default: applies to all frames) |

### Grid encoding

Immediately following the header, grid rows are written as plain symbol strings — one row per line, one symbol per pixel. No quotes, no delimiters, no indentation.

```
[chunk A1 x=0 y=0 w=32 h=32 name=head]
................................
.............KKKKKK.............
...........KKWWWWWWKK...........
...........KWWWWWWWWK...........
.............KKKKKK.............
................................
```

- Row count must equal `h`
- Each row must be exactly `w` characters
- Every symbol must be defined in `[palette]`

### Chunk naming conventions

Chunk IDs use a grid-letter + grid-number scheme for spatial chunks, or a semantic name for named regions:

```
A1  B1  C1  D1       ← top row, left to right
A2  B2  C2  D2       ← second row
A3  B3  C3  D3
```

Semantic names like `head`, `torso`, `left_arm`, `eye_L` are encouraged when the chunk has clear anatomical or structural meaning.

---

## Large Sprites — Chunking

When a sprite exceeds 32×32 pixels, it must be broken into chunks. Each chunk covers at most 32×32 pixels.

| Sprite size | Chunks needed |
|---|---|
| 32×32 | 1 (A1) |
| 64×64 | 4 (A1 B1 A2 B2) |
| 128×128 | 16 (A1–D4) |
| 64×32 | 2 (A1 B1) |

Chunks tile the sprite with no gaps and no overlaps. A `bg` color on a chunk fills any pixels not explicitly painted, so sparse grids stay compact.

---

## Full Example

```ptx
# Example: 128×128 animated character, walk cycle

[meta]
size 128x128
type animated
tile_size 32
background transparent

[palette]
. transparent
K #000000
W #ffffff
R #ff3b30
r #9f1f1f
+ #ffd166
@ #5c7cfa

[frames]
idle_1 120ms
walk_1  90ms
walk_2  90ms

# Thinking layer — scale=4 means each symbol = 4×4 real pixels
# 32×32 symbol grid covers the full 128×128 sprite
[layer thinking render=false scale=4 size=32x32 purpose=whole_sprite_silhouette]

[layer fill    order=10 blend=normal]
frame walk_1
frame walk_2

[layer outline order=20 blend=normal]
frame walk_1
frame walk_2

[layer light   order=30 blend=screen]

# --- walk_1 frame, outline layer ---

[chunk A1 x=0 y=0 w=32 h=32 name=head layer=outline frame=walk_1]
................................
.............KKKKKK.............
...........KKWWWWWWKK...........
...........KWWWWWWWWK...........
..........KW++WWWW++WK..........
..........KW++WWWW++WK..........
...........KWWWWWWWWK...........
...........KWWWWWWWWK...........
.............KKKKKK.............
................................
................................
................................
................................
................................
................................
................................
................................
................................
................................
................................
................................
................................
................................
................................
................................
................................
................................
................................
................................
................................
................................
................................

[chunk B2 x=32 y=32 w=32 h=32 name=torso layer=outline frame=walk_1]
................................
............KKKK................
..........KKRRRRKK..............
..........KRRrrRRK..............
..........KRRrrRRK..............
..........KRRrrRRK..............
..........KKRRRRKK..............
............KKKK................
................................
................................
................................
................................
................................
................................
................................
................................
................................
................................
................................
................................
................................
................................
................................
................................
................................
................................
................................
................................
................................
................................
................................
................................
```

---

## Editing with Standard Tools

Because `.ptx` is plain text with one-pixel-per-character grids, standard text tooling operates on it naturally.

**Unified diff:**
```diff
-..........KW++WWWW++WK..........
+..........KW@@WWWW@@WK..........
```
This changes two eye pixels from gold (`+`) to blue (`@`) in chunk A1.

**Patch application:**
```sh
patch sprite.ptx fix_eye_color.patch
```

**Model editing:**
A coding model can be asked to "change the torso color from red to blue in frame walk_1" and it needs only to read `[chunk B2 ...]` (≤1024 symbols), make targeted symbol substitutions, and output a diff. No full-image context required.

---

## Validation Rules

1. Every symbol used in any chunk grid must appear in `[palette]`
2. Every grid row must have exactly `w` characters
3. Every chunk must have exactly `h` rows
4. Chunk coordinates must not exceed the sprite `size`
5. Chunks must not overlap (within the same layer and frame)
6. `tile_size` must be between 1 and 32 inclusive
7. Thinking layers must not be referenced by `[chunk]` entries that lack `layer=thinking`
8. Frame names referenced in chunks must be declared in `[frames]`
9. Two layers may not share the same `order` value
10. `blend` must be one of: `normal multiply screen overlay add subtract replace erase`

---

## MIME Type and Extension

| | |
|---|---|
| Extension | `.ptx` |
| MIME type | `text/x-ptx` |
| Encoding | UTF-8 |

---

## Changelog

### 1.2.0
- Thinking layer attributes are now explicit and parser-enforceable:
  - `render=false` — required; parsers reject `thinking`/`thinking_*` layers that omit it
  - `scale=N` — each symbol covers an N×N block of real pixels (default `1`)
  - `size=WxH` — grid dimensions in symbols; must match `sprite_size / scale` if provided
  - `purpose=<label>` — model-readable intent hint; ignored by parsers
- Removed the vague "may use a coarser grid" language — scale is now mandatory-explicit
- Validation: `size` mismatch with derived dimensions is an error

### 1.1.0
- Layer `order` attribute: explicit integer render order (bottom = low, top = high)
- Layer `blend` attribute: 8 compositing modes (`normal`, `multiply`, `screen`, `overlay`, `add`, `subtract`, `replace`, `erase`)
- Render order summary: deterministic bottom-up compositing pipeline
- Corrected safe symbol pool (removed ambiguous/non-ASCII characters: `# \ ~ £ € , ' "`)
- Validation rules 9–10: unique `order` values, valid `blend` values

### 1.0.0
- Initial specification
- Palette-index grid encoding
- Layer and frame support
- Chunk system with 32×32 maximum, spatial and semantic addressing
- Thinking layer (non-rendered model aid)
- Diff/patch tooling compatibility
