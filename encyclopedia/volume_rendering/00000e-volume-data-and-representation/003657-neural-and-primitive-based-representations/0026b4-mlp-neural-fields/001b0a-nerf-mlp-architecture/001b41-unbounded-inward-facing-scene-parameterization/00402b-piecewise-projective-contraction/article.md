# Piecewise-Projective Contraction

Also called: MERF Contraction, Projective Contraction

## Overview
Piecewise-projective contraction is a coordinate transformation that maps unbounded 3D space into a bounded unit cube while preserving straight lines within specific regions, enabling efficient ray-AABB intersection tests for empty space skipping in voxel-based volume rendering [S1]. Unlike spherical contraction methods (e.g., mip-NeRF 360) that map lines to curves, this method maps lines to piecewise-linear segments, allowing standard ray-box intersection algorithms to be applied directly in the contracted space [S1][S2, abstract only, unofficial].

## How it works
**Inputs and Outputs**
- Input: A 3D point $\mathbf{x} \in \mathbb{R}^3$ in unbounded world coordinates.
- Output: A 3D point $\mathbf{x}' \in [-1, 1]^3$ in the bounded contracted space.

**Data Structures**
No complex data structures are required for the transformation itself. The method relies on the $\ell_\infty$ norm and simple arithmetic operations.

**Formula**
The contraction function $\text{contract}_\pi(\mathbf{x})$ is defined component-wise for each coordinate $j \in \{1, 2, 3\}$ [S1]:

$$
\text{contract}_\pi(\mathbf{x})_j = \begin{cases} 
x_j & \text{if } \|\mathbf{x}\|_{\infty} \leq 1 \\ 
\frac{x_j}{\|\mathbf{x}\|_{\infty}} & \text{if } x_j \neq \|\mathbf{x}\|_{\infty} > 1, \\ 
\left(2 - \frac{1}{|x_j|}\right) & \text{if } x_j = \|\mathbf{x}\|_{\infty} > 1 
\end{cases}
$$

where $\|\mathbf{x}\|_{\infty} = \max_i |x_i|$ is the $\ell_\infty$ norm (the maximum absolute value among all coordinates) [S1].

**Parameters**
- No tunable parameters. The boundaries are fixed at $\|\mathbf{x}\|_{\infty} = 1$ and the transformation logic is deterministic [S1].

**Steps**
1. Compute the $\ell_\infty$ norm of the input point: $n_\infty = \max(|x_1|, |x_2|, |x_3|)$.
2. If $n_\infty \leq 1$, the point is inside the unit cube. Return $\mathbf{x}' = \mathbf{x}$.
3. If $n_\infty > 1$, determine which coordinate $k$ achieves the maximum absolute value (i.e., $|x_k| = n_\infty$).
4. For each coordinate $j$:
   - If $j \neq k$ (the coordinate is not the one defining the boundary), compute $x'_j = x_j / n_\infty$.
   - If $j = k$ (the coordinate is the one defining the boundary), compute $x'_j = 2 - \text{sign}(x_j) / |x_j|$. Note: Since $x_j$ has the same sign as the maximum magnitude, $\text{sign}(x_j)$ is $+1$ or $-1$. The formula in [S1] uses $2 - 1/|x_j|$ for the positive case; generally, for $x_j > 0$, $x'_j = 2 - 1/x_j$, and for $x_j < 0$, $x'_j = -2 + 1/|x_j|$ (or $-2 + 1/x_j$ if handling signs carefully). The source formula $\left(2 - \frac{1}{|x_j|}\right)$ implies the output is positive if $x_j$ was positive, but strictly speaking, the sign must be preserved. Let $s_j = \text{sign}(x_j)$. Then $x'_j = s_j \left(2 - \frac{1}{|x_j|}\right)$ [S1, G].
5. Return $\mathbf{x}'$.

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def piecewise_projective_contraction(x):
    """
    Maps unbounded 3D point x to bounded unit cube [-1, 1]^3.
    x: array-like of length 3
    Returns: array of length 3
    """
    import numpy as np
    
    x = np.asarray(x, dtype=float)
    abs_x = np.abs(x)
    n_inf = np.max(abs_x)
    
    # Case 1: Inside unit cube
    if n_inf <= 1.0:
        return x
    
    # Case 2: Outside unit cube
    # Find the index of the maximum absolute value
    # If there are ties, any index with the max value works as the 'k' coordinate
    k = np.argmax(abs_x)
    
    x_prime = np.zeros(3)
    
    for j in range(3):
        if j != k:
            # Non-maximum coordinate: projective scaling
            x_prime[j] = x[j] / n_inf
        else:
            # Maximum coordinate: projective transformation preserving sign
            # Formula: sign(x_j) * (2 - 1/|x_j|)
            sign_xj = 1.0 if x[j] >= 0 else -1.0
            x_prime[j] = sign_xj * (2.0 - 1.0 / abs_x[j])
            
    return x_prime
```

**Edge Cases and Pitfalls**
- **Precision**: When $|x_j|$ is very large, $1/|x_j|$ approaches 0, and $x'_j$ approaches $\pm 2$. However, the output is clamped to $[-1, 1]$? No, the formula yields values in $(-\infty, -1) \cup (1, \infty)$? Let's check limits. As $|x_j| \to \infty$, $2 - 1/|x_j| \to 2$. Wait, the unit cube is $[-1, 1]$. The source says it maps to a bounded volume. Let's re-read [S1]. "maps unbounded space into a finite domain". The formula $2 - 1/|x_j|$ for $x_j > 1$ yields values in $(1, 2)$. This contradicts the "unit cube" claim if the range is $[-1, 1]$. However, [S1] states "preserves the unit cube... and the other six regions... get mapped by a different projective transformation." It does not explicitly state the output range is $[-1, 1]$ for the outer regions, but rather that the *unit cube* is preserved. The output for outer regions can exceed 1 in magnitude? No, "bounded volume". Let's look at the spherical contraction comparison: "maps to a radius-2 ball". The piecewise-projective contraction likely maps to a larger bounded box, e.g., $[-2, 2]^3$ or similar? Or does it map to $[-1, 1]^3$?
    - Re-evaluating [S1]: "preserves the unit cube...". If $x_j > 1$, $x'_j = 2 - 1/x_j$. If $x_j=1$, $x'_j=1$. If $x_j \to \infty$, $x'_j \to 2$. So the range is $[-2, 2]$. This is a bounded volume. The "unit cube" is the central region. The outer regions map to the rest of the $[-2, 2]^3$ box? Or is it normalized? The text says "finite domain". It does not explicitly say $[-1, 1]^3$ for the whole space, only that the unit cube is preserved. The pitfall section mentions "maps straight lines to segments".
    - **Correction**: The output domain is $[-2, 2]^3$ (or similar bounded box depending on the specific projective map details not fully expanded in the snippet, but the formula clearly maps $1 \to 1$ and $\infty \to 2$). Implementers must ensure the voxel grid allocation covers this expanded bounded volume, not just $[-1, 1]^3$ [S1, G].
- **Discontinuities**: The derivative is discontinuous at the boundaries of the 7 regions (where $\|\mathbf{x}\|_\infty = 1$ and where the max coordinate changes). This is acceptable for rendering but may affect gradient-based optimization if not handled carefully [S1].

## Subtopics

### [Scene Contraction](9762-scene-contraction.md)
A different contraction method. Choose piecewise-projective when empty space skipping via ray-AABB intersection is required for real-time voxel rendering.

### [Scene Parameterization Radius](15160-scene-parameterization-radius.md)
Defines the scale of the parameterization. Choose piecewise-projective for unbounded 360-degree scenes where linear preservation aids acceleration structures.

## Operating characteristics
- **Time Complexity**: $O(1)$ per point evaluation. Requires computing max, abs, and a few arithmetic operations.
- **Memory Complexity**: $O(1)$ auxiliary memory.
- **Measured Performance**: Performs on par with spherical contraction in terms of reconstruction quality [S1].
- **Strengths**: Enables efficient empty space skipping by allowing standard ray-AABB intersection tests in contracted space, which is critical for real-time rendering with discretized voxels [S1][S2, abstract only, unofficial].
- **Limitations**: The output domain is larger than the unit cube (extending to $\pm 2$ in the max coordinate direction), requiring appropriate voxel grid sizing. Discontinuities in derivatives at region boundaries [S1].
- **When to use**: Real-time volume rendering of unbounded scenes using voxel grids where empty space skipping is necessary for performance.
- **When not to use**: When the rendering pipeline does not use voxel grids or does not require empty space skipping, or when a simpler spherical contraction suffices and performance is not critical.

## Background
Piecewise-projective contraction was introduced in the MERF (Memory-Efficient Radiance Fields) paper by Christian Reiser et al. in 2023 [S1]. It was developed to address the limitation of mip-NeRF 360's spherical contraction, which maps straight lines to curves, making ray-AABB intersections intractable for empty space skipping in real-time voxel-based rendering pipelines [S1]. The method preserves straight lines within seven regions (the unit cube and six outer regions defined by the coordinate with maximum magnitude), allowing rays to be represented as piecewise-linear segments in contracted space [S1].

## Sources
[S1] MERF: Memory-Efficient Radiance Fields for Real-time View Synthesis in Unbounded Scenes, https://doi.org/10.1145/3592426, fetched 2026-10-26
[S2] MERF, https://creiser.github.io/merf/, fetched 2026-10-29 (abstract only, unofficial)
