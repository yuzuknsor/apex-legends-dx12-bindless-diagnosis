# Apex Legends Frame Rate Regression — Diagnostic Report

> [中文](../REPORT.md) · [README](README.md)

| | |
|---|---|
| **Closed** | 2026-10-29 |
| **Test system** | i7-14700KF · RTX 4080 SUPER 16 GB · ROG STRIX B760-G · 2×16 GB DDR5-6000 (XMP) |
| **Operating system** | Windows 11 Pro for Workstations · Insider Preview · Dev channel · **29695.1001** · x64 |
| **Outcome** | Resolved. No hardware fault. Single root cause. No OS reinstall required. |

> This is a single-system technical case record, not official documentation. It is not affiliated with, endorsed by, or authorized by Electronic Arts, Respawn Entertainment, Microsoft, or NVIDIA. See [DISCLAIMER.md](DISCLAIMER.md) for the full disclaimer.

---

## 0. Summary

The original assessment — that performance degradation was caused by a combination of environmental factors — does not hold. The actual cause is a single defective rendering path:

`DX12 migration → bindless resource binding enabled for the three highest-primitive-count geometry categories → degraded driver / OS interface behavior on this system → additional per-primitive submission cost → frame rate inversely proportional to visible primitive count → below 30 fps in match`

**Remediation**: disable that rendering path via Steam launch options. Three console variables; no game files modified, no kernel-level components, no ban exposure.

```
+staticprop_color_use_bindless 0 +staticprop_depth_use_bindless 0 +r_drawworld_depth_bindless 0
```

**Three quantitative facts**

1. The performance deficit is approximately **8×** (below 30 fps in match, against 240+ fps expected for comparable hardware). A deficit of this magnitude indicates a *different code path being executed*, not a fixed proportional overhead. This criterion alone can collapse the hypothesis space early in diagnosis.
2. Five days of system-level work — including a full OS rebuild and a game reinstall — produced a marginal improvement of approximately zero. The symptom persisted after the rebuild, which constitutes a natural-experiment refutation of every environment-class hypothesis.
3. The launch parameters carry a **silent-failure** risk. This is the only regression point that requires long-term monitoring.

**Scope**: this document records observations from one machine and is not a general conclusion. The diagnostic method may be applicable to similar symptoms (a step-change in frame rate after a major patch, with hardware and other titles unaffected); behaviour may differ across hardware and software combinations.

---

## 1. Environment Baseline (measured 2026-10-29)

| Layer | Item | Measured value |
|---|---|---|
| Compute | CPU | i7-14700KF (no throttling, no WHEA records) |
| Memory | Capacity / frequency | 31.8 GB usable · DDR5-6000 dual channel, XMP enabled |
| Graphics | GPU | RTX 4080 SUPER, 16376 MiB, VBIOS 95.03.44.00.e0 |
| Graphics | Driver | NVIDIA **621.01** |
| System | OS | Windows 11 Pro for Workstations · Insider **Dev** channel (**29695.1001**); previously an earlier preview build |
| System | Virtualization security | VBS running (status 2); **HVCI disabled** (SecurityServicesRunning = 0) |
| Anti-cheat | Javelin | EAAntiCheat and EAJavelinInstaller (updated 10-21 / 10-23) |
| Game | Version | v3.0.3.65 · `R5pc_r5-301_J65_CL11630615_FSv30` (build of 2026-10-02) |
| Game | Renderer | **`r5apex_dx12.exe` only — no DX11 executable present in the install** |
| Game | Install path | `E:\<path>\steamapps\common\Apex Legends` |
| Game | Video settings | 2560×1440 fullscreen borderless · VSync off · **shadows off** · SSAO 0 · volumetric lighting and fog off · particle CPU level 0 · texture streaming budget 3 GB |
| Environment | Startup items | 15 (messaging, file sync, self-built keep-alive script, screenshot tool, security software) |

> Note 1: the video settings are a key piece of corroborating evidence. `shadow_enable 0` means the shadow-map stage had already been disabled, so the remaining depth passes consisted only of the world-geometry and static-prop depth prepass — precisely the paths controlled by the two `_depth_bindless` variables.
>
> Note 2: all frame rates in this document are uncontrolled observations from a single machine. They are not benchmark results and are not intended for product-to-product comparison.

---

## 2. Symptoms and Quantitative Characteristics

| Scenario | Measured | Expected for this hardware | Deficit | Note |
|---|---|---|---|---|
| Firing range | < 60 | 300+ | ~5× | Smaller map, fewer visible props |
| In match | < 30 | 240+ | ~8× | Full map, highest visible prop and world-surface count |
| CS2 / Valorant | Normal | — | 0 | Also CPU-intensive; rules out a system-level bottleneck |
| Stability | 0 TDR / 0 WHEA | — | — | No hardware or driver instability |

**Quantitative characteristic**: the deficit scales with the number of visible primitives. This is the signature of a **per-primitive overhead**.

Fixed-overhead factors (HVCI 5–25%, background load, DPC spikes) produce only an approximately constant proportional loss and are structurally incapable of producing an 8× deficit that scales with scene complexity. On this criterion alone, items 3–5 in the ranking of Section 3 can be eliminated before any testing is performed.

---

## 3. Root Cause

### 3.1 Location: three bindless paths

Bindless means placing thousands of shader resource views (SRVs) in a single large descriptor heap and letting shaders index into it directly, eliminating the root descriptor table binding previously required per draw call. In its 9-29 DX12 migration, Apex enabled this mechanism only for the three geometry categories with the highest draw-call counts.

| Console variable | Path disabled | Per-frame magnitude |
|---|---|---|
| `staticprop_color_use_bindless` | Bindless sampling in the colour pass for static props (buildings, rocks, crates, vegetation) | Highest material variety and draw-call count |
| `staticprop_depth_use_bindless` | Depth prepass for the same props | No culling exemption; always executed every frame |
| `r_drawworld_depth_bindless` | Depth pass for world geometry (brushes, displacements) | Coverage equals the entire visible world |

The union of the three covers **all opaque geometry submission**. The remediation therefore bypasses the entire rendering branch rather than mitigating it — which is why the symptom disappeared completely rather than improved partially.

### 3.2 Why disabling the optimization improves performance by ~8×

Bindless is an **optional** path layered on top of a correct baseline. The engine must retain both implementations, otherwise these variables would not exist. Disabling the path is not a repair; it removes an optimization that yields a strongly negative return on this particular system.

Possible mechanisms (identifying the specific one is unnecessary — all three present identically):

1. **Resource residency constraint failure** — bindless requires the entire batch of textures to be resident in the same descriptor heap simultaneously. When the OS memory manager relocates or evicts them, the runtime can only insert fix-up operations between draw calls or wait on a copy-queue fence, leaving the GPU idle.
2. **Driver implements descriptor indexing in software** — the capability bit is reported, but the implementation performs indirect table lookups, costing more than conventional binding.
3. **Incorrect heap capacity or tier determination** — overflow into a fallback path that copies descriptors per draw call, strictly slower than the classic path.

**Corroboration**: the next-level switch published by EA is `+rhi.dx12.EnableGPUUploadHeaps 0`, which places the defect in the "GPU-visible memory and descriptor heap" chain rather than in the rendering algorithm itself.

### 3.3 Why falling back to DX11 was not an option

No `r5apex.exe` (DX11 executable) remains in the install directory; the DX11 fallback path no longer exists in this version. The console variables are the only targeted remediation: retain the remaining benefits of DX12 while disabling the one defective branch.

### 3.4 Why reinstalling the OS did not help

The symptom persisted after a full OS rebuild on 10-25 (updated to build 29695.1001) and only disappeared once the launch parameters were added. This refutes every "system state" hypothesis: the OS instance was replaced, configuration was reset, caches were cleared, and the defect remained unchanged — therefore the defect is on the engine side, not the environment side. This is a natural experiment whose evidential weight exceeds that of any single measurement.

### 3.5 Corrections to the original root-cause ranking

| Original ranking | Correction |
|---|---|
| #1 Javelin plus known DX12 regression (high) | Correct direction, **granularity too coarse**. The actionable form is "one specific bindless sub-path", not "DX12 is slow". Coarse granularity leads directly to reinstalling the OS, a hypothesis now falsified |
| #2 Insider preview build (high) | Downgraded to a **severity multiplier**. The same parameter set resolves a crash (`0x887A0006`) on standard Windows 11 with an RX 9070 XT, while on this system it manifests as a performance drop. The defect is on the Apex side; the environment only determines whether it presents as a crash or as a frame-rate drop |
| #3 VBS and HVCI (medium) | **Magnitude mismatch**: a 5–25% overhead cannot explain an 8× deficit |
| #4 Conflicting software and DPC (medium) | **Magnitude mismatch**: DPC produces transient spikes, not an 8× downward shift in the baseline |
| #5 Background load (low to medium) | As above, and the symptom did not change after cleanup, which is a counter-proof |

---

## 4. Remediation Register

### L1 · Effective (retained)

| Action | Effect |
|---|---|
| Three bindless variables injected via Steam launch options (confirmed written to `localconfig.vdf`, appid 1172470) | Full recovery in both the firing range and in match |

### L2 · Executed, ineffective against this symptom, but system state was changed (recorded, not reverted)

- **System layer**: HVCI disabled (VBS still running); startup items reduced from 20+ to 15; approximately 20 third-party services set to manual start (motherboard lighting and overclocking suites, storage management tools, virtual audio and virtual display devices, vendor telemetry services, network filter drivers, game platform background services); GameDVR and background recording disabled.
- **Game layer**: three shader caches cleared (in-game PSO cache, NVIDIA DXCache, D3DSCache); Steam file integrity verification; game reinstalled.
- **Software layer**: tablet driver suite and a collaboration client uninstalled.
- **OS rebuild**: updated to build 29695.1001; `Windows.old` and the previous user directory retained.

None of these changes had a negative effect and the system has been stable since; they are not reverted. They are recorded because otherwise any future performance comparison loses its baseline.

### L3 · Incident (recovered)

On 10-22 four work services were stopped in error (instant messaging, file sync, voice tool, WSL subsystem). All were restored and verified on 10-23, and the self-built auto-recovery scripts remained intact. One outstanding item: a shell startup shortcut needs to be placed back into `shell:common startup` manually.

---

## 5. Excluded Factors (measured, not inferred)

GPU throttling and hardware fault (0 TDR, 0 WHEA) · memory XMP stability · VRAM capacity · drive health · power plan (minimum and maximum processor state both 100%) · forced HPET · GPU MSI interrupt mode · corrupted game configuration · anti-cheat crashes · DPC latency as the dominant factor · background load as the dominant factor · **system state (falsified by the full OS rebuild)**

---

## 6. Risk and Regression Monitoring

| # | Item | Action |
|---|---|---|
| 1 | **Baseline data missing** (the only substantive gap) | Record 1% low frame rate for the firing range and for a match, one set each |
| 2 | **Silent failure**: if a future patch removes the non-bindless fallback path, the three variables will have no effect and will not report an error | Re-test against the baseline after every major update |
| 3 | Residual stutter | Next-level action: `+rhi.dx12.EnableGPUUploadHeaps 0` |
| 4 | After an official bindless fix | Removing the parameters recovers a few percent of CPU submission cost; worthwhile only when targeting 300+ fps |
| 5 | Ban exposure | Pure engine console variables injected through the official launch-option interface; no memory modification; no ban exposure |
| 6 | System remains on the Insider Dev channel | If similar issues recur, suspect the rendering path before the OS version |

---

## 7. Methodology

1. **Constrain the hypothesis space by the magnitude of the deficit.**
   An 8× deficit does not belong to the fixed-overhead category; it corresponds to a different code path. Fixed-overhead factors can explain percentage-level differences but never multiples. This criterion eliminates a large set of candidate factors in advance.

2. **The observation dimension defines the observability boundary.**
   TDR/WHEA, LatencyMon, service lists and power plans all sit at the OS and process layer. Bindless is an engine-internal resource-binding decision made per draw call; those tools produce no valid signal in that dimension. A dimension that is not instrumented cannot be characterised by collecting more samples.

3. **Change cost must be counted as diagnostic cost.**
   The cost of the "system hygiene" effort in this case: five days, one full OS rebuild, and four work services stopped in error. Once the magnitude of the deficit points to a code path, that expenditure is avoidable.

---

## 8. Open Items

- [ ] Record baseline data (1% low frame rate for firing range and for a match)
- [ ] Track EA's bindless fix and evaluate when to remove the parameters
- [ ] Clean up the previous user directory and `Windows.old` once no rollback is needed

---

## Appendix · Terminology

| Chinese | English |
|---|---|
| 资源绑定 | resource binding |
| 描述符堆 | descriptor heap |
| 根描述符表 | root descriptor table |
| 着色器资源视图 | Shader Resource View (SRV) |
| 深度预通道 | depth prepass |
| 静态道具 | static prop |
| 不透明几何提交 | opaque geometry submission |
| 绘制调用 | draw call |
| 资源常驻 | resource residency |
| 逐图元开销 | per-primitive overhead |
| 性能缺口 | performance deficit |
| 固定开销型因素 | fixed-overhead factor |
| 静默失效 | silent failure |
