# Tooltip Geometry Clipping Design

## Problem

The tooltip geometry builder currently subtracts the complete tooltip frame
from `MaxContentWidth`, even though the constant describes the maximum content
width. The default frame is 12 px on each side: 4 px required by vanilla's
`ActiveTip.DrawInner` contract plus 8 px of model padding. This reduces the
usable content width from 800 px to 776 px.

Pool fact grids remain non-wrapping and draw at their measured natural width.
Existing grids such as Meats and Meals need more than 776 px but no more than
the intended 800 px content limit. Their draw commands therefore extend past
the measured tooltip window, consuming the right padding and clipping the last
one or two count digits.

Wrapped description height has a separate state-dependent failure. `Compose`
calls `Text.CalcHeight`, whose result depends on the ambient
`Text.WordWrap` value. When measurement starts while wrapping is disabled, the
balanced tooltip caches a one-line height and later draws wrapped text into
that undersized rectangle.

Targeted runtime reproduction before the fix:

- The Meats pool shows a badge count of `1134`, while the rightmost tooltip
  value is visible only as `11` and has no right padding.
- The Meals pool shows a badge count of `820`, while the rightmost tooltip
  value is visible only as `82`, with the last pixels clipped.
- Resource tooltips backed by built-in item descriptions display only the
  beginning of the wrapped description.

## Goals

- Preserve the full 800 px content-width limit independently of padding.
- Add the complete frame symmetrically after content layout is finished.
- Measure wrapped prose deterministically regardless of ambient GUI state.
- Preserve the existing balance policy, pool column layout, cache ownership,
  invalidation dependencies, and steady render path.

## Non-goals

- Do not adaptively redistribute pool rows between columns.
- Do not truncate, wrap, or otherwise change non-wrapping pool cells.
- Do not change the behavior of a non-wrapping grid whose natural width itself
  exceeds the 800 px content limit. That requires a separate layout decision.
- Do not make padding participate in the aspect-ratio balancing calculation.
- Do not introduce a production seam or source-text test for Unity GUI state.

## Geometry pipeline

`WrTipUI.Ensure` remains the revision-gated geometry builder. On a cache miss it
will perform these operations in order:

1. Compute `frame = Pad + model.Padding`. With the defaults, `frame` is 12 px.
2. Compute the content limit as `min(maxWidth, MaxContentWidth)`. Do not
   subtract `frame` from this value.
3. Clamp the natural content width to that content limit.
4. Compose the content and apply the existing content-only balance policy. If
   balancing narrows the content, clear the first command set and recompose at
   the balanced content width.
5. Compute the final tooltip size by adding `frame * 2` independently to the
   final content width and content height, then round up to whole UI pixels.

For example, 792 px of natural content under the 800 px limit remains 792 px;
with a 12 px frame, its final window width is 816 px. Content wider than the
limit is capped at 800 px and produces an 824 px window.

Padding is therefore an outer presentation concern. It cannot reduce content
space or influence balancing.

## Wrapped text measurement

On a geometry cache miss, `WrTipUI.Ensure` will save the current font and
word-wrap state, set `Text.Font` to `GameFont.Small`, and set
`Text.WordWrap = true` before any natural-width or height measurement. Its
existing `finally` block will restore both values.

This work remains behind the existing model-width-UI-revision cache gate.
Cache hits and `WrTipUI.Draw` remain unchanged and allocation-free under the
existing render-path contract.

## Cache contract and lifecycle

The tooltip geometry cache keeps its existing contract:

- **Owner:** the immutable `TipModel`.
- **Key:** maximum content width and `UiVersion`.
- **Value:** positioned draw geometry and its final framed size.
- **Dependencies:** model content, font/UI metrics, and maximum content width.
- **Refresh policy:** immediate rebuild after width or UI metric revision
  changes; otherwise reuse.
- **Equality policy:** an equal key preserves cached geometry identity.
- **Teardown:** releasing the `TipModel` releases its geometry.

The fix changes width semantics, not dependencies, ownership, invalidation, or
teardown.

## Automated regression coverage

Add one parameterized executable Core test for the pure sizing policy:

- Natural width 792 px, maximum content width 800 px, frame 12 px produces an
  outer width of 816 px.
- Natural width 900 px, maximum content width 800 px, frame 12 px produces an
  outer width of 824 px.

The test starts from natural width, applies the content cap, and then applies
the symmetric frame. These two cases jointly prove that padding does not reduce
available content width and that the 800 px content cap remains effective.

The runtime-only word-wrap behavior will not receive a source-text test or an
artificial executable seam.

## Verification

Automated verification uses the repository's canonical commands:

```powershell
dotnet build -c Release src\EPrimeReadouts.sln --no-restore
dotnet test src\EPrimeReadouts.sln --no-restore
```

After deployment and a game restart, targeted runtime verification must confirm:

- Meats displays the complete `1134` value and visible right/bottom padding.
- Meals displays the complete `820` value and visible right/bottom padding.
- A resource tooltip with a built-in description displays every wrapped line.
- A repeated hover reuses the cached geometry without visual changes.

## Related engineering-contract update

The testing rule in `AGENTS.md` is revised so executable failing regressions are
required where a reasonable automated boundary exists. Runtime-only RimWorld
or Unity behavior may instead use documented reproduction and manual
verification. The same rule is propagated to WorkRoles and QualityCrafting.
