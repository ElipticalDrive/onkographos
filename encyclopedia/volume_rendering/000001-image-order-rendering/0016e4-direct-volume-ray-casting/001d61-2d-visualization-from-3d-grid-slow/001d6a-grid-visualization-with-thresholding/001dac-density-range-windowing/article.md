# Density Range Windowing

Also called: Density Clipping, Volume Thresholding

## Overview
Density Range Windowing restricts volume rendering to voxels whose scalar density falls within a specified $[min, max]$ interval, enabling the selective visualization of specific anatomical or material structures [S1, unofficial]. Unlike sibling methods such as [Glyph Lines](8934-glyph-lines.md) that visualize vector or gradient fields, this technique operates as a direct scalar interval filter applied during ray casting.

## How it works
**Inputs and Outputs:**
- Input: A 3D scalar grid of density values.
- Output: A 2D image where voxels outside $[min, max]$ contribute zero opacity [G].

**Data Structures:**
The method uses a `window` vector to store bounds:
- `window.x`: Window low ($min$)
- `window.y`: Window high ($max$) [S1, unofficial]

**Steps:**
1. For each sample point along the ray, retrieve the scalar density value $d$ [G].
2. Compare $d$ against the window bounds: if $d < m\_densityMin$ or $d > m\_densityMax$, set opacity $\alpha = 0$ [S1, unofficial].
3. If $m\_densityMin \le d \le m\_densityMax$, evaluate the base opacity using the transfer function [G].
4. Composite the sample using front-to-back or back-to-front accumulation [G].
5. Adjust $m\_densityMin$ to drop semi-transparent structures (e.g., skin/foliage) and retain dense cores (bone), or lower $m\_densityMax$ to peel off densest structures [S1, unofficial].

**Formulas:**
$$
\alpha = \begin{cases} 
0 & \text{if } d < m\_densityMin \text{ or } d > m\_densityMax \\
\text{TF}(d) & \text{otherwise}
\end{cases}
$$
where $d$ is the sampled density, $\alpha$ is the opacity contribution, and $\text{TF}(d)$ is the transfer function evaluation [G].

**Parameters:**
- $m\_densityMin$ (default $0.0f$) and $m\_densityMax$ (default $1.0f$) define the visible density interval [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Precision:** If $m\_densityMin > m\_densityMax$, the interval is invalid and no voxels render [G].
- **Boundaries:** Voxels exactly on the boundary ($m\_densityMin$ or $m\_densityMax$) are included (inclusive interval) [G].
- **Degenerate Input:** If the entire volume density range lies outside $[m\_densityMin, m\_densityMax]$, the result is a black screen [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def apply_density_window(density: float, min_val: float, max_val: float, base_opacity: float) -> float:
    """Applies density range windowing to a sample's base opacity."""
    if density < min_val or density > max_val:
        return 0.0
    return base_opacity  # Base opacity is computed from the transfer function [G]
```

## Subtopics

### [MIDA Visualization](11215-mida-visualization.md)
MIDA Visualization is a density range windowing technique that allows users to define settable ranges of visualized values, enabling selective visibility of specific structures within a volume. It distinguishes itself from standard thresholding by supporting multiple, independent visible ranges (envelopes) for different parts of the volume, facilitating complex exploration tasks such as carving and peeling.

## Operating characteristics
Not given in the sources.

## Background
The technique adapts radiological "Window Level/Width" (WL/WW) practices, where the dynamic range of Hounsfield Units is remapped to display capabilities [S1, unofficial]. In the referenced implementation, presets such as "CT Bone", "CT Soft Tissue", and "CT Angio" use default window levels and widths to control the visible density interval [S1, unofficial].

## Sources
[S1] sho3la/GPUGemsVolumeRendering, https://github.com/sho3la/GPUGemsVolumeRendering, fetched 2026-10-26 (unofficial)
