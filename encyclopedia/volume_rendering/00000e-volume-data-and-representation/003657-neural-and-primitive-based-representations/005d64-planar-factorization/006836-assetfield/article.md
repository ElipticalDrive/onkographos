# AssetField

Also called: Ground Feature Plane Representation, I-AssetField, I-AssetField Baseline, Integrated AssetField, S-AssetField, S-AssetField Baseline, Standard AssetField

## Overview
AssetField is a neural scene representation that factorizes a 3D radiance field into a 2D ground feature plane and a vertical feature axis to enable object-aware scene editing and asset mining [S2]. It solves the problem of sparse-view overfitting and lack of explicit object structure in traditional grid-based methods by aligning the primary feature representation with the physical ground (Bird's Eye View) [S1, S2]. Unlike sibling methods like [K-planes](3317-k-planes.md) which use uniform factorization across all axes, AssetField specializes in ground-aligned scenes by decoupling horizontal spatial information from vertical height information, allowing for intuitive 2D manipulation of 3D assets [S2].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of multi-view 2D images of a scene. Optionally, camera poses.
*   **Output:** A parameterized scene function $F_\Theta: \mathbb{R}^3 \rightarrow \mathbb{R}^k$ (typically density $\sigma$ and color $\mathbf{c}$) and a set of learned feature planes enabling asset extraction and scene editing.

**Data Structures:**
The representation relies on two primary learnable grids:
1.  **Ground Feature Plane ($M$):** A 2D grid of shape $L \times W \times N$. It encodes scene properties (density, color, semantics) projected onto the $xy$-plane (ground).
    *   $L, W$: Spatial resolution of the ground plane.
    *   $N$: Feature dimension.
    *   Variants: $M_\sigma$ (density), $M_c$ (color), $M_{dino}$ (semantic DINO features).
2.  **Vertical Feature Axis ($H$):** A 1D grid of shape $H \times N$. It encodes global vertical features along the $z$-axis.
    *   $H$: Resolution along the vertical axis.
    *   $N$: Feature dimension (must match $M$'s dimension).

**Parameters:**
*   $L, W$: Typically $300 \times 300$ [S2].
*   $H$: Not explicitly specified in sources, but defined as the vertical resolution [S2].
*   $N$: Feature dimension, shared between $M$ and $H$ [S2].
*   Optimizer: Adam [S2].
*   Learning Rate: $0.02$ for AssetField/TensoRF variants; $5 \times 10^{-4}$ for NeRF baseline [S2].
*   Iterations: $50,000$ for AssetField/TensoRF; $200,000$ for NeRF [S2].

**Steps:**

1.  **Coordinate Alignment (Preprocessing):**
    If the scene's coordinate system is not aligned with the physical ground (e.g., table-top or floor), perform Principal Component Analysis (PCA) on the camera poses to rotate the scene such that the $xy$-plane aligns with the ground surface [S2].

2.  **Feature Retrieval:**
    For a query 3D point $\mathbf{q} = (x, y, z)$, retrieve features from the ground plane and vertical axis using interpolation:
    $$
    \mathbf{m} = \text{Interp}(M, (x, y))
    $$
    $$
    \mathbf{h} = \text{Interp}(H, z)
    $$
    Where $\text{Interp}$ denotes bilinear interpolation for $M$ and linear interpolation for $H$ [S2]. The retrieved vectors are $\mathbf{m} = (m_\sigma, m_c)$ and $\mathbf{h} = (h_\sigma, h_c)$ for density and color components respectively [S2].

3.  **Feature Combination:**
    Combine the ground and vertical features. The baseline AssetField uses element-wise operations or concatenation followed by decoding. The S-AssetField and I-AssetField variants explicitly adopt the **outer-product** to combine ground plane features and vertical axis features, following the TensoRF formulation [S2].
    *   *Note:* The parent article [Planar Factorization](17382-planar-factorization.md) describes a general Hadamard product combination. AssetField specifically utilizes the outer-product in its S/I variants to capture cross-dimensional interactions [S2].

4.  **Decoding:**
    The combined features are decoded into density $\sigma$ and view-dependent color $\mathbf{c}$ using Multi-Layer Perceptrons (MLPs) denoted as $\text{Dec}_\sigma$ and $\text{Dec}_{rgb}$ [S2].
    *   **S-AssetField (Standard):** Separately models density, RGB, and DINO fields using separate ground planes and vertical axes [S2].
    *   **I-AssetField (Integrated):** Models the density field separately but uses an integrated RGB-DINO ground feature plane to associate appearance with space occupancy, improving warping plausibility [S2].

5.  **Rendering:**
    Render the color $\hat{C}(\mathbf{r})$ for a ray $\mathbf{r}$ by volumetric integration along the ray:
    $$
    \hat{C}(\mathbf{r}) = \sum_{i=1}^{T_i} \left(1 - \exp(-\sigma_i \delta_i)\right) \mathbf{c}_i
    $$
    $$
    T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right)
    $$
    Where $\sigma_i$ and $\mathbf{c}_i$ are the density and color at sample point $i$, and $\delta_i$ is the distance between samples [S2].

6.  **Optimization:**
    Minimize the 2D image reconstruction loss:
    $$
    \mathcal{L} = \|\hat{C}(\mathbf{r}) - C(\mathbf{r})\|_2^2
    $$
    Where $C(\mathbf{r})$ is the ground truth pixel color [S2].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def assetfield_forward(query_points, M_sigma, M_color, H_sigma, H_color, Dec_sigma, Dec_rgb):
    """
    Forward pass for AssetField representation.
    
    Args:
        query_points: Tensor of shape (N, 3), coordinates (x, y, z).
        M_sigma: Tensor of shape (L, W, N), ground plane for density.
        M_color: Tensor of shape (L, W, N), ground plane for color.
        H_sigma: Tensor of shape (H, N), vertical axis for density.
        H_color: Tensor of shape (H, N), vertical axis for color.
        Dec_sigma: MLP for density decoding.
        Dec_rgb: MLP for color decoding.
        
    Returns:
        sigma: Tensor of shape (N,), predicted density.
        color: Tensor of shape (N, 3), predicted color.
    """
    x = query_points[:, 0]
    y = query_points[:, 1]
    z = query_points[:, 2]
    
    # 1. Retrieve features via interpolation
    # Interp(M, (x, y)) performs bilinear interpolation on the 2D plane
    m_sigma = interpolate_2d(M_sigma, x, y)
    m_color = interpolate_2d(M_color, x, y)
    
    # Interp(H, z) performs linear interpolation on the 1D axis
    h_sigma = interpolate_1d(H_sigma, z)
    h_color = interpolate_1d(H_color, z)
    
    # 2. Combine features
    # S-AssetField/I-AssetField use outer product or concatenation.
    # Here we assume concatenation followed by MLP as a generic baseline,
    # though outer product is specified for S/I variants.
    # For outer product: combined = torch.einsum('...n,...m->...nm', m, h)
    # Simplified to concatenation for standard MLP decoding compatibility:
    combined_sigma = torch.cat([m_sigma, h_sigma], dim=-1)
    combined_color = torch.cat([m_color, h_color], dim=-1)
    
    # 3. Decode
    sigma = Dec_sigma(combined_sigma)
    # Apply sigmoid or similar activation to ensure positive density if required by decoder
    sigma = torch.relu(sigma) 
    
    color = Dec_rgb(combined_color)
    
    return sigma, color

def interpolate_2d(grid_3d, x_coords, y_coords):
    """
    Bilinear interpolation from a 3D grid (L, W, N) given 2D coordinates.
    """
    # Normalize coordinates to [-1, 1] or [0, 1] depending on grid convention
    # This is a placeholder for the specific interpolation implementation
    # Standard grid_sample or similar function would be used here
    pass

def interpolate_1d(grid_2d, z_coords):
    """
    Linear interpolation from a 2D grid (H, N) given 1D coordinates.
    """
    pass
```

**Edge Cases and Pitfalls:**
*   **Overlapping Objects:** AssetField does not directly support manipulating overlapping or stacked objects because the ground plane representation collapses vertical information [S2]. To handle this, the ground plane can be expanded back to 3D feature grids paired with the vertical axis [S2].
*   **Sparse Views:** Full 3D feature grids overfit and produce noisy values with sparse views. The ground feature plane representation is more robust in this scenario [S2].
*   **Coordinate Alignment:** If the scene is not aligned with the physical ground, PCA on camera poses is required to align the $xy$-plane with the ground/table-top; otherwise, the representation fails to capture object structure correctly [S2].
*   **Template Noise:** Imbalanced training view distribution leads to noisy feature patches. Template refinement (optimizing representative category templates instead of individual instances) alleviates this [S2].

## Subtopics

### S-AssetField (Standard AssetField)
Separately models density, RGB, and DINO fields using independent ground planes and vertical axes. Chosen when distinct control over semantic and appearance features is needed without cross-modal constraints [S2].

### I-AssetField (Integrated AssetField)
Integrates RGB and DINO features into a single ground plane while keeping density separate. Chosen for better warping plausibility and robustness to vertical displacement, as it associates appearance with space occupancy [S2].

## Operating characteristics
*   **Training Speed:** Converges 40x faster than NeRF [S2]. Trained for 50k iterations vs NeRF's 200k [S2].
*   **Inference Speed:** Renders 30x faster than NeRF [S2].
*   **Accuracy (PSNR):**
    *   City Scene: NeRF 24.55, TensoRF 27.61, S-AssetField 27.54, I-AssetField 27.95 [S2].
    *   Vertical Displacement Scene: TensoRF 35.873, S-AssetField 36.358, I-AssetField 36.452 [S2].
*   **Memory:** Compact representation compared to full 3D grids. Resolution typically $300 \times 300$ for planes [S2].
*   **Strengths:** Robust to sparse views; enables intuitive 2D scene editing (translation, rescaling, insertion); supports asset mining and grouping; faster training/inference than NeRF [S2].
*   **Limitations:** Does not natively support overlapping/stacked objects; requires ground-aligned scenes (or PCA alignment); sensitive to insufficient point supervision without template refinement [S2].
*   **When to Use:** Ground-level scenes (indoor/outdoor) where object-centric editing and asset reuse are desired, and views may be sparse [S2].
*   **When Not to Use:** Scenes with significant vertical complexity or stacked objects where 3D grid control is necessary; scenes not aligned with a ground plane without preprocessing [S2].

## Background
AssetField was introduced by Xiangli et al. in 2023 [S2]. It builds on the observation that indoor and outdoor environments are structured and repetitive, inspired by traditional pipelines that use asset libraries [S2]. It extends the concept of ground planes used in indoor/outdoor modeling [36, 13, 32] to neural radiance fields, enabling unsupervised asset mining and reconfiguration [S2]. The method was published at ICCV 2023 [S2].

## Sources
[S1] Semantically-aware Neural Radiance Fields for Visual Scene Understanding: A Comprehensive Review, https://link.springer.com/article/10.1007/s11263-025-02663-5, fetched 2026-10-27
[S2] AssetField: Assets Mining and Reconfiguration in Ground Feature Plane Representation, https://doi.org/10.1109/iccv51070.2023.00301, fetched 2026-10-29
