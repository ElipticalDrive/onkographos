# Training Objective and Densification

Also called: MeshGS Training, Tightly/Loosely-bound Training

## Overview
This method defines the loss functions and densification protocols for **MeshGS** [S1], a variant of 3D Gaussian Splatting that aligns primitives with an explicit mesh surface. It solves the problem of Gaussian splats drifting away from geometry or expanding uncontrollably during optimization. It differs from standard 3DGS [22] by introducing mesh-aware losses (projection, normal consistency, scale regularization) and a dual-strategy densification process that treats tightly-bound splats differently from loosely-bound ones. It builds on [Progressive and Multi-stage Training](9966-progressive-and-multi-stage-training.md).

## How it works
**Inputs:**
*   Rendered image $\hat{I}_i$ and ground truth image $I_i$.
*   Set of tightly-bound Gaussian splats $G^t$ and loosely-bound splats $G^l$.
*   Mesh $M$ consisting of faces $F$, where each face $f_j$ has vertices $v_i$ and normal $n_f$.
*   For each Gaussian $i \in G^t$: center $P_i$, scale parameters $S_i = (s_1, s_2, s_3)$, and rotation matrix $R_i$.

**Outputs:**
*   Total loss $L$ for backpropagation.
*   Updated set of Gaussian splats via densification.

**Data Structures:**
*   **Gaussian Splat:** Contains position $P_i$, scale $S_i$, rotation $R_i$, opacity, and color.
*   **Mesh Face:** Defined by vertices and a computed normal vector.

**Parameters:**
*   $\lambda_{img} = 0.001$ [S1]
*   $\lambda_{nc} = 0.1$ [S1]
*   $\lambda_{min} = 0.1$ [S1]
*   $\lambda_{max} = 10$ [S1]
*   $\lambda_{proj} = 50$ [S1]
*   $\rho = 0.1$ (scale threshold) [S1]
*   $d_{th} = 0.01$ (distance threshold to distinguish tightly/loosely bound) [S1]
*   Split scale factor: $1.6$ [S1]
*   Total iterations: $30,000$ [S1]

**Steps:**

1.  **Compute Image Loss ($L_{img}$):**
    Calculate the combination of L1 loss and D-SSIM between the rendered image $\hat{I}_i$ and ground truth $I_i$ [S1]:
    $$ L_{img} = (1 - \lambda_{img})L_{1}(\hat{I}_{i},I_{i}) + \lambda_{img}SSIM(\hat{I}_{i}, I_{i}) $$

2.  **Estimate Tightly-Bound Normals:**
    For each Gaussian $i \in G^t$:
    a. Identify the minimum scale axis among $S_i = (s_1, s_2, s_3)$.
    b. Apply the rotation matrix $R_i$ to transform this axis into world coordinates to obtain the normal $n_i$ [S1].

3.  **Compute Normal Consistency Loss ($L_{nc}$):**
    Align the estimated normals $n_i$ of tightly-bound splats with the corresponding mesh face normals $n_f$. Sum the cosine distance over all tightly-bound splats and their associated mesh faces $F$ [S1]:
    $$ L_{nc} = \sum _{i \in G^{t},f \in F} (1 - n_{i} \cdot n_{f}) $$

4.  **Compute Scale Regularization Loss ($L_{scale}$):**
    Suppress excessive expansion of tightly-bound splats by regularizing the minimum and maximum scales against the threshold $\rho$ [S1]:
    $$ L_{scale} = \sum _{i \in G^{t}}(\lambda _{min}|\text {min}(s_{1}, s_{2}, s_{3})| + \lambda _{max}|\text {max}(s_{1}, s_{2}, s_{3}) - \rho |) $$

5.  **Compute Projection Loss ($L_{proj}$):**
    Project the center $P_i$ of each tightly-bound Gaussian to the closest point $s_i$ on the mesh surface defined by face $f_j$ and vertex $v_i$. The projection is calculated as [S1]:
    $$ s_{i} = P_{i} + ((P_{i} - v_{i}) \cdot f_{j}) \cdot f_{j} $$
    Then, compute the L2 distance between the original center and the projected point [S1]:
    $$ L_{proj} = \sum _{i \in G^{t}}\parallel s_{i} - P_{i} \parallel _{2} $$

6.  **Compute Total Loss:**
    Combine all terms [S1]:
    $$ L = L_{img} + \lambda _{nc} L_{nc} + L_{scale} +\lambda _{proj} L_{proj} $$

7.  **Densification Strategy:**
    Perform densification (cloning/splitting) based on the type of Gaussian [S1]:
    *   **Loosely-bound splats ($G^l$):** Follow the original 3DGS densification strategy [S1].
    *   **Tightly-bound splats ($G^t$):**
        a. Identify the mesh triangle close to the Gaussian.
        b. Conduct point sampling from the triangle surface.
        c. Use the sampled position to initialize new Gaussian splats.
        d. **Cloning:** Copy additional attributes from the parent to the new Gaussian.
        e. **Splitting:** Divide the scale of the new Gaussians by a factor of $1.6$ [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_loss(rendered_image, gt_image, tightly_bound_gaussians, mesh_faces):
    # 1. Image Loss
    l1_loss = L1(rendered_image, gt_image)
    ssim_loss = SSIM(rendered_image, gt_image)
    lambda_img = 0.001
    l_img = (1 - lambda_img) * l1_loss + lambda_img * ssim_loss
    
    # 2. Normal Consistency Loss
    l_nc = 0.0
    lambda_nc = 0.1
    for g in tightly_bound_gaussians:
        # Estimate normal: min scale axis rotated by R
        min_scale_idx = argmin(g.scale)
        local_normal = unit_vector(get_axis(min_scale_idx))
        world_normal = rotate(local_normal, g.rotation)
        
        # Find corresponding mesh face normal (simplified lookup)
        face_normal = get_face_normal(g, mesh_faces)
        
        # Cosine distance
        l_nc += (1 - dot(world_normal, face_normal))
    l_nc *= lambda_nc

    # 3. Scale Regularization Loss
    l_scale = 0.0
    lambda_min = 0.1
    lambda_max = 10
    rho = 0.1
    for g in tightly_bound_gaussians:
        s = g.scale
        min_s = min(s)
        max_s = max(s)
        l_scale += lambda_min * abs(min_s) + lambda_max * abs(max_s - rho)
        
    # 4. Projection Loss
    l_proj = 0.0
    lambda_proj = 50
    for g in tightly_bound_gaussians:
        # Project center P to closest point s on mesh face
        # Note: In implementation, 'closest face' must be determined first
        face = get_closest_face(g.position, mesh_faces)
        v = face.vertex
        f_normal = face.normal # Assuming f_j in formula refers to face normal or plane basis
        
        # Formula from source: s_i = P_i + ((P_i - v_i) . f_j) * f_j
        # This implies f_j is the normal of the plane containing the face
        diff = g.position - v
        projection_offset = dot(diff, f_normal) * f_normal
        s_i = g.position + projection_offset
        
        l_proj += norm(s_i - g.position)
    l_proj *= lambda_proj
    
    # 5. Total Loss
    total_loss = l_img + l_nc + l_scale + l_proj
    return total_loss

def densify_tightly_bound_gaussians(gaussians, mesh_faces):
    for g in gaussians:
        # Find close mesh triangle
        triangle = get_closest_triangle(g, mesh_faces)
        
        # Sample point from triangle
        sample_pos = sample_triangle(triangle)
        
        # Initialize new Gaussian
        new_g = Gaussian()
        new_g.position = sample_pos
        
        # Cloning: copy attributes
        # new_g.scale = g.scale
        # new_g.rotation = g.rotation
        # ... copy other attributes
        
        # Splitting: divide scale by 1.6
        # new_g.scale = g.scale / 1.6
        
        add_gaussian(new_g)
```

## Subtopics

### [Budget-Centered Training](9682-budget-centered-training.md)
Constrains primitive count to a target budget $N^*$ using a redundancy map, eliminating the need for gradient-based pruning.

## Operating characteristics
*   **Time Complexity:** The projection loss requires finding the closest mesh face for every tightly-bound Gaussian, which adds overhead compared to standard 3DGS. The normal consistency loss involves summing over splat-face pairs.
*   **Memory Complexity:** Requires storing the mesh geometry (faces, vertices, normals) in addition to Gaussian attributes.
*   **Performance:** Evaluated on Mip-NeRF360 and Deep Blending datasets [S1]. Trained for 30k iterations [S1].
*   **Strengths:** Prevents splats from drifting from the surface; aligns geometry with explicit mesh; suppresses excessive scale expansion.
*   **Limitations:** Relies on the quality of the input mesh; tightly-bound splats are constrained to the surface, potentially limiting representation of non-surface details if the mesh is inaccurate.

## Background
Originated in **MeshGS: Adaptive Mesh-Aligned Gaussian Splatting for High-Quality Rendering** [S1]. The method adapts 3D Gaussian Splatting [22] to work with explicit mesh representations, addressing the lack of geometric constraints in standard 3DGS. It distinguishes between tightly-bound (surface-aligned) and loosely-bound splats to apply appropriate densification and loss functions.

## Sources
[S1] MeshGS: Adaptive Mesh-Aligned Gaussian Splatting for High-Quality Rendering, https://arxiv.org/abs/2410.08941v1, fetched 2026-10-26
