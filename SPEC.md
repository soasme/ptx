# PTX — Pixel Text Exchange Format
**Version 1.4.5**

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
[frame <name> duration=<ms>]   # one per frame — omit for static art
[animation <name>]             # omit to use implicit default animation
[layer <name>]
[chunk <id> ...]
```

---

## `[meta]`

Describes the overall sprite.

```
[meta]
width 128
height 128
bits_per_pixel 32       # 8 (indexed) | 16 (grayscale) | 32 (rgba)
type animated           # static | animated
tile_size 32            # chunk grid size; must be 1–32 (default 32)
background transparent
```

| Key | Values | Default |
|---|---|---|
| `width` | integer ≥ 1, sprite width in pixels | required |
| `height` | integer ≥ 1, sprite height in pixels | required |
| `bits_per_pixel` | `8` (indexed), `16` (grayscale), `32` (rgba) | `32` |
| `type` | `static` \| `animated` | `static` |
| `tile_size` | integer 1–32 | `32` |
| `background` | color | `transparent` |

---

## `[palette]`

Maps single-character symbols to colors. Symbols are palette indices — a symbol means a color only because the palette says so.

```
[palette]
. transparent
K #000000ff
W #ffffffff
R #ff3b30ff
r #9f1f1fff
+ #ffd166ff
@ #5c7cfaff
```

### Color format

A color value is one of three forms:

**1. 32-bit RGBA hex** — 8 lowercase hex digits, alpha explicit:
```
#rrggbbaa
```
Regex: `^#[0-9a-f]{8}$`

**2. 24-bit RGB hex** — 6 lowercase hex digits, alpha implicitly `ff` (fully opaque):
```
#rrggbb
```
Regex: `^#[0-9a-f]{6}$`

**3. Named color** — a lowercase CSS color name:
```
transparent
green
cyan
cornflowerblue
```

Named colors resolve to their CSS Color Level 4 `#rrggbbaa` equivalents. `transparent` is defined as `#00000000`. All other named colors have alpha `ff` (fully opaque) unless the name itself implies transparency.

Uppercase hex letters are invalid in both hex forms. Named colors must be lowercase.

**Examples:**

| Written | Resolves to | Meaning |
|---|---|---|
| `#40e0d0ff` | `#40e0d0ff` | Fully opaque turquoise |
| `#40e0d0` | `#40e0d0ff` | Same — alpha inferred as `ff` |
| `#000000ff` | `#000000ff` | Fully opaque black |
| `#ff000080` | `#ff000080` | 50% transparent red |
| `#00000000` | `#00000000` | Fully transparent |
| `transparent` | `#00000000` | Alias |
| `cyan` | `#00ffffff ff` | CSS named color |
| `green` | `#008000ff` | CSS named color |

Parsers must normalize all color values to `#rrggbbaa` internally.

**Rules:**
- One entry per line: `<symbol> <color>`
- `<color>` is a color value: `#rrggbbaa`, `#rrggbb`, or a lowercase CSS named color
- `.` (dot) conventionally means `transparent`; redefining it is allowed
- Symbol pool: `. a-z A-Z 0-9 @ % ^ * + = _ | ~ < > [ ] { } ? ! - / : ; ( ) $ &`
  This gives up to 84 distinct colors per file
- Explicitly excluded: `#` (comment marker), `\` (reserved for escaping), `"` and `'` (quoting characters) — using any of these as a palette symbol is a validation error

---

## `[frame <name> duration=<ms>]`

Declares a named frame and its display duration. Required for animated sprites; omit for static art. One header per frame.

```
[frame idle_1 duration=120]
[frame walk_1 duration=90]
[frame walk_2 duration=90]
```

`duration` is an integer number of milliseconds.

---

## `[animation <name>]`

Declares a named animation — an ordered sequence of frames with a loop configuration.

```
[animation walk]
frames walk_1,walk_2,walk_3
loop_type forward
loop_count +inf

[animation run_burst]
frames run_1,run_2,run_3
loop_type ping_pong
loop_count 3
```

| Field | Values | Default |
|---|---|---|
| `frames` | Comma-separated list of frame names defining playback order | required |
| `loop_type` | `forward` \| `reverse` \| `ping_pong` \| `ping_pong_reverse` | `forward` |
| `loop_count` | Positive integer or `+inf` | `+inf` |

### Loop types

| Mode | Playback |
|---|---|
| `forward` | Plays frames in the listed order, then repeats from the first frame. |
| `reverse` | Plays frames in reverse order, then repeats from the last frame. |
| `ping_pong` | Plays forward then backward; endpoint frames are not duplicated at the reversal. For frames A, B, C: plays A B C B A B C B … |
| `ping_pong_reverse` | Like `ping_pong` but starts with the backward pass. For frames A, B, C: plays C B A B C B A B … |

`loop_count` is the number of full cycles before the animation stops. `+inf` means the animation loops forever. A cycle for `ping_pong` and `ping_pong_reverse` is one complete forward-and-back pass.

### Default animation

If no `[animation]` section is present, an implicit animation named `default` is assumed:

```
[animation default]
frames <all [frame] entries in declaration order>
loop_type forward
loop_count +inf
```

Files with no `[frame]` entries (static art) are unaffected — the implicit `default` animation has no frames and is never played.

---

## `[layer <name>]`

Declares a named layer. Layer content is provided by the chunks that reference it.

### Layer attributes

```
[layer <name> order=<n> type=<type> blend=<mode> opacity=<f> visible=<bool>]
```

| Attribute | Description | Default |
|---|---|---|
| `order` | Render order integer. Lower numbers render first (bottom). Higher numbers render on top. | Declaration order (0-based) |
| `type` | Layer type: `normal`, `group`, or `tilemap`. | `normal` |
| `blend` | Compositing mode for this layer onto the result below it. Valid for `normal` and `tilemap` layers. | `normal` |
| `opacity` | Layer opacity, `0.0`–`1.0`. Valid for `normal` and `tilemap` layers. | `1.0` |
| `visible` | Whether the layer is included in rendering. `true` or `false`. | `true` |
| `frame` | (body line) Restrict this layer to specific frames. Repeatable. | all frames |

**`order`** is the authoritative render sequence. Declaration order in the file is the fallback when `order` is omitted. Explicit `order` values need not be contiguous — using multiples of 10 leaves room for insertions without renumbering.

```
[layer shadow  order=10 blend=multiply]
[layer fill    order=20 blend=normal]
[layer outline order=30 blend=normal]
[layer light   order=40 blend=screen]
[layer fx      order=50 blend=addition]
```

Layers are composited bottom-up: `shadow` → `fill` → `outline` → `light` → `fx`.

**`visible`** controls whether a layer participates in rendering at all. A hidden layer (`visible=false`) is preserved in the file — its chunks remain intact — but is skipped during compositing, the same as if the layer did not exist. This is useful for work-in-progress layers, reference layers, and toggling detail passes without deleting data.

```
[layer sketch order=5 blend=normal visible=false]
[layer shadow order=10 blend=multiply]
[layer fill   order=20 blend=normal]
```

Here `sketch` is kept in the file for reference but never renders.

### Blend modes

Blend modes apply only to `normal` and `tilemap` layers. Transparent pixels (`transparent` / `#00000000`) in any layer are always a no-op regardless of blend mode.

| Mode | Description |
|---|---|
| `normal` | Standard alpha compositing (Porter-Duff over). Default. |
| `multiply` | Multiplies pixel values. Darkens; transparent pixels pass through. |
| `screen` | Inverse multiply. Brightens; good for light/glow effects. |
| `overlay` | Multiply where base is dark, screen where base is light. |
| `darken` | Keeps the darker of the two pixels per channel. |
| `lighten` | Keeps the lighter of the two pixels per channel. |
| `color_dodge` | Brightens the base by dividing by the inverse of the layer. |
| `color_burn` | Darkens the base by dividing and inverting. |
| `hard_light` | Multiply/screen based on the layer color rather than the base. |
| `soft_light` | Softer version of hard light; subtle contrast adjustment. |
| `difference` | Absolute difference between layer and base. |
| `exclusion` | Similar to difference but lower contrast. |
| `hue` | Hue of layer with saturation and luminosity of base. |
| `saturation` | Saturation of layer with hue and luminosity of base. |
| `color` | Hue and saturation of layer with luminosity of base. |
| `luminosity` | Luminosity of layer with hue and saturation of base. |
| `addition` | Additive blend. Pixels add up; clamps to white. Good for fire/sparks. |
| `subtract` | Subtracts layer from base. Clamps to black. |
| `divide` | Divides base by layer per channel. Brightens. |

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
3. Skip layers where `visible=false`
4. For each remaining layer (bottom to top): composite its chunks onto the canvas using `blend`
5. Thinking layers are skipped entirely
6. Output the final composited canvas per frame

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
width 128
height 128
bits_per_pixel 32
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

[frame idle_1 duration=120]
[frame walk_1 duration=90]
[frame walk_2 duration=90]

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
4. Chunk coordinates must not exceed the sprite `width` × `height`
5. Chunks must not overlap (within the same layer and frame)
6. `tile_size` must be between 1 and 32 inclusive
7. Thinking layers must not be referenced by `[chunk]` entries that lack `layer=thinking`
8. Frame names referenced in chunks must be declared as `[frame ...]` headers
9. Two layers may not share the same `order` value
10. `blend` must be one of: `normal multiply screen overlay darken lighten color_dodge color_burn hard_light soft_light difference exclusion hue saturation color luminosity addition subtract divide`
11. `visible` must be `true` or `false` if present
12. `type` must be `normal`, `group`, or `tilemap` if present
13. `opacity` must be a float in the range `0.0`–`1.0` if present
14. `blend` and `opacity` are valid only on `normal` and `tilemap` layers; specifying either on a `group` layer is a validation error
15. Every `<color>` value must match `^#[0-9a-f]{8}$`, `^#[0-9a-f]{6}$`, or be a lowercase CSS named color
16. `bits_per_pixel` must be `8`, `16`, or `32` if present
17. Palette symbols must not be `#`, `\`, `"`, or `'`
18. Every frame name in an `[animation]` `frames` list must be declared as a `[frame ...]` header
19. `loop_type` must be `forward`, `reverse`, `ping_pong`, or `ping_pong_reverse` if present
20. `loop_count` must be a positive integer or `+inf` if present
21. Animation names must be unique across the file

---

## MIME Type and Extension

| | |
|---|---|
| Extension | `.ptx` |
| MIME type | `text/x-ptx` |
| Encoding | UTF-8 |

---

## Changelog

See [CHANGELOG.md](CHANGELOG.md) for the full version history.
