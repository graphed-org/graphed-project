# Tracking: torch.jit deprecation in graphed-preserve's frozen m9 suite

**Status: OPEN (watch). Recorded 2026-06-10. No action until the trigger fires.**

## What

`graphed-preserve/tests/frozen/m9/test_ml_plugins.py` builds its TorchScript sample payloads with
`torch.jit.trace` → `torch.jit.save` → `torch.jit.load`. On torch 2.12 these emit 70
DeprecationWarnings per suite run ("switch to torch.compile or torch.export"). The production
package imports no torch — the M9 externals-plugin machinery is format-generic and the canonical
model-exchange format is ONNX (root prompt R10); TorchScript is a secondary sample payload kind.

## Why not fixed now

- The change lives in a FROZEN suite: fixing requires a human-directed freeze amendment (R0.6).
- `torch.export` produces a DIFFERENT artifact whose bytes content-hash differently, so the swap
  touches what the m9 plugin tests pin, not just a call site.
- PyTorch has kept deprecated `torch.jit` loading alive across many releases (serialized
  TorchScript exists in the wild); removal pressure is low.

## Trigger and action

When torch announces actual removal of `torch.jit`, or a CI matrix bump turns the warnings into
errors: perform a recorded freeze amendment (freeze tag bump + attempts.md entry, M10/UPROOT-1
precedent) replacing the fixture's TorchScript path with `torch.export` artifacts. Contained to
one file; expect new content hashes for the sample payloads.

## Companion note

graphed-awkward's single m17 RuntimeWarning (deliberate divide-by-zero proving `nan_to_num`) is
intentionally left in place — see `mvp-shortcomings.md` ("Warning watch").
