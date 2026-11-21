# Gaussian Avatars

Also called: FLAME Gaussian Splatting, PSAvatar, PSAvatar / Relightable Avatar, Relightable Avatar

## Overview
Gaussian Avatars is a method for reconstructing photorealistic, animatable head avatars by anchoring 3D Gaussians to a parametric FLAME mesh [S1][S3, unofficial]. It associates each FLAME mesh triangle with a Gaussian that is optimized for the scene while being constrained to remain within its corresponding triangle [S1][S3, unofficial]. Unlike methods that predict Gaussian parameters via feedforward networks conditioned on pose [S1], Gaussian Avatars optimizes the Gaussian ellipsoid parameters directly while enforcing geometric constraints, enabling high-fidelity rendering and relighting [S2, unofficial].

## How it works
**Inputs and Outputs:**
- **Inputs:** Multi-view video of a talking head; pre-fitted FLAME model parameters (pose, expression, shape) [S2, unofficial].
- **Outputs:** A set of 3D Gaussians associated with FLAME mesh triangles, optimized for appearance and geometry [S1][S3, unofficial].

**Data Structures:**
- **Gaussian:** Each Gaussian $i$ is associated with a specific triangle $t$ of the FLAME mesh [S1][S3, unofficial].
- **Constraint:** The Gaussian position $\mathbf{p}_i$ must lie within the barycentric coordinates of its associated triangle $t$ [S1][S3, unofficial].

**Steps:**
1. **Initialization:** Fit a FLAME model to the input video frames to obtain initial mesh parameters [S2, unofficial].
2. **Gaussian Association:** Initialize one Gaussian per triangle of the FLAME mesh [S1][S3, unofficial].
3. **Constrained Optimization:** Optimize Gaussian parameters using differentiable rendering [S1][S3, unofficial]. The position $\mathbf{p}_i$ is constrained to remain within the barycentric coordinates of triangle $t$ [S1][S3, unofficial]. The FLAME mesh deforms according to the current pose and expression parameters, and the Gaussians follow the deformation while maintaining their triangle constraints [S1][S4, unofficial].
4. **Rendering:** Render the scene using standard 3D Gaussian Splatting rasterization [S1][S3, unofficial].
5. **Loss Function:** Minimize photometric loss ($\mathcal{L}_{L1}$) and perceptual loss ($\mathcal{L}_{MS-SSIM}$) between rendered and ground truth images [S1][S3, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def train_gaussian_avatars(video_frames, camera_params, flame_mesh_init):
    # Fit FLAME model to video frames to obtain pose and expression parameters [G]
    flame_params = [fit_flame_to_frame(frame) for frame in video_frames]

    # Initialize one Gaussian per FLAME mesh triangle [S1][S3, unofficial]
    gaussians = []
    for tri in flame_mesh_init.triangles:
        gaussians.append({
            'position': tri.centroid(),
            'covariance': np.eye(3),
            'opacity': 1.0,
            'sh_coefficients': np.zeros(48),
            'triangle_id': tri.id,
            'barycentric_coords': np.array([1.0, 0.0, 0.0])
        })

    # Training loop
    for frame_idx, frame in enumerate(video_frames):
        cam = camera_params[frame_idx]
        params = flame_params[frame_idx]

        # Deform FLAME mesh with current pose/expression parameters [G]
        current_mesh = deform_flame_mesh(flame_mesh_init, params)

        # Update Gaussian positions constrained to their associated triangles [S1][S3, unofficial]
        for g in gaussians:
            tri = current_mesh.triangles[g['triangle_id']]
            v0, v1, v2 = tri.vertices
            g['position'] = (g['barycentric_coords'][0] * v0 +
                             g['barycentric_coords'][1] * v1 +
                             g['barycentric_coords'][2] * v2)

        # Render scene using standard 3D Gaussian Splatting rasterization [S1][S3, unofficial]
        rendered_image = rasterize_gaussians(gaussians, cam)

        # Compute photometric and perceptual losses [S1][S3, unofficial]
        loss = l1_loss(rendered_image, frame) + ms_ssim_loss(rendered_image, frame)

        # Backpropagate gradients and update Gaussian parameters [S1][S3, unofficial]
        optimizer_step(gaussians, loss)

    return gaussians, flame_params
```

**Parameters:**
- Learning rate: Not given in the sources.
- Number of Gaussians: One per triangle of the FLAME mesh [S1][S3, unofficial].

**Edge Cases:**
- Degenerate triangles: Ensure barycentric coordinates are valid [G].
- Precision: Barycentric coordinate optimization must maintain non-negativity and sum-to-one constraint [G].

## Subtopics

### [Head Avatar Modeling](11813-head-avatar-modeling.md)
A method that combines parametric morphable face models (e.g., FLAME) with deformable 3D Gaussians, employing diffusion strategies and expression-aware deformation fields to disentangle rigid head poses from non-rigid facial movements. It sets itself apart from sibling methods by leveraging the topological consistency and editability of parametric meshes to solve the cross-frame correspondence and topology flexibility challenges inherent in unstructured Gaussian splatting for dynamic faces.

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Proportional to the number of triangles in the FLAME mesh [S1][S3, unofficial].
- **Measured Performance:** Not given in the sources.
- **Strengths:** Riggable; photorealistic head rendering [S2, unofficial].
- **Limitations:** Limited to head avatars; requires FLAME fitting [S1][S2, unofficial].
- **When to use:** When high-fidelity, animatable head avatars are needed [S2, unofficial].
- **When not:** For full-body avatars or scenes without a parametric head model [S1][S3, unofficial].

## Background
Gaussian Avatars was introduced to reconstruct human avatars using 3D Gaussian Splatting [S1][S3, unofficial]. It addresses challenges such as handling dynamic Gaussians and supporting relighting [S1][S3, unofficial]. The method trains a FLAME model of the bust, associating each triangle with a Gaussian [S1][S3, unofficial]. It is also known as PSAvatar or Relightable Avatar [S2, unofficial].

## Sources
[S1] From Volume Rendering to 3D Gaussian Splatting: Theory and Applications, https://doi.org/10.1109/sibgrapi67909.2025.11223276, fetched 2026-10-26
[S2] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S3] From Volume Rendering to 3D Gaussian Splatting, https://www.visgraf.impa.br/Data/RefBib/PS_PDF/tutorial-sib2025/tutorial-sib2025.pdf, fetched 2026-10-28 (unofficial)
[S4] PDF Recent advances in 3D Gaussian splatting - Springer, https://link.springer.com/content/pdf/10.1007/s41095-024-0436-y.pdf, fetched 2026-10-29 (unofficial)
