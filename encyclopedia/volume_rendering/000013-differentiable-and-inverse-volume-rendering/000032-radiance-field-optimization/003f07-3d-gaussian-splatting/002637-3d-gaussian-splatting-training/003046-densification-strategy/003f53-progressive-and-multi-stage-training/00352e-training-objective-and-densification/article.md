# Training Objective and Densification

Also called: MeshGS Training, Tightly/Loosely-bound Training

## Overview
This method defines the loss functions and densification protocols for **MeshGS** (Adaptive Mesh-Aligned Gaussian Splatting), a variant of 3D Gaussian Splatting that aligns primitives with an explicit mesh surface [S1]. It solves the problem of geometric inconsistency in standard 3DGS by introducing a **tightly-bound** subset of Gaussians constrained to a mesh and a **loosely-bound** subset that follows standard optimization [S1]. It differs from its siblings by explicitly separating densification logic based on proximity to the mesh: loosely-bound splats use standard gradient-based splitting/cloning, while tightly-bound splats use mesh-aligned point sampling for densification and are penalized by projection, normal consistency, and scale regularization losses [S1].

## How it works

### Inputs and Outputs
*   **Inputs:**
    *   Set of Gaussian splats $G$, partitioned into tightly-bound $G^t$ and loosely-bound $G^l$ based on distance threshold $d_{th} = 0.01$ [S1].
    *   Explicit mesh $M$ consisting of faces $F$ and vertices $V$ [S1].
    *   Ground truth images $I_i$ and rendered images $\hat{I}_i$ [S1].
*   **Outputs:**
    *   Optimized Gaussian parameters (position, rotation, scale, opacity, color) [S1].
    *   Updated set of Gaussals after densification steps [S1].

### Data Structures
*   **Gaussian Splat $i$:**
    *   Center $P_i \in \mathbb{R}^3$ [S1].
    *   Scale $S_i = (s_1, s_2, s_3) \in \mathbb{R}^3$ [S1].
    *   Rotation $R_i$ (quaternion or matrix) [S1].
    *   Normal $n_i$ (derived from $S_i$ and $R_i$) [S1].
*   **Mesh Face $f_j$:**
    *   Vertices $v_i$ [S1].
    *   Face normal $n_f$ [S1].

### Parameters
*   $\lambda_{img} = 0.001$ [S1].
*   $\lambda_{nc} = 0.1$ [S1].
*   $\lambda_{min} = 0.1$ [S1].
*   $\lambda_{max} = 10$ [S1].
*   $\lambda_{proj} = 50$ [S1].
*   $\rho = 0.1$ (scale threshold) [S1].
*   $d_{th} = 0.01$ (distance threshold for binding) [S1].
*   Split scale factor: $1.6$ [S1].
*   Training iterations: $30,000$ [S1].
*   Densification frequency: Every $500$ iterations [S1].

### Formulas
1.  **Image Loss:**
    $$L_{img} = (1-\lambda_{img})L_{1}(\hat{I}_{i},I_{i}) + \lambda_{img}SSIM(\hat{I}_{i}, I_{i})$$
    Where $L_1$ is the L1 loss and $SSIM$ is the Structural Similarity Index [S1].

2.  **Normal Consistency Loss:**
    $$L_{nc} = \sum_{i \in G^{t},f \in F} (1 - n_{i} \cdot n_{f})$$
    Where $n_i$ is the splat normal and $n_f$ is the face normal [S1].

3.  **Scale Regularization Loss:**
    $$L_{scale} = \sum_{i \in G^{t}}(\lambda_{min}|\text{min}(s_{1}, s_{2}, s_{3})| + \lambda_{max}|\text{max}(s_{1}, s_{2}, s_{3}) - \rho|)$$
    This suppresses excessive expansion of tightly-bound splats [S1].

4.  **Projection Loss:**
    $$s_{i} = P_{i} + ((P_{i} - v_{i}) \cdot f_{j}) \cdot f_{j}$$
    $$L_{proj} = \sum_{i \in G^{t}}\parallel s_{i} - P_{i} \parallel _{2}$$
    Where $s_i$ is the projection of center $P_i$ onto the mesh surface defined by face $f_j$ and vertex $v_i$ [S1].

5.  **Total Loss:**
    $$L = L_{img} + \lambda_{nc} L_{nc} + L_{scale} +\lambda_{proj} L_{proj}$$
    [S1]

### Numbered Steps

1.  **Initialization:** Initialize Gaussian splats. Classify each splat $i$ as tightly-bound ($G^t$) if its distance to the mesh is $\le d_{th}$, otherwise loosely-bound ($G^l$) [S1].
2.  **Normal Estimation (for $L_{nc}$):** For each tightly-bound splat $i$, identify the minimum scale axis among $S_i$. Apply rotation matrix $R_i$ to transform this axis into world coordinates to estimate the normal $n_i$ [S1].
3.  **Projection Calculation (for $L_{proj}$):** For each tightly-bound splat $i$, find the closest mesh face $f_j$ and vertex $v_i$. Calculate the closest point $s_i$ on the mesh surface using the projection formula [S1].
4.  **Loss Computation:** Compute $L_{img}$, $L_{nc}$, $L_{scale}$, and $L_{proj}$ using the formulas above. Sum them with weights to get total loss $L$ [S1].
5.  **Gradient Update:** Backpropagate $L$ to update Gaussian parameters [S1].
6.  **Densification (Every 500 iterations):**
    *   **For Loosely-bound Splats ($G^l$):** Apply the original 3DGS densification strategy (splitting/cloning based on gradient magnitude) [S1].
    *   **For Tightly-bound Splats ($G^t$):**
        *   Identify the mesh triangle close to the splat [S1].
        *   Perform random point sampling from the triangle surface [S1].
        *   Use the sampled position to initialize new Gaussian splats [S1].
        *   **Cloning:** Copy additional attributes from the parent splat to the new splat [S1].
        *   **Splitting:** Divide the scale of the new splats by a factor of $1.6$ [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_normal(splat):
    """Estimate normal for a tightly-bound Gaussian splat."""
    # Identify minimum scale axis
    min_scale_idx = argmin(splat.scale)
    # Get the axis direction in local space (e.g., [1,0,0], [0,1,0], or [0,0,1])
    local_axis = identity_matrix[:, min_scale_idx]
    # Transform to world coordinates using rotation matrix
    world_normal = splat.rotation @ local_axis
    return normalize(world_normal)

def project_to_mesh(center, mesh):
    """Project Gaussian center to closest point on mesh surface."""
    # Find closest face f_j and vertex v_i
    face, vertex = mesh.find_closest_face_and_vertex(center)
    # Calculate projection
    # Note: The source formula uses a specific vector projection logic.
    # s_i = P_i + ((P_i - v_i) . f_j) * f_j
    # This implies f_j is treated as a normal or direction vector in the source context.
    # Assuming f_j is the face normal for projection onto the plane of the face.
    displacement = center - vertex
    projection_component = dot(displacement, face.normal)
    projected_point = center + (projection_component * face.normal)
    return projected_point

def densify_tightly_bound_splats(gaussians, mesh, iteration):
    """Densify tightly-bound Gaussals using mesh sampling."""
    if iteration % 500 != 0:
        return
    
    new_gaussians = []
    for g in gaussians:
        if g in tightly_bound_set:
            # Find close mesh triangle
            triangle = mesh.find_closest_triangle(g.center)
            
            # Random sampling from triangle
            sample_point = triangle.random_sample()
            
            # Initialize new Gaussian
            new_g = Gaussian()
            new_g.center = sample_point
            
            # Cloning: Copy attributes
            new_g.opacity = g.opacity
            new_g.color = g.color
            new_g.rotation = g.rotation
            
            # Splitting: Scale division
            new_g.scale = g.scale / 1.6
            
            new_gaussians.append(new_g)
            
    return new_gaussians

def train_step(gaussians, mesh, images, gt_images, iteration):
    total_loss = 0
    
    # 1. Image Loss
    rendered_images = render(gaussians, camera)
    l_img = (1 - lambda_img) * L1(rendered_images, gt_images) + lambda_img * SSIM(rendered_images, gt_images)
    total_loss += l_img
    
    # 2. Normal Consistency Loss (Tightly-bound only)
    l_nc = 0
    for g in gaussians:
        if g in tightly_bound_set:
            n_i = compute_normal(g)
            # Find corresponding face normal n_f
            face = mesh.find_closest_face(g.center)
            n_f = face.normal
            l_nc += (1 - dot(n_i, n_f))
    total_loss += lambda_nc * l_nc
    
    # 3. Scale Regularization Loss (Tightly-bound only)
    l_scale = 0
    for g in gaussians:
        if g in tightly_bound_set:
            s = g.scale
            min_s = min(s)
            max_s = max(s)
            l_scale += lambda_min * abs(min_s) + lambda_max * abs(max_s - rho)
    total_loss += l_scale
    
    # 4. Projection Loss (Tightly-bound only)
    l_proj = 0
    for g in gaussians:
        if g in tightly_bound_set:
            s_i = project_to_mesh(g.center, mesh)
            l_proj += norm(s_i - g.center)
    total_loss += lambda_proj * l_proj
    
    # Backpropagate and update
    total_loss.backward()
    optimizer.step()
    
    # Densification
    if iteration % 500 == 0:
        new_tightly_bound = densify_tightly_bound_splats(gaussians, mesh, iteration)
        gaussians.extend(new_tightly_bound)
        
    return total_loss
```

### Edge Cases and Pitfalls
*   **Precision:** The projection formula relies on the correct interpretation of $f_j$ as a normal vector for plane projection. If $f_j$ is an edge vector, the projection logic fails [S1].
*   **Boundary:** The distance threshold $d_{th}=0.01$ determines the split between $G^t$ and $G^l$. Splats near this boundary may oscillate if not handled carefully [S1].
*   **Degenerate Input:** If the mesh has non-manifold geometry or zero-area faces, finding the "closest face" or projecting onto it may be undefined [S1].

## Subtopics

### [Budget-Centered Training](9682-budget-centered-training.md)
Budget-Centered Training (BCT) is a training strategy for 3D Gaussian Splatting that initializes and constrains the primitive count to the "knee" of the scene's capacity floor curve ($N^*$), ensuring the model is born at the optimal size for a target quality rather than growing large and being pruned later. It differs from standard densification pipelines by replacing gradient-based pruning signals with a redundancy map ($\theta$) and by treating the primitive budget as a hard constraint during the optimization phase, thereby eliminating wasted computation on primitives that will eventually be removed.

## Operating characteristics

*   **Time Complexity:** The additional overhead comes from mesh intersection queries for projection and normal consistency. Finding the closest face/vertex for every tightly-bound splat at every step is $O(N \log M)$ or $O(N)$ with spatial hashing, where $N$ is the number of Gaussians and $M$ is the number of mesh faces [S1].
*   **Memory Complexity:** Requires storing the mesh geometry alongside the Gaussian parameters. The number of Gaussians is controlled by the densification strategy, which for $G^t$ is driven by mesh sampling rather than gradient magnitude alone [S1].
*   **Measured Performance:** Evaluated on Mip-NeRF360 and Deep Blending datasets. Trained for 30k iterations [S1].
*   **Strengths:** Ensures tight alignment with mesh geometry, preventing "floating" artifacts common in standard 3DGS [S1].
*   **Limitations:** Dependent on the quality of the input mesh. If the mesh is inaccurate, the tightly-bound splats will be constrained to incorrect geometry [S1].
*   **When to use:** When an explicit mesh is available and high-fidelity geometric alignment is required [S1].
*   **When not to use:** When no mesh is available or when the scene geometry is highly dynamic and cannot be represented by a static mesh [S1].

## Background

*   **Origin:** MeshGS (Adaptive Mesh-Aligned Gaussian Splatting for High-Quality Rendering) [S1].
*   **History:** Builds upon the original 3D Gaussian Splatting (3DGS) [S1]. It addresses the limitation of 3DGS in explicitly representing geometry by introducing mesh-aligned constraints [S1].
*   **Usage:** Used for high-quality rendering of large-scale real-world scenes where mesh reconstruction is also desired [S1].

## Sources

[S1] MeshGS: Adaptive Mesh-Aligned Gaussian Splatting for High-Quality Rendering, https://arxiv.org/abs/2410.08941v1, fetched 2026-10-26
