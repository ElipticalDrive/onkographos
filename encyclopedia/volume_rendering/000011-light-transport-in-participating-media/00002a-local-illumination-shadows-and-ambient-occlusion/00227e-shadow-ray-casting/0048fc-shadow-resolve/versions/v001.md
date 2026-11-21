# Shadow Resolve


## Overview
Shadow Resolve aggregates discrete shadow ray casting results into a continuous visibility value and penumbra width for spatial denoising [S1, unofficial]. It differs from sibling methods by operating as a post-processing or finalization step that converts binary hit/miss data from multiple samples into soft shadow metrics, rather than performing the ray traversal itself [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** `gShadowResults`, a `StructuredBuffer<TraceResult>` at register `t6` [S1, unofficial]. Each `TraceResult` contains `hit` (uint) and `distance` (float) [S1, unofficial].
*   **Output:** `gShadowVisibility`, an `RWStructuredBuffer<float>` at register `u2` [S1, unofficial]. `gShadowPenumbraMeters`, an `RWStructuredBuffer<float>` at register `u3` [S1, unofficial].
*   **Constants:** `cbuffer ShadowResolveConstants` at register `b1` containing `uint gShadowPixelCount` and three padding uints [S1, unofficial].

**Data Structures:**
*   `TraceResult`: Contains `hit` (uint, 1 if occluded, 0 otherwise) and `distance` (float, distance to blocker) [S1, unofficial].
*   `kMaximumShadowSamples`: Constant value 16 [S1, unofficial].

**Steps:**
1.  Initialize `gShadowPenumbraMeters[pixelIndex] = 0.0F` [S1, unofficial].
2.  Check `gShadowMode`. If `kShadowOff`, set `gShadowVisibility[pixelIndex] = 1.0F` and return [S1, unofficial].
3.  If `gShadowMode` is `kShadowHard` or `kShadowContact`:
    *   Compute `baseIndex = pixelIndex * kMaximumShadowSamples` [S1, unofficial].
    *   Fetch `result = gShadowResults[baseIndex].hit` [S1, unofficial].
    *   Set `gShadowVisibility[pixelIndex] = VisibilityFromOcclusion(float(result))` [S1, unofficial].
    *   Return [S1, unofficial].
4.  For soft/hybrid modes:
    *   Set `softOffset = 1` if `kShadowHybrid`, else `0` [S1, unofficial].
    *   Compute `softSamples = min(max(gShadowSamples, 1u), kMaximumShadowSamples - softOffset)` [S1, unofficial].
    *   Initialize `occluded = 0u` and `blockerDistance = 0.0F` [S1, unofficial].
    *   Initialize `softOcclusion = 0.0F` [G] (implied by usage in step 5).
5.  Loop `sample` from `0` to `softSamples - 1`:
    *   Fetch `result = gShadowResults[baseIndex + softOffset + sample]` [S1, unofficial].
    *   Increment `occluded` by `result.hit` [S1, unofficial].
    *   If `result.hit != 0u`, add `result.distance` to `blockerDistance` [S1, unofficial].
6.  If `gShadowMode == kShadowHybrid`:
    *   Fetch `contactResult = gShadowResults[baseIndex].hit` [S1, unofficial].
    *   Update `softOcclusion = max(softOcclusion, float(contactResult))` [S1, unofficial].
7.  Compute `softOcclusion = occluded / softSamples` [S1, unofficial].
8.  Compute `gShadowVisibility[pixelIndex] = VisibilityFromOcclusion(softOcclusion)` [S1, unofficial].
9.  If `occluded > 0`:
    *   Compute `penumbraMeters = 2.0 * tan(gShadowSoftnessRadians) * VoxelUnitsToMeters(blockerDistance / occluded)` [S1, unofficial].
    *   Set `gShadowPenumbraMeters[pixelIndex] = penumbraMeters` [S1, unofficial].

**Formulas:**
*   Soft Occlusion:
    $$ softOcclusion = \frac{occluded}{softSamples} $$
    where `occluded` is the count of hit samples and `softSamples` is the number of soft samples processed [S1, unofficial].

*   Penumbra Width:
    $$ penumbraMeters = 2.0 \times \tan(gShadowSoftnessRadians) \times VoxelUnitsToMeters\left(\frac{blockerDistance}{occluded}\right) $$
    Calculated only if `occluded > 0` [S1, unofficial]. `blockerDistance` is the sum of hit distances, `occluded` is the count of hits [S1, unofficial].

*   Visibility from Occlusion:
    $$ VisibilityFromOcclusion(occlusion) = \text{saturate}(1.0 - \text{saturate}(gShadowStrength) \times \text{saturate}(occlusion)) $$
    [S1, unofficial].

**Parameters:**
*   `kMaximumShadowSamples`: 16 [S1, unofficial].
*   `gShadowSamples`: Number of soft samples per pixel [S1, unofficial].
*   `gShadowSoftnessRadians`: Softness angle in radians [S1, unofficial].
*   `gShadowStrength`: Shadow intensity factor [S1, unofficial].
*   `VoxelUnitsToMeters`: Conversion factor from voxel units to meters [S1, unofficial].

**Edge Cases and Pitfalls:**
*   Division by zero: `blockerDistance / occluded` is guarded by `if (occluded > 0)` [S1, unofficial].
*   `softSamples` calculation ensures at least 1 sample is used if `gShadowSamples` is 0, but clamped by `kMaximumShadowSamples` [S1, unofficial].
*   Hybrid mode requires careful ordering of contact shadow sample vs soft samples to ensure `softOcclusion` correctly incorporates the hard contact shadow [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** O(N) where N is the number of shadow samples per pixel. The loop runs `softSamples` times.
*   **Memory Complexity:** O(1) per pixel for the resolve step, assuming input buffers are already allocated.
*   **Strengths:** Provides continuous visibility and penumbra data suitable for spatial denoising [S1, unofficial]. Supports hard, soft, and hybrid shadow modes [S1, unofficial].
*   **Limitations:** Relies on pre-cast shadow rays; does not perform ray traversal itself [S1, unofficial]. Precision depends on the accuracy of `TraceResult` distances [S1, unofficial].
*   **When to use:** When soft shadows are required and spatial denoising is planned [S1, unofficial].
*   **When not to use:** When hard shadows are sufficient and performance is critical, as hard mode skips the soft sample loop [S1, unofficial].

## Background
Shadow Resolve is part of the destructible voxel engine implementation by HiroSakuraba [S1, unofficial]. It is designed to work with the Shadow Ray Casting family of methods, specifically aggregating results from multiple shadow rays to produce soft shadows [S1, unofficial].

## Sources
[S1] HiroSakuraba/destructible-voxel-engine, https://github.com/HiroSakuraba/destructible-voxel-engine, fetched 2026-10-28 (unofficial)
