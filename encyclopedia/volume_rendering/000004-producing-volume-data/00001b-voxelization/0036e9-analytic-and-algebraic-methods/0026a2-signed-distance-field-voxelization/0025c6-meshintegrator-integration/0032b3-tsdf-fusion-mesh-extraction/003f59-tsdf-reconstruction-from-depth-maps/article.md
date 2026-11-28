# TSDF Reconstruction from Depth Maps

Also called: Depth Map Fusion, TSDF Fusion for Mesh Extraction, TSDF Reconstruction, TSDF Volume Integration and Mesh Extraction

## Overview
TSDF Reconstruction from Depth Maps is a method to reconstruct a 3D mesh from multi-view depth images by integrating them into a Truncated Signed Distance Function (TSDF) volume and extracting the surface via marching cubes [S1, unofficial][S2, unofficial]. It converts per-surfel depth outputs into a unified 3D mesh by computing a robust depth value per pixel and fusing the resulting depth maps via TSDF [S3]. It differs from direct neural rendering mesh extraction by relying on explicit geometric fusion rather than differentiable surface optimization [S1, unofficial].

## How it works
**Inputs:** [S1, unofficial] A set of $N$ depth maps $D_i$ and optionally color maps $C_i$ for $i \in \{1, \dots, N\}$. Camera poses $P_i$ (World-to-Camera transformation matrices, $4 \times 4$) and intrinsic matrices $K_i$ ($3 \times 3$) [S1, unofficial]. An Axis-Aligned Bounding Box (AABB) defined by `aabb[0]` (min corner) and `aabb[1]` (max corner) [S1, unofficial]. Volume dimensions `volume_dims` = $(X, Y, Z)$ [S1, unofficial]. Truncation margin parameter `truncation_margin` [S1, unofficial].

**Outputs:** [G] A 3D mesh (vertices, faces, normals, colors).

**Data Structures:** [S1, unofficial] The TSDF volume is stored as a 3D grid with the following fields: `voxel_coords` (TensorType[3, X, Y, Z]), `values` (TensorType[X, Y, Z], initialized to -1), `weights` (TensorType[X, Y, Z], initialized to 0), `colors` (TensorType[X, Y, Z, 3], initialized to 0), `voxel_size` (TensorType[3]), `origin` (TensorType[3]).

**Parameters:** [S3] `voxel_size`: 0.004. `truncation_threshold`: 0.02. [S1, unofficial] `truncation_margin`: multiplier for voxel size. Truncation distance $t = \text{voxel\_size}[0] \times \text{truncation\_margin}$ [S1, unofficial].

**Algorithm Steps:**
1. **Initialize Volume:** [S1, unofficial] Set `origin = aabb[0]`. Calculate `voxel_size = (aabb[1] - aabb[0]) / volume_dims` [S1, unofficial]. Create voxel coordinates using a meshgrid with 'ij' indexing: $\text{grid} = \text{meshgrid}(\text{range}(X), \text{range}(Y), \text{range}(Z), \text{indexing}='ij')$ [S1, unofficial]. Compute $\text{voxel\_coords} = \text{origin}.view(3, 1, 1, 1) + \text{grid} \times \text{voxel\_size}.view(3, 1, 1, 1)$ [S1, unofficial]. Initialize `values` to -1, `weights` to 0, and `colors` to 0 [S1, unofficial].
2. **Compute Robust Depth Maps:** [S3] For each view $i$, if the input depth is derived from splats, compute the median depth $t_{median}$ for each pixel by selecting the maximum depth $t_k$ among all splats $k$ where the transmittance $T_k$ exceeds 0.5: $t_{median} = \max \{ t_k \mid T_k > 0.5 \}$ [S3]. If standard depth maps are provided, use them directly [G].
3. **Project Voxels to Image Space:** [S1, unofficial] For each view $i$: Reshape `voxel_coords` to `[3, -1]`. Make homogeneous by appending ones: $\mathbf{v}_{world} = [\mathbf{v}_{coords}; 1]$ [S1, unofficial]. Expand to batch dimension $[N, 4, M]$ where $M = X \cdot Y \cdot Z$ [S1, unofficial]. Transform to camera space using the inverse pose $P_i^{-1}$: $\mathbf{v}_{cam} = P_i^{-1} \mathbf{v}_{world}$ [S1, unofficial]. Flip Z and Y axes: $\mathbf{v}_{cam}[1, :] = -\mathbf{v}_{cam}[1, :]$ and $\mathbf{v}_{cam}[2, :] = -\mathbf{v}_{cam}[2, :]$ [S1, unofficial]. Calculate depth: $d_{voxel} = \|\mathbf{v}_{cam}[0:3, :]\|_2$ [S1, unofficial]. Project to pixel coordinates: $\mathbf{v}_{cam\_points} = K_i \frac{\mathbf{v}_{cam}[0:3, :]}{\mathbf{v}_{cam}[2, :]}$ and $\mathbf{u}_{pixel} = \mathbf{v}_{cam\_points}[0:2, :]$ [S1, unofficial].
4. **Sample Depth and Color:** [S1, unofficial] Normalize pixel coordinates $\mathbf{u}_{pixel}$ to $[-1, 1]$ range relative to image dimensions $(H_i, W_i)$: $\mathbf{u}_{norm} = 2.0 \frac{\mathbf{u}_{pixel}}{[W_i, H_i]} - 1.0$. Sample the depth map $D_i$ and color map $C_i$ at $\mathbf{u}_{norm}$ using nearest-neighbor interpolation with zero padding: $d_{sampled} = \text{grid\_sample}(D_i, \mathbf{u}_{norm}, \text{mode}='nearest', \text{padding\_mode}='zeros')$ and $c_{sampled} = \text{grid\_sample}(C_i, \mathbf{u}_{norm}, \text{mode}='nearest', \text{padding\_mode}='zeros')$ [S1, unofficial].
5. **Integrate TSDF:** [S1, unofficial] For each view $i$: Identify valid voxels using the validity mask: $valid = (d_{voxel} > 0) \land (d_{sampled} > 0) \land (d_{sampled} - d_{voxel} > -t)$, where $t = \text{voxel\_size}[0] \times \text{truncation\_margin}$ [S1, unofficial]. For valid voxels, retrieve old values $v_{old}$ and weights $w_{old}$ [S1, unofficial]. Set new weight $w_{new} = 1.0$. Calculate total weight: $w_{total} = w_{old} + w_{new}$ [S1, unofficial]. Update TSDF value: $v_{new} = \frac{v_{old} w_{old} + v_{sampled} w_{new}}{w_{total}}$, where $v_{sampled}$ is the signed distance from the voxel to the sampled depth surface, clamped to $[-t, t]$ [S1, unofficial]. Clamp weight: $w_{total} = \min(w_{total}, 1.0)$ [S1, unofficial]. If colors exist, update color similarly: $c_{new} = \frac{c_{old} w_{old} + c_{sampled} w_{new}}{w_{total}}$ [S1, unofficial]. Store $v_{new}, w_{total}, c_{new}$ back to the volume.
6. **Extract Mesh:** [S1, unofficial] Clamp TSDF values to $[-1, 1]$ and convert to numpy array. Run Marching Cubes algorithm with `level=0` and `allow_degenerate=False` to extract vertices and faces. Map vertices to voxel indices and retrieve colors from the `colors` tensor. Transform vertices back to world space: $\mathbf{v}_{world} = \text{origin} + \mathbf{v}_{voxel} \times \text{voxel\_size}$. Return Mesh object with vertices, faces, normals, and colors [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn.functional as F
from skimage import measure

def tsdf_reconstruct(depth_maps, color_maps, poses, intrinsics, aabb, volume_dims, truncation_margin):
    origin = aabb[0]
    voxel_size = (aabb[1] - aabb[0]) / volume_dims
    t = voxel_size[0] * truncation_margin

    X, Y, Z = volume_dims
    grid = torch.meshgrid(torch.arange(X), torch.arange(Y), torch.arange(Z), indexing='ij')
    grid = torch.stack(grid, dim=0) # [3, X, Y, Z]
    voxel_coords = origin.view(3, 1, 1, 1) + grid * voxel_size.view(3, 1, 1, 1)

    values = torch.full((X, Y, Z), -1.0)
    weights = torch.zeros((X, Y, Z))
    colors = torch.zeros((X, Y, Z, 3))

    for i in range(len(depth_maps)):
        # Project voxels to image space
        v_world = voxel_coords.reshape(3, -1)
        v_world = torch.cat([v_world, torch.ones(1, v_world.shape[1])], dim=0) # [4, M]
        v_world = v_world.unsqueeze(0) # [1, 4, M]

        # Transform to camera space
        v_cam = torch.bmm(torch.inverse(poses[i]).unsqueeze(0), v_world).squeeze(0) # [4, M]
        v_cam[1, :] = -v_cam[1, :] # Flip Y
        v_cam[2, :] = -v_cam[2, :] # Flip Z

        # Calculate depth
        d_voxel = torch.sqrt(torch.sum(v_cam[:3, :] ** 2, dim=0, keepdim=True)) # [1, M]

        # Project to pixel coordinates
        v_cam_points = torch.bmm(intrinsics[i].unsqueeze(0), (v_cam[:3, :] / v_cam[2, :].unsqueeze(0)).unsqueeze(0)).squeeze(0) # [3, M]
        u_pixel = v_cam_points[:2, :] # [2, M]

        # Normalize to [-1, 1]
        H, W = depth_maps[i].shape[0], depth_maps[i].shape[1]
        u_norm = 2.0 * u_pixel / torch.tensor([W, H]).view(1, 1, 2) - 1.0 # [2, M]
        u_norm = u_norm.permute(1, 0).unsqueeze(0).unsqueeze(0) # [1, M, 1, 2] for grid_sample

        # Sample depth and color
        d_sampled = F.grid_sample(depth_maps[i].unsqueeze(0).unsqueeze(0), u_norm,
                                  mode='nearest', padding_mode='zeros', align_corners=False).squeeze() # [M]
        c_sampled = F.grid_sample(color_maps[i].unsqueeze(0), u_norm,
                                  mode='nearest', padding_mode='zeros', align_corners=False).squeeze() # [M, 3]

        # Validity mask
        valid = (d_voxel.squeeze(0) > 0) & (d_sampled > 0) & (d_sampled - d_voxel.squeeze(0) > -t)

        # Integrate TSDF
        v_old = values[valid]
        w_old = weights[valid]
        w_new = torch.ones_like(w_old)
        w_total = torch.clamp(w_old + w_new, max=1.0)

        v_sampled = torch.clamp(d_sampled[valid] - d_voxel.squeeze(0)[valid], min=-t, max=t)
        v_new = (v_old * w_old + v_sampled * w_new) / w_total
        values[valid] = v_new
        weights[valid] = w_total

        if color_maps is not None:
            c_old = colors[valid]
            c_new = (c_old * w_old.unsqueeze(1) + c_sampled[valid] * w_new.unsqueeze(1)) / w_total.unsqueeze(1)
            colors[valid] = c_new

    # Extract mesh
    values_clamped = torch.clamp(values, -1.0, 1.0)
    mesh_vertices, mesh_faces, _, _ = measure.marching_cubes(values_clamped.numpy(), level=0, allow_degenerate=False)
    mesh_colors = colors[mesh_vertices[:, 0], mesh_vertices[:, 1], mesh_vertices[:, 2]]
    mesh_vertices_world = origin + mesh_vertices * voxel_size
    return Mesh(vertices=mesh_vertices_world, faces=mesh_faces, colors=mesh_colors)
```

## Subtopics

### [Progressive TSDF Fusion & Mesh Extraction](10786-progressive-tsdf-fusion-mesh-extraction.md)
Progressive TSDF Fusion & Mesh Extraction is a multi-resolution surface reconstruction method that fuses rendered depth and alpha maps into a Truncated Signed Distance Field (TSDF) grid, progressively refining the voxel resolution while pruning empty space, and finally extracts a mesh via Marching Cubes. It differs from single-resolution TSDF fusion by reducing memory usage and computational cost through early pruning of empty voxels and subdividing only where necessary, and it differs from direct neural rendering mesh extraction by relying on explicit geometric fusion rather than differentiable surface optimization.

### Volumetric Fusion for Surface Reconstruction
A surface reconstruction method circumventing dependency on Gaussian normals by rendering median depth and fusing it into a mesh using VDBFusion [S4, unofficial]. It is scalable, memory efficient, and time efficient for large-scale scenes [S4, unofficial]. Steps: 1. Render the median depth. 2. Fuse the median depth into a mesh using VDBFusion [S4, unofficial]. Parameters: VDBFusion parameters: voxel size = $r/256$, SDF truncation value = $r/64$, space carving = False, where $r$ is the radius of the bounding sphere of the input GS [S4, unofficial].

## Operating characteristics
*   **Time Complexity:** [G] Linear with respect to the number of voxels and number of views for integration. Marching cubes is generally efficient but depends on surface complexity.
*   **Memory Complexity:** [G] Proportional to the volume resolution $X \cdot Y \cdot Z$.
*   **Performance:** On the TNT dataset, full QGS achieves a mean F1-Score of 0.50 with a training time of 75 minutes [S3]. On Mip-NeRF 360 indoor scenes, full QGS achieves PSNR 30.45, SSIM 0.919, and LPIPS 0.184 [S3]. On Mip-NeRF 360 outdoor scenes, full QGS achieves PSNR 24.32, SSIM 0.706, and LPIPS 0.242 [S3].
*   **Strengths:** Robust to noise via median depth selection and TSDF fusion. Produces watertight meshes.
*   **Limitations:** Requires careful tuning of voxel size and truncation threshold. Sensitive to overfitting in sparse regions [S2, unofficial].
*   **When to use:** [G] When a watertight mesh is required from multi-view depth data, especially when differentiable surface optimization is too slow or unstable.
*   **When not to use:** [G] When high-frequency details are lost due to voxel discretization, or when the scene is extremely large and memory is constrained (consider progressive fusion instead).

## Background
The method originates from the integration of depth maps into a TSDF volume, a technique widely used in 3D reconstruction [S1, unofficial][S2, unofficial]. It is commonly used in conjunction with Gaussian Splatting variants, such as Quadratic Gaussian Splatting (QGS), to extract high-quality meshes from 2D Gaussian primitives [S3]. The specific implementation details described here are derived from the SDFStudio repository and the QGS paper [S1, unofficial][S3].

## Sources
[S1] autonomousvision/sdfstudio, https://github.com/autonomousvision/sdfstudio, fetched 2026-10-26 (unofficial)
[S2] PDF Recent advances in 3D Gaussian splatting - Springer, https://link.springer.com/content/pdf/10.1007/s41095-024-0436-y.pdf, fetched 2026-10-29 (unofficial)
[S3] Quadratic Gaussian Splatting: High Quality Surface Reconstruction with Second-Order Geometric Primitives, https://doi.org/10.1109/ICCV51701.2025.02624, fetched 2026-10-30
[S4] GauStudio: A Modular Framework for 3D Gaussian Splatting and Beyond, https://doi.org/10.48550/arXiv.2403.19632, fetched 2026-10-30 (unofficial)
