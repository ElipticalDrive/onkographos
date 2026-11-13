# TSDF Reconstruction from Depth Maps

Also called: Depth Map Fusion, TSDF Reconstruction, TSDF Volume Integration and Mesh Extraction

## Overview
TSDF Reconstruction from Depth Maps is a method to reconstruct a 3D mesh from multi-view depth images by integrating them into a Truncated Signed Distance Function (TSDF) volume and extracting the surface via marching cubes [S1, unofficial][S2]. It converts per-surfel depth outputs into a unified 3D mesh by computing a robust depth value per pixel and fusing the resulting depth maps via TSDF [S2]. It differs from direct neural rendering mesh extraction by relying on explicit geometric fusion rather than differentiable surface optimization [S1, unofficial].

## How it works
**Inputs:**
*   A set of $N$ depth maps $D_i$ and optionally color maps $C_i$ for $i \in \{1, \dots, N\}$.
*   Camera poses $P_i$ (World-to-Camera transformation matrices, $4 \times 4$) and intrinsic matrices $K_i$ ($3 \times 3$).
*   An Axis-Aligned Bounding Box (AABB) defined by `aabb[0]` (min corner) and `aabb[1]` (max corner).
*   Volume dimensions `volume_dims` = $(X, Y, Z)$.
*   Truncation margin parameter `truncation_margin`.

**Outputs:**
*   A 3D mesh (vertices, faces, normals, colors).

**Data Structures:**
The TSDF volume is stored as a 3D grid with the following fields:
*   `voxel_coords`: Tensor of shape `[3, X, Y, Z]` storing the world-space coordinates of each voxel center.
*   `values`: Tensor of shape `[X, Y, Z]`, initialized to $-1$. Stores the integrated TSDF values.
*   `weights`: Tensor of shape `[X, Y, Z]`, initialized to $0$. Stores the integration weights.
*   `colors`: Tensor of shape `[X, Y, Z, 3]`, initialized to $0$. Stores the integrated colors (optional).
*   `voxel_size`: Tensor of shape `[3]`.
*   `origin`: Tensor of shape `[3]`, equal to `aabb[0]`.

**Parameters:**
*   `voxel_size`: The physical size of a voxel. Typical value: $0.004$ [S2].
*   `truncation_threshold`: The distance beyond which the TSDF value is clamped. Typical value: $0.02$ [S2].
*   `truncation_margin`: A multiplier for voxel size to determine truncation. $t = \text{voxel\_size}[0] \times \text{truncation\_margin}$ [S1, unofficial].

**Algorithm Steps:**

1.  **Initialize Volume:**
    1.  Set `origin = aabb[0]`.
    2.  Calculate `voxel_size = (aabb[1] - aabb[0]) / volume_dims`.
    3.  Create voxel coordinates using a meshgrid with 'ij' indexing:
        $$ \text{grid} = \text{meshgrid}(range(X), range(Y), range(Z), indexing='ij') $$
        $$ \text{voxel\_coords} = \text{origin}.view(3, 1, 1, 1) + \text{grid} \times \text{voxel\_size}.view(3, 1, 1, 1) $$
    4.  Initialize `values` to $-1$, `weights` to $0$, and `colors` to $0$.

2.  **Compute Robust Depth Maps:**
    For each view $i$, if the input depth is derived from splats (e.g., Gaussian Splatting), compute the median depth $t_{median}$ for each pixel by selecting the maximum depth $t_k$ among all splats $k$ where the transmittance $T_k$ exceeds $0.5$ [S2]:
    $$ t_{median} = \max \{ t_k \mid T_k > 0.5 \} $$
    If standard depth maps are provided, use them directly.

3.  **Project Voxels to Image Space:**
    For each view $i$:
    1.  Reshape `voxel_coords` to `[3, -1]` (flattened spatial dimensions).
    2.  Make homogeneous by appending ones:
        $$ \mathbf{v}_{world} = [\mathbf{v}_{coords}; 1] $$
    3.  Expand to batch dimension $[N, 4, M]$ where $M = X \cdot Y \cdot Z$.
    4.  Transform to camera space using the inverse pose $P_i^{-1}$:
        $$ \mathbf{v}_{cam} = P_i^{-1} \mathbf{v}_{world} $$
    5.  Flip Z and Y axes to match camera coordinate conventions (if necessary):
        $$ \mathbf{v}_{cam}[1, :] = -\mathbf{v}_{cam}[1, :] $$
        $$ \mathbf{v}_{cam}[2, :] = -\mathbf{v}_{cam}[2, :] $$
    6.  Calculate depth (distance from camera center):
        $$ d_{voxel} = \|\mathbf{v}_{cam}[0:3, :]\|_2 $$
    7.  Project to pixel coordinates:
        $$ \mathbf{v}_{cam\_points} = K_i \frac{\mathbf{v}_{cam}[0:3, :]}{d_{voxel}} $$
        $$ \mathbf{u}_{pixel} = \mathbf{v}_{cam\_points}[0:2, :] $$

4.  **Sample Depth and Color:**
    1.  Normalize pixel coordinates $\mathbf{u}_{pixel}$ to $[-1, 1]$ range relative to image dimensions $(H_i, W_i)$:
        $$ \mathbf{u}_{norm} = 2.0 \frac{\mathbf{u}_{pixel}}{[W_i, H_i]} - 1.0 $$
    2.  Sample the depth map $D_i$ and color map $C_i$ at $\mathbf{u}_{norm}$ using nearest-neighbor interpolation with zero padding:
        $$ d_{sampled} = \text{grid\_sample}(D_i, \mathbf{u}_{norm}, \text{mode}='nearest', \text{padding\_mode}='zeros') $$
        $$ c_{sampled} = \text{grid\_sample}(C_i, \mathbf{u}_{norm}, \text{mode}='nearest', \text{padding\_mode}='zeros') $$

5.  **Integrate TSDF:**
    For each view $i$ (or batch of views):
    1.  Identify valid voxels using the validity mask:
        $$ valid = (d_{voxel} > 0) \land (d_{sampled} > 0) \land (d_{sampled} - d_{voxel} > -t) $$
        where $t$ is the truncation threshold [S1, unofficial]. Note: Ray directions must avoid division by zero when computing intersection fractions by adding epsilon [S1, unofficial].
    2.  For valid voxels, retrieve old values $v_{old}$ and weights $w_{old}$.
    3.  Set new weight $w_{new} = 1.0$.
    4.  Calculate total weight:
        $$ w_{total} = w_{old} + w_{new} $$
    5.  Update TSDF value:
        $$ v_{new} = \frac{v_{old} w_{old} + v_{sampled} w_{new}}{w_{total}} $$
        where $v_{sampled}$ is the signed distance from the voxel to the sampled depth surface, clamped to $[-t, t]$.
    6.  Clamp weight:
        $$ w_{total} = \min(w_{total}, 1.0) $$
    7.  If colors exist, update color similarly:
        $$ c_{new} = \frac{c_{old} w_{old} + c_{sampled} w_{new}}{w_{total}} $$
    8.  Store $v_{new}, w_{total}, c_{new}$ back to the volume.

6.  **Extract Mesh:**
    1.  Clamp TSDF values to $[-1, 1]$ and convert to numpy array.
    2.  Run Marching Cubes algorithm with `level=0` and `allow_degenerate=False` to extract vertices and faces [S1, unofficial].
    3.  Map vertices to voxel indices and retrieve colors from the `colors` tensor.
    4.  Transform vertices back to world space:
        $$ \mathbf{v}_{world} = \text{origin} + \mathbf{v}_{voxel} \times \text{voxel\_size} $$
    5.  Return Mesh object with vertices, faces, normals, and colors.

**Pitfalls:**
*   Mask images are not supported; passing `mask_images` raises `NotImplementedError` [S1, unofficial].
*   High geometric fitting capability of quadric surfels can cause overfitting in regions with sparse views or low texture, resulting in surface artifacts like dents [S2].
*   Ray directions must avoid division by zero when computing intersection fractions by adding epsilon [S1, unofficial].

## Subtopics

### [Progressive TSDF Fusion & Mesh Extraction](10786-progressive-tsdf-fusion-mesh-extraction.md)
Progressive TSDF Fusion & Mesh Extraction is a multi-resolution surface reconstruction method that fuses rendered depth and alpha maps into a Truncated Signed Distance Field (TSDF) grid, progressively refining the voxel resolution while pruning empty space, and finally extracts a mesh via Marching Cubes. It differs from single-resolution TSDF fusion by reducing memory usage and computational cost through early pruning of empty voxels and subdividing only where necessary, and it differs from direct neural rendering mesh extraction by relying on explicit geometric fusion rather than differentiable surface optimization.

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of voxels and number of views for integration. Marching cubes is generally efficient but depends on surface complexity.
*   **Memory Complexity:** Proportional to the volume resolution $X \cdot Y \cdot Z$. For a voxel size of $0.004$ and a scene bounding box of typical size, memory usage can be significant [S2].
*   **Performance:** On the TNT dataset, full QGS (using this method) achieves a mean F1-Score of $0.50$ with a training time of $75$ minutes [S2]. On Mip-NeRF 360 indoor scenes, it achieves PSNR $30.45$, SSIM $0.919$, and LPIPS $0.184$ [S2]. On Mip-NeRF 360 outdoor scenes, it achieves PSNR $24.32$, SSIM $0.706$, and LPIPS $0.242$ [S2].
*   **Strengths:** Robust to noise via median depth selection and TSDF fusion. Produces watertight meshes.
*   **Limitations:** Requires careful tuning of voxel size and truncation threshold. Sensitive to overfitting in sparse regions [S2].
*   **When to use:** When a watertight mesh is required from multi-view depth data, especially when differentiable surface optimization is too slow or unstable.
*   **When not to use:** When high-frequency details are lost due to voxel discretization, or when the scene is extremely large and memory is constrained (consider progressive fusion instead).

## Background
The method originates from the integration of depth maps into a TSDF volume, a technique widely used in 3D reconstruction [S1, unofficial][S2]. It is commonly used in conjunction with Gaussian Splatting variants, such as Quadratic Gaussian Splatting (QGS), to extract high-quality meshes from 2D Gaussian primitives [S2]. The specific implementation details described here are derived from the SDFStudio repository and the QGS paper [S1, unofficial][S2].

## Sources
[S1] autonomousvision/sdfstudio, https://github.com/autonomousvision/sdfstudio, fetched 2026-10-26 (unofficial)
[S2] Quadratic Gaussian Splatting: High Quality Surface Reconstruction with Second-Order Geometric Primitives, https://doi.org/10.1109/ICCV51701.2025.02624, fetched 2026-10-30
