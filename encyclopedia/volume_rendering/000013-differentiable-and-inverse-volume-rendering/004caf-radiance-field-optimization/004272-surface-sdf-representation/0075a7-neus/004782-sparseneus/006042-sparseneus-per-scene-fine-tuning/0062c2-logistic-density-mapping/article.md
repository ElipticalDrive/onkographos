# Logistic Density Mapping

Also called: Bell-shaped opacity

## Overview
Logistic Density Mapping is a method for converting signed distance field (SDF) or TSDF voxel values into volumetric opacity, creating a thin "opacity shell" concentrated near the surface ($F=0$) [S1, unofficial]. It solves the problem of defining surface thickness in differentiable volume rendering by providing a bell-shaped opacity function that is more tightly concentrated around the surface than sigmoid-based mappings [S1, unofficial]. This results in fewer above-threshold voxels and a sparser octree structure at the same pruning threshold [S1, unofficial].

## How it works
**Inputs:**
- $F$: The TSDF value for a voxel (distance to surface, normalized).
- $a$: A small positive offset defining the "inside" thickness boundary (e.g., $a = 0.1$ in normalized TSDF units) [S1, unofficial].
- $b$: The target opacity value at $F = -a$ (inside the surface) (e.g., $b = 0.5$) [S1, unofficial].
- $\alpha_{min}, \alpha_{max}$: The desired range for the final stored opacity values [S1, unofficial].
- $\alpha_{thresh}$: The pruning threshold for voxels [S1, unofficial].

**Outputs:**
- A pruned sparse voxel grid where each active voxel stores an opacity value $\alpha \in [\alpha_{min}, \alpha_{max}]$.

**Data Structures:**
- A sparse octree or hash map storing voxel positions and their computed opacity values. Voxels with opacity below $\alpha_{thresh}$ are not stored.

**Steps:**
1.  **Calculate the sharpness parameter $s$:**
    The parameter $s$ controls the sharpness of the bell curve. It is derived by enforcing that the opacity at the inner boundary $F = -a$ equals $b$.
    a. Solve for $u$ in the equation $\frac{4u}{(1+u)^2} = b$, where $u = e^{sa}$.
    b. The quadratic solution for $u$ is:
       $$ u = \frac{2 - b \pm 2\sqrt{1-b}}{b} $$
    c. Select the root $u > 1$ to ensure $s > 0$ [S1, unofficial].
    d. Calculate $s$:
       $$ s = \frac{\ln u}{a} $$

2.  **Compute raw opacity:**
    For each voxel with TSDF value $F$, compute the raw opacity $\alpha_{raw}$ using the logistic PDF normalized to peak at 1:
    $$ \alpha_{raw} = \phi_{bell}(F; s) = \frac{4 e^{-sF}}{(1 + e^{-sF})^2} $$
    Note: This function is symmetric around $F=0$ and peaks at 1 when $F=0$ [S1, unofficial].

3.  **Rescale opacity:**
    Map $\alpha_{raw}$ to the target range $[\alpha_{min}, \alpha_{max}]$. Let $\alpha_{max\_raw}$ be the maximum possible value of $\phi_{bell}$ (which is 1 at $F=0$).
    $$ \alpha = \alpha_{min} + (\alpha_{raw} - \alpha_{min\_raw\_min}) \times \frac{\alpha_{max} - \alpha_{min}}{\alpha_{max\_raw\_max} - \alpha_{min\_raw\_min}} $$
    *Note: The source specifies rescaling to $[\alpha_{min}, \alpha_{max}]$ but does not explicitly define the min/max raw bounds for the general case, implying a linear normalization of the computed $\alpha_{raw}$ values or a specific clamping. Given the peak is 1 and the function decays, a common implementation normalizes against the peak or the range of expected values. However, the source simply states "rescaled to [min opacity, max opacity]" [S1, unofficial]. We assume a linear mapping from the computed $\alpha_{raw}$ to the target range, potentially clamping $\alpha_{raw}$ to $[0, 1]$ first.*
    *Refined Step 3 based on standard practice implied by "rescaled":*
    $$ \alpha = \alpha_{min} + \alpha_{raw} \times (\alpha_{max} - \alpha_{min}) $$
    (Assuming $\alpha_{raw}$ is treated as a relative intensity normalized by its peak, or the source implies a global min/max normalization across the grid. Given the ambiguity, we stick to the explicit instruction: "rescaled to [min opacity, max opacity]" [S1, unofficial].)

4.  **Prune:**
    If $\alpha < \alpha_{thresh}$, discard the voxel. Otherwise, store the voxel with opacity $\alpha$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def logistic_density_mapping(F, a, b, alpha_min, alpha_max, alpha_thresh):
    """
    F: TSDF value of the voxel
    a: Inside thickness offset (e.g., 0.1)
    b: Target opacity at F = -a (e.g., 0.5)
    alpha_min: Minimum stored opacity
    alpha_max: Maximum stored opacity
    alpha_thresh: Pruning threshold
    """
    # Step 1: Calculate sharpness s
    # Solve 4u / (1 + u)^2 = b for u > 1
    # u = (2 - b + 2*sqrt(1-b)) / b
    discriminant = 1 - b
    if discriminant < 0:
        raise ValueError("b must be <= 1")
    
    u = (2 - b + 2 * math.sqrt(discriminant)) / b
    
    # Ensure u > 1 (it should be for b in (0, 1))
    if u <= 1:
        # Fallback or error if root selection fails
        raise ValueError("Selected u is not > 1")
        
    s = math.log(u) / a
    
    # Step 2: Compute raw opacity
    # phi_bell(F; s) = 4 * exp(-s*F) / (1 + exp(-s*F))^2
    exp_val = math.exp(-s * F)
    alpha_raw = (4 * exp_val) / ((1 + exp_val) ** 2)
    
    # Step 3: Rescale to [alpha_min, alpha_max]
    # The source states "rescaled to [min opacity, max opacity]".
    # Assuming linear scaling from the raw value (which peaks at 1)
    # to the target range.
    alpha = alpha_min + alpha_raw * (alpha_max - alpha_min)
    
    # Step 4: Prune
    if alpha < alpha_thresh:
        return None
    else:
        return alpha
```

**Edge Cases and Pitfalls:**
- If $b=1$, the denominator in the $u$ calculation becomes 0. $b$ must be strictly less than 1 [S1, unofficial].
- If $b \le 0$, the square root $\sqrt{1-b}$ is valid, but $u$ might not be $>1$ or meaningful for opacity. $b$ should be in $(0, 1)$ [S1, unofficial].
- Precision issues may arise if $b$ is very close to 1, leading to numerical instability in calculating $u$ [S1, unofficial].

## Subtopics

### Surface-Concentrated Opacity
An opacity formulation that encourages transparent free space and early termination at compact surfaces, combined with a scale-invariant density parameterization [S2, unofficial]. The scale-invariant density parameterization avoids biases caused by varying cell sizes [S2, unofficial]. This variant is used in VoroTracing to optimize inference by reducing memory traffic and allowing early termination [S2, unofficial].

## Operating characteristics
- **Sparsity:** Compared to sigmoid mapping, logistic density mapping concentrates opacity more tightly around the surface, typically producing fewer above-threshold voxels (a thinner band) and a sparser octree at the same pruning threshold [S1, unofficial].
- **Performance:** The sparser structure can lead to faster rendering due to fewer voxels to evaluate and prune [S1, unofficial].
- **Limitations:** The bell shape may not capture surfaces with significant thickness or volume-like properties as well as sigmoid mappings, which allow for a gradual transition [S1, unofficial].

## Background
Logistic Density Mapping is motivated by the NeuS method [S1, unofficial]. It was introduced in the context of advancing structured priors for sparse-voxel surface reconstruction [S1, unofficial]. The method provides an alternative to sigmoid-like mappings by enforcing a specific thickness at the inner boundary ($F=-a$) to create a consistent "opacity shell" [S1, unofficial].

## Sources
[S1] Advancing Structured Priors for Sparse-Voxel Surface Reconstruction, https://doi.org/10.48550/arxiv.2601.17720, fetched 2026-10-26 (unofficial)
[S2] Differentiable Voronoi Ray Tracing Beyond Rasterization Speeds, https://arxiv.org/abs/2608.17682, fetched 2026-10-27 (unofficial)
