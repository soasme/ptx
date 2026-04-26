# PTX Changelog

All notable changes to the PTX specification are documented here.

---

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
