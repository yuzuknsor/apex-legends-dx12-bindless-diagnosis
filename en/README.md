# Apex Legends DX12 Bindless Frame Rate Regression

> [中文](../README.md) · [Full report](REPORT.md) · [报告全文（中文）](../REPORT.md)

A single-machine diagnostic record of a step-change frame rate drop in Apex Legends after the DX12 migration, and the three console variables that resolved it.

## The fix

Steam → Library → right-click Apex Legends → Properties → Launch Options:

```
+staticprop_color_use_bindless 0 +staticprop_depth_use_bindless 0 +r_drawworld_depth_bindless 0
```

## What it does

Apex's DX12 renderer uses **bindless resource binding** for the three geometry categories with the highest draw-call counts: static-prop colour, static-prop depth, and world-geometry depth. Together these cover essentially all opaque geometry submission.

On the affected system that path was pathological. Disabling it does not "add speed" — it removes an optimization that was costing roughly an order of magnitude, because the cost scaled with the number of visible primitives.

The three variables are engine console variables, not file modifications. EA support publishes this same combination as a troubleshooting step for DX12 issues, and provides `+rhi.dx12.EnableGPUUploadHeaps 0` as the next-level switch.

## Symptoms this addresses

- Frame rate far below expectation after a major patch, while hardware is healthy and other CPU-heavy titles run normally
- Deficit scales with scene complexity: worse in full matches than in the firing range
- Reinstalling GPU drivers, the game, or Windows does not help
- No TDR, no WHEA, no crashes — the machine is stable, just slow

If your deficit is a percentage rather than a multiple, this is probably not your problem. The report explains why that distinction matters.

## Files

| Path | Content |
|---|---|
| [`REPORT.md`](REPORT.md) | Full diagnostic report (English) |
| [`../REPORT.md`](../REPORT.md) | 诊断报告全文（中文） |
| [`DISCLAIMER.md`](DISCLAIMER.md) | Disclaimer (English) |
| [`../DISCLAIMER.md`](../DISCLAIMER.md) | 免责声明（中文） |
| [`../LICENSE`](../LICENSE) | CC0 1.0 |

## Verification

1. Record a baseline before changing anything (firing range and one match, 1% low).
2. Add the parameters, restart the game, re-measure.
3. Keep the baseline. If a future patch removes the non-bindless fallback path, these variables will silently stop working with no error message.

## License

[CC0 1.0](../LICENSE) — released into the public domain. Copy, modify, translate, redistribute, use commercially; no permission and no credit required.

Apex Legends and related marks are trademarks of Electronic Arts Inc. This repository is unofficial and is not affiliated with or endorsed by EA or Respawn Entertainment.
