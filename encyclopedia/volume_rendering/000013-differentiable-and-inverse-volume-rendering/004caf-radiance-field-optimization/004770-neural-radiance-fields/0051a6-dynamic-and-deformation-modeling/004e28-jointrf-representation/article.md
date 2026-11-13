# JointRF Representation

Also called: JointRF

## Overview
JointRF is a compact residual feature grid representation designed for dynamic Neural Radiance Fields (NeRF) to handle large motions and long sequences without compromising quality [S1]. It decomposes the radiance field into a long-reference basis feature grid and a coefficient feature grid, using residual grids to capture temporal variations [S1]. Unlike canonical space warping methods which are fragile to large motions, JointRF uses residual grids to diminish temporal redundancy and support efficient compression [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A sequence of dynamic radiance fields over time, organized into Groups of Feature grids (GOF).
*   **Output:** A set of feature grids (Basis $B$, Coefficient $C$, Residual $R$) that reconstruct the radiance field for any frame $t$ in the sequence.

**Data Structures:**
The representation relies on 3D feature grids. The exact resolution and dimensionality of these grids are not specified in the sources [G: Implementers must define grid resolution $D \times D \times D$ and feature dimension $F$ based on scene complexity and hardware constraints].

1.  **Basis Feature Grid ($B$):** A long-reference grid capturing signal commonality across the sequence.
2.  **Coefficient Feature Grid ($C$):** A grid depicting spatial variations, used in conjunction with $B$ via Hadamard product.
3.  **Residual Feature Grid ($R_t$):** A compact grid representing the difference between the current frame's basis and the keyframe's basis.

**Numbered Steps:**

1.  **Grouping:** Divide the long sequence of dynamic radiance fields into multiple continuous Groups of Feature grids (GOF). Each GOF $G = \{f_t\}_{t=1}^N$ consists of $N$ successive frames [S1].
2.  **Keyframe Initialization:** For the first frame (keyframe) $f_1$ of a GOF, initialize a long-reference basis feature grid $B_1$ and a coefficient feature grid $C_1$ [S1].
3.  **Residual and Coefficient Generation:** For each subsequent frame $t$ ($1 < t \le N$) in the GOF:
    *   Compute a compact residual feature grid $R_t$ [S1].
    *   Compute a coefficient feature grid $C_t$ [S1].
    *   Note: Learning residuals for $C_t$ holds no physical significance; only the basis grid residuals are modeled [S1].
4.  **Basis Reconstruction:** Calculate the basis feature grid for frame $t$ by adding the residual compensation to the keyframe's basis grid:
    $$
    B_t = B_1 + R_t
    $$
5.  **Radiance Field Recovery:** Recover the radiance field for frame $t$ using the reconstructed basis $B_t$ and coefficient $C_t$. The color $\mathbf{c}$ and density $\sigma$ are computed by passing interpolated features through a tiny MLP $\Phi$:
    $$
    (\mathbf{c}, \sigma) = \Phi(\text{interp}(\mathbf{x}, C_t) \circ \text{interp}(\mathbf{x}, B_t), \mathbf{d})
    $$
    where $\text{interp}(\cdot)$ is the trilinear interpolation function on the grids, $\circ$ denotes the Hadamard (element-wise) product, $\mathbf{x}$ is the 3D position, and $\mathbf{d}$ is the view direction [S1].
6.  **Volume Rendering:** Render the final color $\hat{c}(r)$ for a ray $r$ using standard volume rendering:
    $$
    \hat{c}(r) = \sum_{i=1}^{N_s} T_i (1 - \exp(-\sigma_i \delta_i)) c_i
    $$
    where $T_i = \exp(-\sum_{j=1}^{i-1} \sigma_j \delta_j)$, $\delta_i$ is the distance between adjacent samples, $c_i$ is the color, and $\sigma_i$ is the density [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def jointrf_render(ray, keyframe_grids, residual_grids, coefficient_grids, t_index):
    """
    Renders a frame using the JointRF representation.
    
    Args:
        ray: Ray origin and direction.
        keyframe_grids: Tuple (B1, C1) for the keyframe (t=1).
        residual_grids: List of residual grids R_t for t > 1.
        coefficient_grids: List of coefficient grids C_t for t > 1.
        t_index: Current frame index (1-based).
        
    Returns:
        Rendered color.
    """
    # 1. Retrieve or reconstruct Basis Grid B_t
    if t_index == 1:
        B_t = keyframe_grids.B1
    else:
        # B_t = B_1 + R_t
        R_t = residual_grids[t_index - 2] # Assuming list is 0-indexed for t=2
        B_t = add_grids(keyframe_grids.B1, R_t)
        
    # 2. Retrieve Coefficient Grid C_t
    if t_index == 1:
        C_t = keyframe_grids.C1
    else:
        C_t = coefficient_grids[t_index - 2]
        
    # 3. Sample along ray and compute radiance
    samples = sample_ray(ray)
    colors = []
    transmittance = 1.0
    
    for sample in samples:
        x = sample.position
        d = sample.direction
        
        # Interpolate features from grids
        # interp(x, Grid) performs trilinear interpolation
        feat_c = interpolate_grid(C_t, x)
        feat_b = interpolate_grid(B_t, x)
        
        # Hadamard product
        combined_feat = elementwise_multiply(feat_c, feat_b)
        
        # Tiny MLP Phi to get color and density
        # Phi takes combined_feat and view direction d
        c_i, sigma_i = tiny_mlp_phi(combined_feat, d)
        
        # Volume rendering step
        delta = sample.distance_to_next
        alpha = 1.0 - exp(-sigma_i * delta)
        weight = transmittance * alpha
        colors.append(weight * c_i)
        transmittance *= (1.0 - alpha)
        
    return sum(colors) + transmittance * background_color

def add_grids(grid_a, grid_b):
    """Element-wise addition of two feature grids."""
    return grid_a + grid_b

def interpolate_grid(grid, point):
    """Trilinear interpolation of a 3D feature grid at a point."""
    # Implementation depends on specific grid library, 
    # but conceptually retrieves weighted average of 8 neighbors.
    pass

def tiny_mlp_phi(features, direction):
    """
    A small Multi-Layer Perceptron.
    Input: Feature vector (from Hadamard product) + Direction vector.
    Output: Color (3 floats) and Density (1 float).
    """
    # Standard MLP forward pass
    pass
```

**Parameters:**
*   **GOF Size ($N$):** The number of frames in a Group of Feature grids. Not given in the sources.
*   **Grid Resolution:** Not given in the sources.
*   **Feature Dimension:** Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Large Motions:** Canonical space warping methods fail under large motions; JointRF handles this via residual grids [S1].
*   **Coefficient Residuals:** Learning residuals for the coefficient grid $C_t$ is physically meaningless and should be avoided; only basis grid residuals are modeled [S1].
*   **Temporal Redundancy:** The method relies on short-term similarity; if temporal coherence is low, residual compression efficiency may drop [S1].

## Operating characteristics
*   **Time Complexity:** Rendering requires interpolating from two grids ($B_t, C_t$) and passing through a tiny MLP per sample. The reconstruction of $B_t$ is $O(1)$ per frame if $R_t$ is precomputed. Parallel rendering of non-keyframes is supported since only one keyframe is needed during rendering [S1].
*   **Memory Complexity:** Significantly reduced compared to storing full feature grids for every frame. Only $B_1, C_1$ and compact residuals $R_t$ and coefficients $C_t$ are stored. The compactness of $R_t$ facilitates compression [S1].
*   **Strengths:** Handles large motions without quality loss; diminishes temporal redundancy; supports efficient compression; enables parallel rendering of non-keyframes [S1].
*   **Limitations:** Requires grouping frames into GOFs; performance depends on the effectiveness of residual compression for the specific motion patterns [S1].
*   **When to use:** Dynamic scenes with large motions and long durations where compression efficiency is critical [S1].
*   **When not to use:** Static scenes (simpler representations suffice) or scenes with extreme topological changes that break the assumption of residual compactness [S1].

## Background
JointRF was proposed in 2024 by researchers including Qiang Hu and Xiaoyun Zhang [S1]. It addresses challenges in storing and transmitting volumetric video using NeRF, particularly for sequences with arbitrary motions and long durations [S1]. It builds on the idea of residual radiance fields (e.g., ReRF) but introduces a joint optimization scheme for representation and compression [S1].

## Sources
[S1] JOINTRF: End-To-End Joint Optimization for Dynamic Neural Radiance Field Representation and Compression, https://doi.org/10.1109/icip51287.2024.10647336, fetched 2026-10-26
