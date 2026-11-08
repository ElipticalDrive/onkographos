# Piecewise Linear Opacity Mapping with Gradient Scaling

Also called: Gradient-scaled opacity mapping, Gradient-scaled piecewise linear opacity, Surface-enhanced volume rendering

## Overview
This method maps scalar volume values to opacity using piecewise linear functions defined by tissue types, then scales the resulting opacity by the magnitude of the local gradient vector to suppress tissue interiors and enhance bounding surfaces [S1, S2, unofficial]. It solves the problem of visualizing thin, wispy structures that binary thresholding would miss, while simultaneously creating semi-transparent surface overlays (e.g., skin over bone) by attenuating interior opacity [S1, S2, unofficial]. Unlike [Linear Opacity Interpolation](7016-linear-opacity-interpolation.md), which maps opacity directly without gradient scaling, this method explicitly uses gradient magnitude to modulate visibility, making it suitable for rendering multiple semi-transparent surfaces in superposition [S1, S2, unofficial].

## How it works
**Inputs:**
- Scalar volume data $f(x_i)$ for voxels at positions $x_i$.
- $N$ tissue types, each defined by a known CT number (scalar value) $f_{vn}$ and a target opacity $\alpha_{vn}$.
- The tissue types are ordered such that $f_{v1} < f_{v2} < \dots < f_{vN}$ [S1, S2, unofficial].

**Outputs:**
- Opacity $\alpha(x_i)$ for each voxel $x_i$, ready for compositing.

**Data Structures:**
- **Tissue Definition:** An array of $N$ pairs $(f_{vn}, \alpha_{vn})$.
- **Gradient Magnitude:** A pre-computed scalar field $|\nabla f(x_i)|$ for all voxels. If not pre-computed, it must be calculated per voxel.
- **Lookup Table (Optional):** A table mapping the pair $(f(x_i), |\nabla f(x_i)|)$ to $\alpha(x_i)$, or separate tables if computed in stages.

**Assumptions and Constraints:**
The method assumes the scene meets specific adjacency criteria: tissues of each type touch at most two other types, and if ordered by CT number, each type touches only types adjacent to it in the ordering [S1, S2, unofficial]. Formally, no tissue of CT number $f_{vn}$ touches any tissue of CT number $f_{vm}$ where $|n - m| > 1$ [S1, S2, unofficial]. Violation of this criteria leads to ambiguous classification and incorrect rendering [S1, S2, unofficial].

**Steps:**

1.  **Gradient Calculation:** Compute the local gradient vector $\nabla f(x_i)$ for each voxel. The magnitude $|\nabla f(x_i)|$ is derived from this vector. The source specifies using the operator described in Section 3 of the original document [S1, S2, unofficial]. [G: Standard central difference or Sobel operators are typically used for gradient approximation in volume data.]

2.  **Interval Identification:** For a voxel with value $f(x_i)$, determine which tissue interval $[f_{vn}, f_{vn+1}]$ it falls into.
    - If $f_{vn} \le f(x_i) \le f_{vn+1}$ for some $n \in \{1, \dots, N-1\}$, proceed to Step 3.
    - If $f(x_i)$ is outside all defined intervals (i.e., $f(x_i) < f_{v1}$ or $f(x_i) > f_{vN}$), set $\alpha(x_i) = 0$ [S2, unofficial].

3.  **Linear Interpolation of Base Opacity:** Calculate the base opacity by linearly interpolating between $\alpha_{vn}$ and $\alpha_{vn+1}$ based on the position of $f(x_i)$ within $[f_{vn}, f_{vn+1}]$ [S1, S2, unofficial].

4.  **Gradient Scaling:** Multiply the interpolated base opacity by the gradient magnitude $|\nabla f(x_i)|$ [S1, S2, unofficial].

5.  **Final Opacity:** The resulting value is $\alpha(x_i)$. [G: The source implies $\alpha(x_i)$ may exceed 1.0 if $|\nabla f(x_i)| > 1$. Standard compositing requires $\alpha \in [0, 1]$. Clamp to 1.0 if necessary, or rely on the compositing step's handling of values $>1$ if the implementation supports it. The source does not explicitly state clamping, but physical opacity is bounded.]

**Formula:**
For $n = 1, \dots, N-1$, if $f_{vn} \le f(x_i) \le f_{vn+1}$:
$$
\alpha(x_i) = |\nabla f(x_i)| \left( \frac{f(x_i) - f_{vn}}{f_{vn+1} - f_{vn}} \alpha_{vn+1} + \frac{f_{vn+1} - f(x_i)}{f_{vn+1} - f_{vn}} \alpha_{vn} \right)
$$
Otherwise:
$$
\alpha(x_i) = 0
$$
[S1, S2, unofficial]

**Parameters:**
- $f_{vn}$: Known CT number for tissue type $n$. Ordered $f_{v1} < f_{v2} < \dots < f_{vN}$ [S1, S2, unofficial].
- $\alpha_{vn}$: Target opacity for tissue type $n$ at $f_{vn}$ [S1, S2, unofficial].
- $N$: Number of tissue types, $N \ge 1$ [S1, S2, unofficial].

**Edge Cases and Pitfalls:**
- **Non-adjacent tissues:** If tissues with non-adjacent CT numbers touch, voxels at boundaries cannot be unambiguously classified [S1, S2, unofficial]. Solution: Interactive clipping or segmentation to isolate subsets meeting adjacency criteria [S1, S2, unofficial].
- **Thin features:** Binary thresholds may miss thin features of tissue type B if their voxel values are less than $f_{vB}$. This method maps all voxels to non-zero opacity (within intervals), rendering thin regions as faint wisps [S1, S2, unofficial].
- **Gradient constancy:** The thickness of the transition region around isosurface contours remains constant only if the local gradient magnitude is constant within a radius of $r$ voxels around the surface [S1, S2, unofficial].
- **Striping artifacts:** CT scanners have anisotropic spatial sensitivity, causing striping. Patient motion also causes striping because shading depends on gradient orientation, and misalignments between adjacent slices disrupt this [S1, S2, unofficial].
- **High-resolution aspect ratio:** For high-resolution datasets (e.g., 512x512), casting 4 rays per slice in the vertical direction can correct for aspect ratio issues [S2, unofficial]. Alternatively, expand the dataset using cubic B-spline interpolation in the vertical direction and cast one ray per slice [S2, unofficial].
- **Blurring:** Slight blurring of opacities softens silhouettes without affecting surface detail crispness [S1, S2, unofficial]. Blurring acquired values may affect other details [S2, unofficial].
- **Super-sampling:** Interpolating additional samples between acquired ones prior to compositing improves visibility calculation accuracy and reduces aliasing, but substantially increases computational expense [S1, S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_opacity_gradient_scaled(f_voxel, f_v, alpha_v, grad_magnitude):
    """
    Computes opacity for a single voxel using piecewise linear mapping with gradient scaling.
    
    Args:
        f_voxel (float): Scalar value of the voxel f(xi).
        f_v (list of float): Ordered list of known CT numbers for N tissue types [fv1, fv2, ..., fvN].
        alpha_v (list of float): List of target opacities corresponding to f_v.
        grad_magnitude (float): Magnitude of the local gradient vector |grad f(xi)|.
        
    Returns:
        float: Computed opacity alpha(xi).
    """
    N = len(f_v)
    
    # Find the interval [f_v[n], f_v[n+1]] containing f_voxel
    n = -1
    for i in range(N - 1):
        if f_v[i] <= f_voxel <= f_v[i+1]:
            n = i
            break
            
    # If voxel is outside all defined intervals, opacity is 0
    if n == -1:
        return 0.0
        
    # Linear interpolation of base opacity
    fv_n = f_v[n]
    fv_n1 = f_v[n+1]
    alpha_n = alpha_v[n]
    alpha_n1 = alpha_v[n+1]
    
    # Avoid division by zero if adjacent tissue types have same CT number (unlikely but safe)
    if fv_n1 == fv_n:
        base_opacity = alpha_n
    else:
        t = (f_voxel - fv_n) / (fv_n1 - fv_n)
        base_opacity = (1 - t) * alpha_n + t * alpha_n1
        
    # Scale by gradient magnitude
    alpha = grad_magnitude * base_opacity
    
    # [G] Clamp opacity to [0, 1] as physical opacity is bounded.
    # The source does not explicitly state clamping, but compositing typically requires alpha <= 1.
    if alpha > 1.0:
        alpha = 1.0
    if alpha < 0.0:
        alpha = 0.0
        
    return alpha

def precompute_gradients(volume_data, shape):
    """
    Computes gradient magnitude for all voxels using central differences.
    [G] The source references an operator from Section 3 but does not detail it.
    Standard central differences are used here as a common implementation.
    """
    import numpy as np
    
    # Pad volume to handle boundaries
    padded = np.pad(volume_data, 1, mode='edge')
    
    # Compute gradients in x, y, z directions
    # Assuming shape is (z, y, x)
    gx = (padded[1:-1, 1:-1, 2:] - padded[1:-1, 1:-1, :-2]) / 2.0
    gy = (padded[1:-1, 2:, 1:-1] - padded[1:-1, :-2, 1:-1]) / 2.0
    gz = (padded[2:, 1:-1, 1:-1] - padded[:-2, 1:-1, 1:-1]) / 2.0
    
    # Compute magnitude
    grad_mag = np.sqrt(gx**2 + gy**2 + gz**2)
    
    return grad_mag
```

## Subtopics

### Linear Opacity Interpolation
Maps opacity linearly between tissue types without gradient scaling. Choose when surface enhancement via gradient magnitude is not required or when computational simplicity is prioritized over surface crispness. See [Linear Opacity Interpolation](7016-linear-opacity-interpolation.md).

## Operating characteristics
**Time and Memory Complexity:**
- **Opacity Computation:** One lookup table reference per voxel if gradient magnitudes are pre-stored. New lookup table generation is required if classification parameters change [S1, S2, unofficial].
- **Ray Tracing/Compositing:** For a square image of $n$ pixels with orthographic projection and $K$ samples per ray, the cost is $3Kn^2$ additions, $2Kn^2$ tri-linear interpolations, and $Kn^2$ linear interpolations [S1, S2, unofficial].

**Measured Performance:**
- **Molecular Graphics (Cytochrome B5, 113x113x113 voxels):** Classification/shading took 5 minutes on a SUN 4/280 with 32MB RAM. Ray tracing/compositing took 40 minutes [S1, S2, unofficial].
- **Medical Imaging (CT Cadaver, 113 slices of 256x256):** Computation of each set of colors/opacities took 20 minutes. Each view/compositing took an additional 20 minutes [S1, S2, unofficial].

**Strengths and Limitations:**
- **Strengths:** Renders thin, wispy surfaces instead of holes for regions not meeting strict threshold criteria. Suppresses tissue interiors while enhancing bounding surfaces, allowing superposition of multiple semi-transparent surfaces (e.g., skin and bone) [S1, S2, unofficial]. Non-binary classification carries 3-D bandlimiting into the 2-D projection, reducing aliasing artifacts from point sampling (one ray per pixel) [S1, S2, unofficial].
- **Limitations:** Sensitive to acquisition artifacts like striping from CT scanner anisotropy or patient motion, as shading depends on gradient orientation [S1, S2, unofficial]. Requires adjacency criteria to be met for unambiguous classification. Fixing shading (colors/opacities) while rotating causes light sources to appear to travel with the data, creating incorrect highlights [S1, S2, unofficial]. Super-sampling improves quality but substantially increases computational expense [S1, S2, unofficial].

**When to Use:**
- When visualizing multiple semi-transparent surfaces (e.g., skin over bone) is desired.
- When thin structures need to be visible even if their voxel values do not strictly match a tissue type's CT number.
- When computational resources allow for gradient calculation and ray tracing.

**When Not to Use:**
- When the data violates adjacency criteria and interactive clipping is not feasible.
- When strict, sharp binary boundaries are required without the "faint wisp" effect.
- When real-time performance is critical and gradient pre-computation or ray tracing is too expensive.

## Background
Originating from the work of Levoy in 1988, this method was introduced in "Display of Surfaces from Volume Data" [S1, S2, unofficial]. It was applied to molecular graphics (Cytochrome B5 electron density map) and medical imaging (CT cadaver study) [S1, S2, unofficial]. The method builds on the principles of piecewise linear mapping and integration [9000-piecewise-linear-mapping-and-integration.md], adding gradient scaling to enhance surface visibility and suppress interior opacity. It addresses limitations of binary classification, which creates holes for thin structures, by mapping all voxels to non-zero opacity within defined intervals [S1, S2, unofficial].

## Sources
[S1] DTIC ADA193649: Rendering of Surfaces from Volume Data, https://archive.org/details/DTIC_ADA193649, fetched 2026-10-26 (unofficial)
[S2] Display of Surfaces from Volume Data, https://graphics.stanford.edu/papers/volume-cga88/volume.pdf, fetched 2026-10-26 (unofficial)
