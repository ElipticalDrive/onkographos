# Shadow Resolve

## Overview
Shadow Resolve aggregates discrete shadow ray casting results into a continuous visibility value and an estimated penumbra width for each pixel [S1, unofficial]. It solves the problem of converting hard, per-sample occlusion hits from [Shadow Ray Casting](2796-shadow-ray-casting.md) into soft shadows suitable for spatial denoising [S1, unofficial]. Unlike sibling methods that compute visibility per-sample or use lookup tables, Shadow Resolve performs a post-processing reduction step that combines multiple samples into a single visibility scalar and a geometric penumbra metric [S1, unofficial].

## How it works
### Inputs and Outputs
**Inputs:**
*   `gShadowResults`: A `StructuredBuffer<TraceResult>` at register `t6` containing the raw results of shadow ray casts [S1, unofficial].
*   `gShadowMode`: A `uint` constant indicating the active shadowing mode [S1, unofficial].
*   `gShadowSamples`: A `uint` constant specifying the requested number of soft samples [S1, unofficial].
*   `gShadowStrength`: A `float` constant controlling the shadow intensity [S1, unofficial].
*   `gShadowSoftnessRadians`: A `float` constant defining the angular size of the light source for penumbra calculation [S1, unofficial].
*   `VoxelUnitsToMeters`: A conversion factor from voxel units to metres [S1, unofficial].

**Outputs:**
*   `gShadowVisibility`: An `RWStructuredBuffer<float>` at register `u2` storing the final visibility value per pixel [S1, unofficial].
*   `gShadowPenumbraMeters`: An `RWStructuredBuffer<float>` at register `u3` storing the estimated penumbra width in metres per pixel [S1, unofficial].

**Constants:**
*   `kMaximumShadowSamples`: A `uint` with value 16 [S1, unofficial].
*   `cbuffer ShadowResolveConstants` at register `b1` containing `uint gShadowPixelCount` and three padding `uint`s [S1, unofficial].

### Data Structures
The input buffer `gShadowResults` is indexed using `baseIndex = pixelIndex * kMaximumShadowSamples` [S1, unofficial]. The buffer layout is a flat array where each pixel's samples are stored contiguously [G]. Each `TraceResult` contains a `hit` field (`uint`, 1 if occluded, 0 otherwise) and a `distance` field (`float`, distance to the occluder along the ray) [S1, unofficial].

### Steps
1. Initialize `gShadowPenumbraMeters[pixelIndex] = 0.0F` [S1, unofficial].
2. If `gShadowMode == kShadowOff`, set `gShadowVisibility[pixelIndex] = 1.0F` and return [S1, unofficial].
3. If `gShadowMode == kShadowHard` or `gShadowMode == kShadowContact`, fetch `result = gShadowResults[baseIndex]`, compute `gShadowVisibility[pixelIndex] = VisibilityFromOcclusion(float(result.hit))`, and return [S1, unofficial].
4. For soft/hybrid modes, set `softOffset = 1` if `gShadowMode == kShadowHybrid`, else `0` [S1, unofficial].
5. Calculate `softSamples = min(max(gShadowSamples, 1u), kMaximumShadowSamples - softOffset)` [S1, unofficial].
6. Initialize `occluded = 0u`, `blockerDistance = 0.0F`, and `softOcclusion = 0.0F` [S1, unofficial].
7. Loop `sample` from `0` to `softSamples - 1`: fetch `result = gShadowResults[baseIndex + softOffset + sample]`, increment `occluded` by `result.hit`, and if `result.hit != 0u`, add `result.distance` to `blockerDistance` [S1, unofficial].
8. If `gShadowMode == kShadowHybrid`, update `softOcclusion = max(softOcclusion, float(gShadowResults[baseIndex].hit))` to include the contact shadow sample [S1, unofficial].
9. Compute the final occlusion ratio `softOcclusion = float(occluded) / float(softSamples)` [S1, unofficial].
10. Compute `gShadowVisibility[pixelIndex] = VisibilityFromOcclusion(softOcclusion)` [S1, unofficial].
11. If `occluded > 0`, compute `penumbraMeters = 2.0F * tan(gShadowSoftnessRadians) * VoxelUnitsToMeters(blockerDistance / float(occluded))` and store it in `gShadowPenumbraMeters[pixelIndex]` [S1, unofficial].

### Formulas
*   **Soft Occlusion Ratio:**
    $$ softOcclusion = \frac{occluded}{softSamples} $$
    Where `occluded` is the count of hit samples and `softSamples` is the number of soft samples processed [S1, unofficial].

*   **Penumbra Width (Meters):**
    $$ penumbraMeters = 2.0 \times \tan(gShadowSoftnessRadians) \times VoxelUnitsToMeters\left(\frac{blockerDistance}{occluded}\right) $$
    Calculated only if `occluded > 0` [S1, unofficial].

*   **Visibility from Occlusion:**
    $$ VisibilityFromOcclusion(occlusion) = \text{saturate}(1.0 - \text{saturate}(gShadowStrength) \times \text{saturate}(occlusion)) $$
    Where `saturate(x)` clamps `x` to `[0.0, 1.0]` [S1, unofficial].

### Parameters
*   `kMaximumShadowSamples`: 16 [S1, unofficial].
*   `gShadowSamples`: User-defined, capped by `kMaximumShadowSamples - softOffset` [S1, unofficial].
*   `gShadowStrength`: User-defined float [S1, unofficial].
*   `gShadowSoftnessRadians`: User-defined float [S1, unofficial].

### Edge Cases and Pitfalls
*   **Division by Zero:** The penumbra calculation divides by `occluded` [S1, unofficial]. The algorithm explicitly checks `if (occluded > 0)` before computing penumbra [S1, unofficial].
*   **Hybrid Mode Indexing:** In `kShadowHybrid` mode, the contact shadow sample is at `baseIndex`, while soft samples start at `baseIndex + 1` [S1, unofficial]. The `softOffset` variable handles this offset correctly [S1, unofficial].
*   **Sample Count Capping:** `softSamples` is capped at `kMaximumShadowSamples - softOffset` to prevent buffer overruns when `gShadowSamples` exceeds the maximum [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def shadow_resolve(pixel_index, g_shadow_results, g_shadow_mode, g_shadow_samples,
                   g_shadow_strength, g_shadow_softness_radians, voxel_units_to_meters):
    """
    Resolves shadow ray results into visibility and penumbra.
    
    Args:
        pixel_index (int): The current pixel index.
        g_shadow_results (list): List of TraceResult dicts with 'hit' (uint) and 'distance' (float).
        g_shadow_mode (int): Shadow mode (kShadowOff, kShadowHard, kShadowContact, kShadowHybrid).
        g_shadow_samples (int): Number of soft samples requested.
        g_shadow_strength (float): Shadow intensity.
        g_shadow_softness_radians (float): Light angular size.
        voxel_units_to_meters (float): Conversion factor.
        
    Returns:
        tuple: (visibility, penumbra_meters)
    """
    k_maximum_shadow_samples = 16
    k_shadow_off = 0
    k_shadow_hard = 1
    k_shadow_contact = 2
    k_shadow_hybrid = 3
    
    base_index = pixel_index * k_maximum_shadow_samples
    penumbra_meters = 0.0
    
    if g_shadow_mode == k_shadow_off:
        return 1.0, penumbra_meters
    
    if g_shadow_mode in (k_shadow_hard, k_shadow_contact):
        result = g_shadow_results[base_index]
        visibility = visibility_from_occlusion(float(result['hit']), g_shadow_strength)
        return visibility, penumbra_meters
    
    soft_offset = 1 if g_shadow_mode == k_shadow_hybrid else 0
    soft_samples = min(max(g_shadow_samples, 1), k_maximum_shadow_samples - soft_offset)
    
    occluded = 0
    blocker_distance = 0.0
    soft_occlusion = 0.0
    
    for sample in range(soft_samples):
        result = g_shadow_results[base_index + soft_offset + sample]
        occluded += result['hit']
        if result['hit'] != 0:
            blocker_distance += result['distance']
            
    if g_shadow_mode == k_shadow_hybrid:
        contact_result = g_shadow_results[base_index]
        soft_occlusion = max(soft_occlusion, float(contact_result['hit']))
        
    soft_occlusion_ratio = float(occluded) / float(soft_samples)
    visibility = visibility_from_occlusion(soft_occlusion_ratio, g_shadow_strength)
    
    if occluded > 0:
        avg_blocker_distance = blocker_distance / float(occluded)
        penumbra_meters = 2.0 * math.tan(g_shadow_softness_radians) * voxel_units_to_meters * avg_blocker_distance
        
    return visibility, penumbra_meters

def visibility_from_occlusion(occlusion, strength):
    """
    Computes visibility from occlusion ratio.
    """
    occlusion = max(0.0, min(1.0, occlusion))
    strength = max(0.0, min(1.0, strength))
    return max(0.0, min(1.0, 1.0 - strength * occlusion))
```

## Subtopics

### [Ambient Occlusion via Shadow Rays](878-ambient-occlusion-via-shadow-rays.md)
Approximates indirect illumination using hemisphere sampling [G]. Choose when a lightweight global illumination approximation is needed [G].

### [Direct Illumination with Shadow Rays](2011-direct-illumination-with-shadow-rays.md)
Computes direct light contribution per sample [G]. Choose for primary lighting calculations [G].

### [Lookup Table Shadow Rendering](3588-lookup-table-shadow-rendering.md)
Uses precomputed tables for shadow visibility [G]. Choose for performance-critical applications with limited shadow variations [G].

### [Recursive Ray Tracing](5600-recursive-ray-tracing.md)
Handles multiple bounces of light [G]. Choose for full global illumination [G].

### [Soft Shadows via Area Lights](7320-soft-shadows-via-area-lights.md)
Computes soft shadows using area light sampling [G]. Choose for physically accurate soft shadows without post-processing [G].

### [Bounded Directional Shadow Ray Generation](12387-bounded-directional-shadow-ray-generation.md)
Generates shadow rays within spatial bounds [G]. Choose for directional lights with specific spatial constraints [G].

### [Shadow Volume Atmospheric Scattering](13668-shadow-volume-atmospheric-scattering.md)
Integrates shadow volumes with atmospheric effects [G]. Choose for large-scale atmospheric rendering [G].

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of shadow samples per pixel, due to a single linear pass over the samples [G].
*   **Memory Complexity:** $O(1)$ per pixel for the reduction buffers `gShadowVisibility` and `gShadowPenumbraMeters` [S1, unofficial].
*   **Measured Performance:** Not given in the sources [S1, unofficial].
*   **Scaling:** Scales linearly with the number of shadow samples [G].
*   **Strengths:** Simple to implement, efficient reduction, provides penumbra data for denoising [S1, unofficial].
*   **Limitations:** Relies on pre-cast shadow rays and does not compute shadows itself [S1, unofficial].
*   **When to Use:** When shadow rays have already been cast and a soft visibility value is needed for shading or denoising [S1, unofficial].
*   **When Not to Use:** When real-time shadow ray casting is required or when penumbra data is not needed [G].

## Background
Shadow Resolve is part of the destructible voxel engine developed by HiroSakuraba [S1, unofficial]. It is designed to work with the engine's voxel-based shadow casting system, providing a bridge between raw ray cast results and the final shading pipeline [S1, unofficial].

## Sources
[S1] HiroSakuraba/destructible-voxel-engine, https://github.com/HiroSakuraba/destructible-voxel-engine, fetched 2026-10-28 (unofficial)
