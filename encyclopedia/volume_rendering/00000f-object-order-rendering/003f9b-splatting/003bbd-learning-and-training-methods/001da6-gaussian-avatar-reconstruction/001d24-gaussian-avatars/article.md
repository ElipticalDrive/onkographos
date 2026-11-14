# Gaussian Avatars

Also called: FLAME Gaussian Splatting, PSAvatar, PSAvatar / Relightable Avatar, Relightable Avatar

## Overview
Gaussian Avatars is a method for reconstructing photorealistic, animatable head avatars by anchoring 3D Gaussians to a parametric FLAME mesh [S1][S3, unofficial]. It solves the challenge of maintaining topological consistency and rigging capabilities in dynamic head reconstruction by constraining each Gaussian to its corresponding mesh triangle [S1][S3, unofficial]. Unlike methods that predict Gaussian parameters via MLPs conditioned on pose [S1][S3, unofficial], Gaussian Avatars optimizes Gaussians directly while enforcing geometric constraints, enabling high-fidelity rendering and relighting [S2, unofficial].

## How it works
**Inputs and Outputs:**
- **Inputs:** Multi-view video of a talking head; pre-fitted FLAME model parameters (pose, expression, shape) [S2, unofficial].
- **Outputs:** A set of 3D Gaussians associated with FLAME mesh triangles, optimized for appearance and geometry [S1][S3, unofficial].

**Data Structures:**
- **Gaussian:** Each Gaussian $i$ is associated with a specific triangle $t$ of the FLAME mesh.
- **Attributes:** Position $\mathbf{p}_i$, covariance $\mathbf{\Sigma}_i$, opacity $\alpha_i$, spherical harmonics coefficients $\mathbf{c}_i$ [S1][S3, unofficial].
- **Constraint:** The Gaussian position $\mathbf{p}_i$ must lie within the barycentric coordinates of its associated triangle $t$ [S1][S3, unofficial].

**Steps:**
1. **Initialization:** Fit a FLAME model to the input video frames to obtain initial mesh parameters [S2, unofficial].
2. **Gaussian Association:** Initialize one Gaussian per triangle of the FLAME mesh [S1][S3, unofficial]. The initial position is set to the triangle centroid or a point within the triangle [S1][S3, unofficial].
3. **Constrained Optimization:** Optimize Gaussian parameters (position, covariance, opacity, SH coefficients) using differentiable rendering [S1][S3, unofficial].
   - **Position Constraint:** The position $\mathbf{p}_i$ is parameterized using barycentric coordinates $(u, v, w)$ within the triangle $t$:
     $$ \mathbf{p}_i = u \mathbf{v}_1 + v \mathbf{v}_2 + w \mathbf{v}_3 $$
     where $\mathbf{v}_1, \mathbf{v}_2, \mathbf{v}_3$ are the vertices of triangle $t$, and $u+v+w=1, u,v,w \ge 0$ [S1][S3, unofficial].
   - **Deformation:** The FLAME mesh is deformed according to the current pose and expression parameters. The Gaussians are deformed rigidly with the mesh [S1][S3, unofficial].
4. **Rendering:** Render the scene using standard 3D Gaussian Splatting rasterization [S1][S3, unofficial].
5. **Loss Function:** Minimize photometric loss ($\mathcal{L}_{L1}$ or $\mathcal{L}_{MSE}$) and perceptual loss ($\mathcal{L}_{MS-SSIM}$) between rendered and ground truth images [S1][S3, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Initialize FLAME mesh and parameters
flame_mesh = load_flame_model()
pose_params = fit_flame_to_video(video_frames)

# Initialize Gaussians per triangle
gaussians = []
for triangle in flame_mesh.triangles:
    gaussian = Gaussian()
    gaussian.position = triangle.centroid()
    gaussian.covariance = identity_matrix()
    gaussian.opacity = 1.0
    gaussian.sh_coefficients = random_vector()
    gaussian.triangle_id = triangle.id
    gaussians.append(gaussian)

# Training loop
for frame in video_frames:
    # Get current FLAME mesh with pose/expression
    current_mesh = flame_mesh.deform(pose_params[frame])
    
    # Update Gaussian positions based on mesh deformation
    for gaussian in gaussians:
        triangle = current_mesh.get_triangle(gaussian.triangle_id)
        # Barycentric coordinates are optimized or fixed?
        # Sources imply Gaussians are constrained to remain within triangle
        # Optimization updates barycentric coords or position directly with constraint
        gaussian.position = triangle.get_point(gaussian.barycentric_coords)
        
    # Render
    rendered_image = render_gaussians(gaussians, camera_params[frame])
    
    # Compute loss
    loss = photometric_loss(rendered_image, frame) + perceptual_loss(rendered_image, frame)
    
    # Backpropagate and update Gaussian parameters
    update_gaussians(gaussians, loss)
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
- **Strengths:** Maintains topological consistency; riggable; photorealistic [S1][S2, S3, unofficial].
- **Limitations:** Limited to head avatars; requires FLAME fitting [S1][S2, S3, unofficial].
- **When to use:** When high-fidelity, animatable head avatars are needed [S2, unofficial].
- **When not:** For full-body avatars or scenes without a parametric head model [S1][S3, unofficial].

## Background
Gaussian Avatars was introduced as a method to reconstruct human avatars using 3D Gaussian Splatting [S1][S3, unofficial]. It addresses challenges such as handling dynamic Gaussians and supporting relighting [S1][S3, unofficial]. The method trains a FLAME model of the bust, associating each triangle with a Gaussian [S1][S3, unofficial]. It is also known as PSAvatar or Relightable Avatar [S2, unofficial].

## Sources
[S1] From Volume Rendering to 3D Gaussian Splatting: Theory and Applications, https://doi.org/10.1109/sibgrapi67909.2025.11223276, fetched 2026-10-26
[S2] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S3] From Volume Rendering to 3D Gaussian Splatting, https://www.visgraf.impa.br/Data/RefBib/PS_PDF/tutorial-sib2025/tutorial-sib2025.pdf, fetched 2026-10-28 (unofficial)
