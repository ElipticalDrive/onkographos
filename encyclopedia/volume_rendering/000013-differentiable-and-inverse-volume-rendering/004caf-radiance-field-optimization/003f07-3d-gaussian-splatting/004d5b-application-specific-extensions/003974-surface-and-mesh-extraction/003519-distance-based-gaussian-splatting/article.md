# Distance-based Gaussian Splatting

Also called: Mesh-based Gaussian Splatting, Tightly/Loosely-bound Gaussian Splatting

## Overview
Distance-based Gaussian Splatting (MeshGS) constructs a high-fidelity radiance field by establishing a topological connection between a pre-existing large-scale mesh and 3D Gaussian splats [S1]. It solves the problem of redundant memory usage and geometric artifacts inherent in standard 3D Gaussian Splatting when applied to scenes with complex or noisy mesh reconstructions [S1]. Unlike sibling methods that rely on opacity fields or segmentation, this method explicitly classifies splats as "tightly-bound" or "loosely-bound" based on their distance to the mesh surface, allowing for adaptive alignment and occlusion culling [S1].

## How it works
**Inputs:**
*   A pre-reconstructed triangle mesh $M$ with faces $f_j$ and centers $m_j$ [S1].
*   A set of training views with camera poses $P_i$ and ground truth images [S1].
*   Standard 3D Gaussian parameters: position $\mathbf{p}_i$, rotation $\mathbf{R}_i$, scale $\mathbf{S}_i$, opacity $o_i$, and spherical harmonic coefficients for color $\mathbf{c}_i$ [S1].

**Outputs:**
*   Optimized Gaussian splats $\mathcal{G}$ classified into tightly-bound ($G_{ti}$) and loosely-bound ($G_{li}$) sets [S1].
*   Rendered images and depth maps [S1].

**Data Structures:**
*   **Mesh $M$**: A set of triangles. Each triangle $f_j$ has a center $m_j$ and associated geometric properties (normals, area) [S1].
*   **Gaussian Set $\mathcal{G}$**: A list of $M$ Gaussians. Each Gaussian $i$ stores:
    *   $\mathbf{p}_i \in \mathbb{R}^3$: Center position [S1].
    *   $\mathbf{R}_i \in \mathbb{R}^{3 \times 3}$: Rotation matrix (encoded by quaternion $q \in \mathbb{R}^4$) [S1].
    *   $\mathbf{S}_i \in \mathbb{R}^3$: Scaling vector [S1].
    *   $o_i \in [0, 1]$: Opacity [S1].
    *   $\mathbf{c}_i$: View-dependent color (SH) [S1].
    *   Classification flag: `tight` or `loose` [S1].

**Initialization:**
1.  Initialize one 3D Gaussian splat at the center $\mathbf{m}_j$ of each triangle $f_j$ in the mesh $M$ [S1].
2.  Set the initial covariance $\mathbf{\Sigma}_i$ based on the triangle size and orientation [S1].

**Rendering Pipeline:**
For each training view $P_i$:
1.  Transform 3D Gaussians to camera space:
    $$ \mathbf{p}_{ci} = \mathbf{R}_{cw} \mathbf{p}_i + \mathbf{t}_{cw} $$
    $$ \mathbf{\Sigma}_{ic} = \mathbf{R}_{cw} \mathbf{\Sigma}_i \mathbf{R}_{cw}^T $$
    where $\mathbf{R}_{cw}$ and $\mathbf{t}_{cw}$ are the camera rotation and translation [S1].
2.  Transform to ray space using the affine transformation $\mathbf{J}_i \in \mathbb{R}^{2 \times 3}$ defined by $\mathbf{p}_{ci}$:
    $$ \mathbf{\Sigma}'_i = \mathbf{J}_i \mathbf{\Sigma}_{ic} \mathbf{J}_i^T $$
    [S1].
3.  Compute the 2D Gaussian value:
    $$ g_i(\mathbf{x}) = e^{-\frac{1}{2}(\mathbf{x} - \mathbf{p}'_i)^T {\mathbf{\Sigma}'_i}^{-1}(\mathbf{x} - \mathbf{p}'_i)} $$
    [S1].
4.  Calculate blending weight:
    $$ \alpha_i = o_i g_i(\mathbf{x}) $$
    [S1].
5.  Render color via alpha blending:
    $$ \mathbf{c} = \sum_{i=1}^{n} \mathbf{c}_i \alpha_i \prod_{j=1}^{i-1} (1 - \alpha_j) $$
    [S1].
6.  Render depth $D_i$. Add a small offset of $0.01$ to the rendered depth $D_i$ to handle precision issues [S1].

**Occlusion Culling and Pruning:**
1.  Assume the mesh surface is entirely opaque [S1].
2.  Compare the rendered depth of each Gaussian splat against the mesh depth map $D_i$ from the current viewpoint [S1].
3.  Mask out (exclude from rendering) any Gaussian splats that are occluded by the mesh surface (i.e., splats located behind the mesh) [S1].
4.  Assess visibility across all training views. Identify splats located behind the mesh surface that are never observed by any training view [S1].
5.  Remove these redundant, occluded Gaussian splats at specific training intervals (every 500 iterations) [S1].

**Distance-Based Classification and Alignment:**
1.  Calculate the distance $d_i$ between each Gaussian splat’s center $\mathbf{p}_i$ and the corresponding mesh face $f_j$ [S1].
2.  Classify splats using a distance threshold $d_{th}$:
    *   **Tightly-bound ($G_{ti}$)**: If $d_i < d_{th}$ [S1].
    *   **Loosely-bound ($G_{li}$)**: If $d_i > d_{th}$ [S1].
3.  For tightly-bound splats ($G_{ti}$), flatten and align them with the mesh surface to ensure geometric consistency [S1]. This involves applying a normal consistency loss $\mathcal{L}_{nc}$ to refine the normals $\mathbf{n}_i$ of the Gaussians, estimated from the minimum scale axis of the covariance matrix [S1].

**Loss Functions:**
The total loss combines image fidelity and geometric constraints [S1]:
$$ \mathcal{L}_{img} = (1 - \lambda_{img}) \mathcal{L}_1(\hat{I}_i, I_i) + \lambda_{img} \text{SSIM}(\hat{I}_i, I_i) $$
[ S1 ].
Additional losses include normal consistency $\mathcal{L}_{nc}$ and projection constraints $\mathcal{L}_{proj}$, with weights $\lambda_{nc}=0.1$, $\lambda_{min}=0.1$, $\lambda_{max}=10$, $\lambda_{proj}=50$, and $\lambda_{img}=0.001$ [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def initialize_gaussians(mesh):
    """Initialize one Gaussian per triangle center."""
    gaussians = []
    for triangle in mesh.triangles:
        g = Gaussian()
        g.position = triangle.center
        g.covariance = compute_initial_covariance(triangle)
        g.opacity = 1.0
        g.classification = 'unknown'
        gaussians.append(g)
    return gaussians

def render_view(gaussians, camera_pose, mesh):
    """Render image and depth, applying occlusion culling."""
    # Transform to camera and ray space
    projected_gaussians = []
    for g in gaussians:
        g_cam = transform_to_camera(g, camera_pose)
        g_ray = transform_to_ray(g_cam, camera_pose)
        projected_gaussians.append(g_ray)
    
    # Sort by depth for alpha blending
    projected_gaussians.sort(key=lambda g: g.depth)
    
    rendered_color = 0.0
    rendered_depth = 0.0
    alpha_accum = 1.0
    
    for g in projected_gaussians:
        # Check occlusion against mesh depth map
        mesh_depth = get_mesh_depth_at_pixel(g.screen_pos, mesh)
        # Add offset to mesh depth for stability
        if g.depth > mesh_depth + 0.01:
            continue # Occluded by mesh, skip rendering
            
        # Compute 2D Gaussian value
        g_val = compute_2d_gaussian(g)
        alpha = g.opacity * g_val
        
        # Alpha blend
        color_contribution = g.color * alpha * alpha_accum
        rendered_color += color_contribution
        rendered_depth += g.depth * alpha * alpha_accum
        alpha_accum *= (1 - alpha)
        
    return rendered_color, rendered_depth

def update_gaussians(gaussians, mesh, iteration):
    """Prune occluded splats and classify based on distance."""
    if iteration % 500 == 0:
        # Prune splats occluded by mesh in all views
        visible_splats = set()
        for view in training_views:
            for g in gaussians:
                if not is_occluded_by_mesh(g, view.mesh_depth_map):
                    visible_splats.add(g.id)
        
        # Remove invisible splats
        gaussians = [g for g in gaussians if g.id in visible_splats]
    
    # Classify based on distance to mesh
    for g in gaussians:
        dist = distance_to_mesh_face(g.position, mesh)
        if dist < 0.01: # d_th = 0.01
            g.classification = 'tight'
            apply_normal_consistency_loss(g, mesh)
        else:
            g.classification = 'loose'
            
    return gaussians

def train_step(gaussians, batch):
    """Perform one training step."""
    rendered_img, rendered_depth = render_view(gaussians, batch.camera, batch.mesh)
    
    # Compute losses
    l1_loss = compute_l1(rendered_img, batch.gt_image)
    ssim_loss = compute_ssim(rendered_img, batch.gt_image)
    img_loss = (1 - 0.001) * l1_loss + 0.001 * ssim_loss
    
    # Geometric losses for tight splats
    geo_loss = compute_normal_consistency(gaussians, batch.mesh)
    
    total_loss = img_loss + 0.1 * geo_loss
    
    # Backpropagate and update parameters
    update_parameters(gaussians, total_loss)
    
    # Periodic maintenance
    gaussians = update_gaussians(gaussians, batch.mesh, current_iteration)
    
    return total_loss
```

**Parameters:**
*   Distance threshold $d_{th} = 0.01$ [S1].
*   Depth offset = $0.01$ [S1].
*   Pruning interval = 500 iterations [S1].
*   Loss weights: $\lambda_{img} = 0.001$, $\lambda_{nc} = 0.1$, $\lambda_{min} = 0.1$, $\lambda_{max} = 10$, $\lambda_{proj} = 50$ [S1].
*   Total training iterations: 30,000 [S1].

**Edge Cases and Pitfalls:**
*   **Mesh Artifacts:** If the mesh contains severe artifacts, tightly binding all splats degrades rendering quality. The loose-bound classification allows splats to deviate from the mesh to correct these artifacts [S1].
*   **Occlusion Precision:** The addition of $0.01$ to the depth map is critical to prevent self-occlusion errors due to floating-point precision [S1].
*   **Redundancy:** Without pruning, the number of Gaussians scales with the number of mesh triangles, which can be very large (e.g., 44 million triangles in BakedSDF), leading to excessive GPU memory usage [S1].

## Operating characteristics
*   **Memory Complexity:** Significantly reduced compared to standard 3DGS on high-poly meshes. By pruning occluded splats and not initializing splats for every triangle if they are redundant, it mitigates the memory explosion seen in naive mesh-to-Gaussian mapping [S1].
*   **Time Complexity:** Rendering is comparable to standard 3DGS, but occlusion culling adds a depth-map lookup overhead. Pruning every 500 iterations adds negligible overhead [S1].
*   **Strengths:** Handles large-scale meshes efficiently; corrects mesh artifacts via loosely-bound splats; maintains high rendering quality with fewer splats than standard 3DGS on complex geometry [S1].
*   **Limitations:** Requires a pre-existing mesh; performance depends on mesh quality; tightly-bound splats may fail to represent geometry if the mesh is inaccurate [S1].
*   **When to use:** When a high-quality mesh is available and standard 3DGS results in excessive memory usage or geometric inaccuracies due to mesh noise [S1].
*   **When not to use:** When no mesh is available or the scene is highly dynamic where mesh reconstruction is unstable [S1].

## Background
Distance-based Gaussian Splatting was introduced in the paper "MeshGS: Adaptive Mesh-Aligned Gaussian Splatting for High-Quality Rendering" by Jaehoon et al. [S1]. The method addresses the limitations of existing approaches like LTM (Lightweight Triangle Mesh) and SuGaR, which either require excessive triangle counts or vertex constraints that fail to capture complex geometry [S1]. It builds upon the foundational 3D Gaussian Splatting framework [22] but integrates explicit mesh alignment and distance-based classification [S1].

## Sources
[S1] MeshGS: Adaptive Mesh-Aligned Gaussian Splatting for High-Quality Rendering, https://arxiv.org/abs/2410.08941v1, fetched 2026-10-26
