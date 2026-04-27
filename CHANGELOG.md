# PTX Changelog

All notable changes to the PTX specification are documented here.

---

## 1.4.2

- Color type now supports three forms: `#rrggbbaa` (32-bit RGBA), `#rrggbb` (24-bit RGB, alpha inferred as `ff`), and lowercase CSS named colors (e.g. `transparent`, `green`, `cyan`).
- Named colors resolve to CSS Color Level 4 `#rrggbbaa` equivalents; `transparent` = `#00000000`.
- `[meta] background` now typed as color (was a freeform string).
- Parsers must normalize all color forms to `#rrggbbaa` internally.
- Validation rule 15 updated: accepts all three color forms.

## 1.4.1

- Color format is now strictly defined as 32-bit RGBA: `#rrggbbaa`, 8 lowercase hex digits.
- **Regex:** `^#[0-9a-f]{8}$` — uppercase letters are invalid.
- `transparent` is a canonical alias for `#00000000`. Valid anywhere a color is expected.
- Removed shorthand `#rrggbb` (6-digit) support — all colors must include the alpha channel explicitly.
- All palette examples updated to 8-digit form (e.g. `#000000ff`).
- Validation rule 15: every color value must match `^#[0-9a-f]{8}$` or be `transparent`.

## 1.4.0

- Frames are now declared as individual `[frame <name> duration=<ms>]` headers instead of rows inside a `[frames]` block. Duration is an integer number of milliseconds (e.g. `duration=120`).
- Layer `type` attribute: `normal` (default), `group`, or `tilemap`.
- Layer `opacity` attribute: float `0.0`–`1.0` (default `1.0`).
- `blend` and `opacity` are valid only on `normal` and `tilemap` layers; specifying either on a `group` layer is a validation error.
- Blend modes expanded from 8 to 19: added `darken`, `lighten`, `color_dodge`, `color_burn`, `hard_light`, `soft_light`, `difference`, `exclusion`, `hue`, `saturation`, `color`, `luminosity`, `addition`, `divide`; renamed `add` → `addition`; removed `replace` and `erase`.
- Validation rules 12–14: layer `type` values, `opacity` range, and blend/opacity restriction on group layers.

## 1.3.0

- Layer `visible` attribute: `true` (default) or `false`. Hidden layers are skipped in compositing but preserved in the file — chunks remain intact for reference, WIP, or toggling detail passes.
- Render order summary updated: step 3 now explicitly skips `visible=false` layers.
- Validation rule 11: `visible` must be `true` or `false` if present.
- Changelog moved to dedicated `CHANGELOG.md`.

## 1.2.0

- Thinking layer attributes are now explicit and parser-enforceable:
  - `render=false` — required; parsers reject `thinking`/`thinking_*` layers that omit it
  - `scale=N` — each symbol covers an N×N block of real pixels (default `1`)
  - `size=WxH` — grid dimensions in symbols; must match `sprite_size / scale` if provided
  - `purpose=<label>` — model-readable intent hint; ignored by parsers
- Removed the vague "may use a coarser grid" language — scale is now mandatory-explicit
- Validation: `size` mismatch with derived dimensions is an error

## 1.1.0

- Layer `order` attribute: explicit integer render order (bottom = low, top = high)
- Layer `blend` attribute: 8 compositing modes (`normal`, `multiply`, `screen`, `overlay`, `add`, `subtract`, `replace`, `erase`)
- Render order summary: deterministic bottom-up compositing pipeline
- Corrected safe symbol pool (removed ambiguous/non-ASCII characters: `# \ ~ £ € , ' "`)
- Validation rules 9–10: unique `order` values, valid `blend` values

## 1.0.0

- Initial specification
- Palette-index grid encoding
- Layer and frame support
- Chunk system with 32×32 maximum, spatial and semantic addressing
- Thinking layer (non-rendered model aid)
- Diff/patch tooling compatibility
