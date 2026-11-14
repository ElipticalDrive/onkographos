# Density Range Windowing

Also called: Density Clipping, Volume Thresholding

## Overview
Density Range Windowing restricts volume rendering to voxels whose scalar density falls within a specified $[min, max]$ interval, enabling the selective visualization of specific anatomical or material structures [S1, unofficial]. It distinguishes itself from its parent [Grid Visualization with Thresholding](1515-grid-visualization-with-thresholding.md) by operating on the continuous density value directly during ray casting or slicing, rather than filtering grid cells for a 3D scatter plot, and from [Glyph Lines](8934-glyph-lines.md) by preserving volumetric opacity integration rather than geometric abstraction.

## How it works
**Inputs and Outputs:**
- Input: A 3D scalar grid $V$ of density values, typically normalized to $[0, 1]$ or physical units (e.g., Hounsfield Units).
- Parameters: $m\_densityMin$ (default $0.0f$) and $m\_densityMax$ (default $1.0f$) defining the visible interval [S1, unofficial].
- Output: A 2D image where voxels outside $[m\_densityMin, m\_densityMax]$ contribute zero opacity.

**Data Structures:**
The method relies on the transfer function evaluation context. In the referenced implementation, a `window` vector stores the bounds:
- `window.x`: Window low ($min$)
- `window.y`: Window high ($max$)
- `window.z`: Shade strength (illumination factor, not part of the core windowing logic but often co-located)
- `window.w`: Opacity correction factor [S1, unofficial]

**Steps:**
1. For each sample point $p$ along the ray, retrieve the scalar density value $d = V(p)$.
2. Compare $d$ against the window bounds:
   $$
   \text{if } d < m\_densityMin \text{ or } d > m\_densityMax, \text{ then opacity } \alpha = 0.
   $$
3. If $m\_densityMin \le d \le m\_densityMax$, compute opacity $\alpha$ using the standard transfer function (e.g., linear interpolation between stops) [S1, unofficial].
4. Composite the sample using front-to-back or back-to-front accumulation.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def evaluate_density_window(density: float, min_val: float, max_val: float) -> float:
    """
    Returns opacity contribution based on density range windowing.
    Assumes a base transfer function tf(density) returns alpha in [0, 1].
    """
    # Clamp density to window range; if outside, alpha is zero
    if density < min_val or density > max_val:
        return 0.0
    
    # Evaluate standard transfer function for values within the window
    # Note: In some implementations, the transfer function is defined over [0,1]
    # and the window acts as a mask. Here we assume tf handles the mapping.
    alpha = transfer_function(density)
    return alpha
```

**Edge Cases and Pitfalls:**
- **Precision:** If $min > max$, the interval is invalid and no voxels render [G].
- **Boundaries:** Voxels exactly on the boundary ($min$ or $max$) are included (inclusive interval) [S1, unofficial].
- **Degenerate Input:** If the entire volume density range lies outside $[min, max]$, the result is a black screen.

## Subtopics

### [MIDA Visualization](11215-mida-visualization.md)
MIDA Visualization is a density range windowing technique that allows users to define settable ranges of visualized values, enabling selective visibility of specific structures within a volume. It distinguishes itself from standard thresholding by supporting multiple, independent visible ranges (envelopes) for different parts of the volume, facilitating complex exploration tasks such as carving and peeling.

## Operating characteristics
- **Time Complexity:** $O(1)$ per sample, as it involves a simple comparison and potential transfer function lookup. No additional memory allocation.
- **Memory Complexity:** $O(1)$ auxiliary memory for window parameters.
- **Strengths:** Immediate visual isolation of structures (e.g., bone vs. soft tissue); computationally inexpensive; intuitive for radiological inspection (Window Level/Width).
- **Limitations:** Cannot visualize structures with overlapping density ranges simultaneously without multiple passes or advanced techniques like MIDA; abrupt cutoffs at boundaries may cause aliasing if not smoothed [G].
- **When to use:** When the goal is to isolate specific tissue types or materials with distinct density values.
- **When not to use:** When multiple overlapping structures need simultaneous visualization with different opacities (use MIDA or multi-range transfer functions).

## Background
Originating from radiological imaging practices, "Windowing" (or Window Level/Width) is a standard technique for displaying CT scans where the dynamic range of Hounsfield Units exceeds display capabilities. The implementation in [S1, unofficial] adapts this concept to general GPU volume rendering, allowing real-time adjustment of the visible density interval. It is commonly used in medical visualization tools for bone, soft tissue, and angiography presets.

## Sources
[S1] sho3la/GPUGemsVolumeRendering, https://github.com/sho3la/GPUGemsVolumeRendering, fetched 2026-10-26 (unofficial)
