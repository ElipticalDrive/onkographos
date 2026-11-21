# Deformation Network (Dynamic)


## Overview
The Deformation Network (Dynamic) is a method for modeling temporal dynamics in 3D Gaussian Splatting by predicting time-dependent deformations for Gaussian attributes. It solves the problem of animating static Gaussian splats by warping their means, scales, opacities, and spherical harmonics coefficients based on a query time $t$ [S1, unofficial]. It sets itself apart from sibling methods like Dynamic Object Embedding by applying explicit displacement fields to the Gaussian parameters rather than relying on static scene embeddings or separate dynamic object tracking [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Ray-point embeddings `rays_pts_emb` (containing 3D coordinates), pre-computed Gaussian attributes `scales_emb`, `rotations_emb`, `opacity_emb`, and `shs_emb` (Spherical Harmonics coefficients), and a scalar time value `time_emb`.
*   **Outputs:** Deformed Gaussian attributes (`means3D_final`, `scales_final`, `opacity_final`, `shs_final`) ready for rasterization.

**Data Structures:**
*   `rays_pts_emb`: Tensor of shape $[N, D_{pts}]$, where $N$ is the number of Gaussian points and $D_{pts}$ includes spatial coordinates and potentially other embeddings.
*   `scales_emb`: Tensor of shape $[N, 3]$.
*   `rotations_emb`: Tensor of shape $[N, 4]$ (quaternions).
*   `opacity_emb`: Tensor of shape $[N, 1]$.
*   `shs_emb`: Tensor of shape $[N, K, 3]$, where $K$ is the number of SH bands (e.g., 16 for degree 2).
*   `time_emb`: Scalar or tensor representing the normalized time $t \in [0, 1]$.

**Parameters:**
*   `args.static_mlp`: Boolean flag. If true, uses a static MLP to compute a mask. [S1, unofficial]
*   `args.empty_voxel`: Boolean flag. If true, uses empty voxel queries for masking. [S1, unofficial]
*   `args.no_dx`: Boolean flag. If false, predicts position deformation. [S1, unofficial]
*   `args.no_ds`: Boolean flag. If false, predicts scale deformation. [S1, unofficial]
*   `args.no_do`: Boolean flag. If false, predicts opacity deformation. [S1, unofficial]
*   `args.no_dshs`: Boolean flag. If false, predicts SH deformation. [S1, unofficial]

**Steps:**
1.  **Query Time-Dependent Features:** Compute a hidden representation by concatenating spatial, scale, rotation, and time embeddings and passing them through a time-query network $\Psi_t$.
    $$ \text{hidden} = \Psi_t(\text{rays\_pts\_emb}, \text{scales\_emb}, \text{rotations\_emb}, \text{time\_feature}, \text{time\_emb}) $$
2.  **Compute Mask:** Determine a mask $m$ to distinguish static from dynamic regions or to handle empty space.
    *   If `args.static_mlp` is true: $m = \sigma(\text{StaticMLP}(\text{hidden}))$. [S1, unofficial]
    *   Else if `args.empty_voxel` is true: $m = \text{EmptyVoxelQuery}(\text{rays\_pts\_emb}[:, :3])$. [S1, unofficial]
    *   Else: $m = \mathbf{1}$ (all ones). [S1, unofficial]
3.  **Deform Position:** If `args.no_dx` is false, predict displacement $\Delta x$ and update means.
    $$ \Delta x = \text{PosDeform}(\text{hidden}) $$
    $$ \mathbf{c}_{new} = \mathbf{c}_{old} \cdot m + \Delta x $$
    [S1, unofficial]
4.  **Deform Scale:** If `args.no_ds` is false, predict scale change $\Delta s$ and update scales.
    $$ \Delta s = \text{ScalesDeform}(\text{hidden}) $$
    $$ \mathbf{s}_{new} = \mathbf{s}_{old} \cdot m + \Delta s $$
    [S1, unofficial]
5.  **Deform Opacity:** If `args.no_do` is false, predict opacity change $\Delta \alpha$ and update opacity.
    $$ \Delta \alpha = \text{OpacityDeform}(\text{hidden}) $$
    $$ \alpha_{new} = \alpha_{old} \cdot m + \Delta \alpha $$
    [S1, unofficial]
6.  **Deform SH Coefficients:** If `args.no_dshs` is false, predict SH changes $\Delta \text{SH}$ and update coefficients.
    $$ \Delta \text{SH} = \text{SHSDeform}(\text{hidden}).\text{reshape}([N, 16, 3]) $$
    $$ \text{SH}_{new} = \text{SH}_{old} \cdot m_{\text{unsqueeze}} + \Delta \text{SH} $$
    [S1, unofficial]
7.  **Rasterize:** Pass the final deformed attributes to the rasterizer.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def deform_gaussians(rays_pts_emb, scales_emb, rotations_emb, opacity_emb, shs_emb, time_emb, args):
    # 1. Query time-dependent features
    # Assumes query_time is a defined MLP or network
    hidden = query_time(rays_pts_emb, scales_emb, rotations_emb, time_emb)
    
    # 2. Compute mask for static vs dynamic regions
    if args.static_mlp:
        mask = static_mlp(hidden)
    elif args.empty_voxel:
        mask = empty_voxel_query(rays_pts_emb[:, :3])
    else:
        mask = torch.ones_like(opacity_emb[:, 0]).unsqueeze(-1)
        
    # 3. Compute position deformation
    if not args.no_dx:
        dx = pos_deform(hidden)
        means3D = rays_pts_emb[:, :3] * mask + dx
    else:
        means3D = rays_pts_emb[:, :3]
        
    # 4. Compute scale deformation
    if not args.no_ds:
        ds = scales_deform(hidden)
        scales = scales_emb[:, :3] * mask + ds
    else:
        scales = scales_emb[:, :3]
        
    # 5. Compute opacity deformation
    if not args.no_do:
        do = opacity_deform(hidden)
        opacity = opacity_emb[:, :1] * mask + do
    else:
        opacity = opacity_emb[:, :1]
        
    # 6. Compute SHS deformation
    if not args.no_dshs:
        dshs = shs_deform(hidden).reshape([shs_emb.shape[0], 16, 3])
        shs = shs_emb * mask.unsqueeze(-1) + dshs
    else:
        shs = shs_emb
        
    # 7. Rasterize
    rendered_image, radii, depth = rasterizer(
        means3D=means3D,
        means2D=None, # screenspace_points computed internally
        shs=shs,
        colors_precomp=None,
        opacities=opacity,
        scales=scales,
        rotations=rotations_emb,
        cov3D_precomp=None
    )
    
    return {
        "render": rendered_image,
        "viewspace_points": None,
        "visibility_filter": radii > 0,
        "radii": radii,
        "depth": depth
    }
```

**Edge Cases and Pitfalls:**
*   **Masking Logic:** The mask multiplication $m$ applies to the base attributes. If $m=0$, the deformation is zeroed out, effectively keeping the attribute static or zeroed. Implementers must ensure $m$ is correctly normalized if it represents a probability. [S1, unofficial]
*   **SH Reshaping:** The SH deformation output must be reshaped to match the input SH tensor dimensions before addition. [S1, unofficial]
*   **Time Normalization:** `time_emb` should be normalized to a consistent range (e.g., $[0, 1]$) across the dataset. [G]

## Subtopics

### [Surface-Preserving Kernel Adaptation](19023-surface-preserving-kernel-adaptation.md)
A method for updating 3D Gaussian kernels after deformation to prevent "spike-like" artifacts by preserving surface coverage without geometric proxies. Choose when naive affine transformations fail to represent the occupancy region.

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of Gaussians $N$ for the deformation network forward pass, plus the standard $O(N)$ rasterization cost. The deformation network query is typically a small MLP.
*   **Memory Complexity:** Requires storing the deformation network weights and the additional deformation fields for position, scale, opacity, and SH. This increases memory usage compared to static 3DGS.
*   **Strengths:** Allows for continuous time interpolation and smooth animation of Gaussian attributes.
*   **Limitations:** Performance depends on the capacity of the deformation network; overfitting can occur if not regularized. [S1, unofficial]
*   **When to use:** When animating dynamic scenes with continuous motion is required.
*   **When not to use:** For static scenes or when computational resources are severely limited.

## Background
The Deformation Network (Dynamic) approach is implemented in the unofficial 4DGS repository by ElenaViewSynthesis. It builds on the principles of dynamic 3D Gaussian Splatting, where time is treated as an additional input to predict attribute variations. [S1, unofficial]

## Sources
[S1] ElenaViewSynthesis/4dgs, https://github.com/ElenaViewSynthesis/4dgs, fetched 2026-10-28 (unofficial)
