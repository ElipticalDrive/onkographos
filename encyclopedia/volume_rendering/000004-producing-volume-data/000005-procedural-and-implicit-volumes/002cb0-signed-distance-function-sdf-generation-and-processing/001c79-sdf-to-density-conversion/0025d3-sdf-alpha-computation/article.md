# SDF Alpha Computation
## Overview
Computes the output alpha for a Signed Distance Field (SDF) given the Screen Space Ratio (SSR), Signed Distance (SD), Signed Distance Ratio (SDR), isoPerimeter (dilation/contraction), and softness [S1, unofficial]. It converts a signed distance value into a density value using alpha and beta parameters, building on [SDF to Density Conversion](1282-sdf-to-density-conversion.md) [S1, unofficial].
## How it works
**Inputs:**
- `SD`: Signed Distance value [S1, unofficial].
- `SSR`: Screen Space Ratio [S1, unofficial].
- `SDR`: Signed Distance Ratio [S1, unofficial].
- `isoPerimeter`: Iso-perimeter value controlling dilation/contraction [S1, unofficial].
- `softness`: Softness parameter for edge blending [S1, unofficial].
- `d`: Intermediate distance vector for variants like `ComputeSDF44_float` [S1, unofficial].

**Outputs:**
- `outAlpha`: Computed alpha value [S1, unofficial].

**Data Structures:**
- `d`: A vector type (e.g., `float` for scalar variants, `float4` for `ComputeSDF44_float`) [S1, unofficial].
- For `ComputeSDF44_float`, `d` contains components `x, y, z, w` [S1, unofficial].

**Steps:**
1. Compute intermediate distance `d_val`: $d\_val = (SD - 0.5) \times SDR$ [S1, unofficial].
2. Variant-specific preprocessing: If using `ComputeSDF44_float` with outline enabled, set $d.w = \max(\max(d.x, d.y), d.z)$ before computing alpha [S1, unofficial].
3. Compute Alpha: $outAlpha = \text{saturate}\left( \frac{d\_val \times 2.0 \times SSR + 0.5 + isoPerimeter \times SDR \times SSR + softness \times 0.5}{1.0 + softness} \right)$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_sdf_alpha(SD, SSR, SDR, isoPerimeter, softness, d=None, outline_enabled=False):
    # Step 1: Compute intermediate distance d
    d_val = (SD - 0.5) * SDR
    
    # Step 2: Variant-specific preprocessing
    if outline_enabled and d is not None:
        d.w = max(d.x, max(d.y, d.z))
        d_val = d.w
        
    # Step 3: Compute Alpha
    numerator = d_val * 2.0 * SSR + 0.5 + isoPerimeter * SDR * SSR + softness * 0.5
    denominator = 1.0 + softness
    outAlpha = numerator / denominator
    
    # Saturate to [0, 1]
    if outAlpha < 0.0:
        outAlpha = 0.0
    elif outAlpha > 1.0:
        outAlpha = 1.0
        
    return outAlpha
```

**Parameters:**
- `SSR`, `SDR`, `isoPerimeter`, `softness`: Typical values are Not given in the sources [S1, unofficial].

**Edge Cases:**
- Division by zero: The denominator is `1.0 + softness`; if `softness == -1.0`, division by zero occurs [G]. Guard against `softness == -1.0` before division [G].
- Precision: Floating-point precision may affect `d_val` calculation for very small or large `SD` values [G].

## Subtopics

### UV Offset Computation
Computes UV offset based on texture dimensions, offset values, and Signed Distance Ratio (SDR) [S1, unofficial].
**Formula:** $uvOffset = \text{float2}\left( \frac{-offset.x \times SDR}{texWidth}, \frac{-offset.y \times SDR}{texHeight} \right)$ [S1, unofficial].
**Inputs:** `offset` (float2), `SDR` (float), `texWidth` (float), `texHeight` (float) [S1, unofficial].
**Output:** `uvOffset` (float2) [S1, unofficial].

## Operating characteristics
Time and memory complexity, measured performance, strengths, limitations, and usage guidelines are Not given in the sources [S1, unofficial].

## Background
Origin, history, and development context are Not given in the sources [S1, unofficial].

## Sources
[S1] andersblomqvist/unity-nanovdb-renderer, https://github.com/andersblomqvist/unity-nanovdb-renderer, fetched 2026-10-27 (unofficial)
