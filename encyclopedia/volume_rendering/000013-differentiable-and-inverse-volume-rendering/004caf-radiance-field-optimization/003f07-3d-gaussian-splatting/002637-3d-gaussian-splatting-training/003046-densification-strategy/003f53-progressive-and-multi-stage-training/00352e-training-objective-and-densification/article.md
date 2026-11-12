# Training Objective and Densification

Also called: MeshGS Training, Tightly/Loosely-bound Training

## Overview
This method defines the loss function and densification strategy for **MeshGS** (Adaptive Mesh-Aligned Gaussian Splatting), a variant of 3D Gaussian Splatting that aligns primitives with an explicit mesh surface [S1]. It solves the problem of Gaussian splats drifting away from geometry or expanding excessively by introducing tight coupling between the splats and the mesh [S1]. It differs from standard 3D Gaussian Splatting [3DGS] by splitting the primitive population into **tightly-bound** ($G^t$) and **loosely-bound** ($G^l$) sets, applying mesh-specific regularization and projection losses only to the tightly-bound set, while retaining the original densification logic for the loosely-bound set [S1].

## How it works

### Inputs and Outputs
**Inputs:**
*   Rendered image $\hat{I}_i$ and ground truth image $I_i$.
*   Set of tightly-bound Gaussian splats $G^t$ and loosely-bound splats $G^l$.
*   Mesh $M$ consisting of faces $F$, where each face $f_j$ has a normal $n_f$ and vertices $v_i$.
*   Current Gaussian attributes: center $P_i$, scale $S_i = (s_1, s_2, s_3)$, rotation $R_i$, opacity, and color.
*   Hyperparameters: $\lambda_{img}=0.001$, $\lambda_{nc}=0.1$, $\lambda_{min}=0.1$, $\lambda_{max}=10$, $\lambda_{proj}=50$, $\rho=0.1$ [S1].

**Outputs:**
*   Total loss $L$ for backpropagation.
*   Updated Gaussian attributes (centers, scales, etc.) after optimization steps.
*   New Gaussian splats generated via densification.

### Data Structures
*   **Gaussian Splat $i$**:
    *   $P_i \in \mathbb{R}^3$: Center position.
    *   $S_i \in \mathbb{R}^3$: Scale parameters (log-space or linear, as per 3DGS implementation).
    *   $R_i \in \mathbb{R}^{3 \times 3}$: Rotation matrix (quaternion converted).
    *   $n_i \in \mathbb{R}^3$: Estimated normal vector.
    *   Flag: `is_tightly_bound` (boolean).
*   **Mesh Face $f_j$**:
    *   Vertices $v_0, v_1, v_2$.
    *   Normal $n_f \in \mathbb{R}^3$.
    *   Barycentric coordinates for sampling.

### Numbered Steps

1.  **Classification**: Classify each Gaussian splat $i$ as tightly-bound ($G^t$) or loosely-bound ($G^l$) based on its distance to the mesh surface. If the distance is below a threshold $d_{th}=0.01$, it is tightly-bound [S1].
2.  **Image Loss Calculation**: Compute the image reconstruction loss for all splats using the combined L1 and SSIM metric [S1]:
    $$ L_{img} = (1-\lambda_{img})L_{1}(\hat{I}_i, I_i) + \lambda_{img}SSIM(\hat{I}_i, I_i) $$
    where $\hat{I}_i$ is the rendered image and $I_i$ is the ground truth [S1].
3.  **Normal Estimation for Tightly-Bound Splats**: For each $i \in G^t$, estimate the normal $n_i$ by identifying the axis of the minimum scale in $S_i$ and applying the rotation matrix $R_i$ to transform this axis into world coordinates [S1].
4.  **Normal Consistency Loss**: Compute the alignment between the estimated splat normals $n_i$ and the corresponding mesh face normals $n_f$ for all $i \in G^t$ and faces $f \in F$ [S1]:
    $$ L_{nc} = \sum_{i \in G^t, f \in F} (1 - n_i \cdot n_f) $$
    Note: The summation implies checking alignment with the closest face or all faces; the source formula sums over $f \in F$, but logically this refers to the face $f_j$ corresponding to the splat's projection [S1].
5.  **Scale Regularization**: Apply regularization to the scale parameters $S_i = (s_1, s_2, s_3)$ of tightly-bound splats to prevent excessive expansion and align with the mesh surface [S1]:
    $$ L_{scale} = \sum_{i \in G^t} (\lambda_{min}|\min(s_1, s_2, s_3)| + \lambda_{max}|\max(s_1, s_2, s_3) - \rho|) $$
    where $\rho=0.1$ is the threshold for the maximum scale [S1].
6.  **Projection Loss**: For each $i \in G^t$, project the center $P_i$ to the closest point $s_i$ on the mesh surface defined by face $f_j$ and vertex $v_i$. The projection is calculated as [S1]:
    $$ s_i = P_i + ((P_i - v_i) \cdot f_j) \cdot f_j $$
    *Correction*: The source formula uses $f_j$ as a vector in the projection term $((P_i - v_i) \cdot f_j) \cdot f_j$. This implies $f_j$ is treated as the surface normal or a specific vector direction. Standard projection onto a plane with normal $n_f$ would be $P_i - ((P_i - v_i) \cdot n_f)n_f$. The source explicitly writes $f_j$ in the dot product and multiplication. We follow the source notation [S1]:
    $$ L_{proj} = \sum_{i \in G^t} \| s_i - P_i \|_2 $$
    This loss ensures splats remain close to the mesh surface [S1].
7.  **Total Loss**: Combine all terms [S1]:
    $$ L = L_{img} + \lambda_{nc} L_{nc} + L_{scale} + \lambda_{proj} L_{proj} $$
8.  **Densification**:
    *   **Loosely-bound ($G^l$)**: Follow the original 3DGS densification strategy (cloning/splitting based on gradient magnitude and opacity) [S1].
    *   **Tightly-bound ($G^t$)**: Perform cloning and splitting by sampling points from the corresponding mesh triangle. Initialize new Gaussian splats at these sampled positions [S1].
        *   **Cloning**: Copy additional attributes (color, opacity, etc.) from the parent splat to the new splat [S1].
        *   **Splitting**: Divide the scale of the parent splat by a factor of 1.6 [S1].
9.  **Optimization**: Update Gaussian parameters using gradient descent with the total loss $L$ for 30,000 iterations [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_loss_and_densify(Gaussians, Mesh, Images, Iteration):
    """
    Computes the MeshGS training objective and performs densification.
    """
    L_img = 0.0
    L_nc = 0.0
    L_scale = 0.0
    L_proj = 0.0
    
    # Hyperparameters
    lambda_img = 0.001
    lambda_nc = 0.1
    lambda_min = 0.1
    lambda_max = 10.0
    lambda_proj = 50.0
    rho = 0.1
    d_th = 0.01
    
    G_t = [] # Tightly-bound splats
    G_l = [] # Loosely-bound splats
    
    # 1. Classification and Image Loss
    for splat in Gaussians:
        # Calculate distance to mesh
        dist = distance_to_mesh(splat.P, Mesh)
        
        if dist < d_th:
            G_t.append(splat)
        else:
            G_l.append(splat)
            
        # Render and compute image loss (standard 3DGS rendering)
        rendered_img = render(Gaussians, camera)
        L1_loss = L1(rendered_img, ground_truth_img)
        SSIM_loss = SSIM(rendered_img, ground_truth_img)
        L_img += (1 - lambda_img) * L1_loss + lambda_img * SSIM_loss
        
    # 2. Normal Consistency Loss for Tightly-Bound Splats
    for splat in G_t:
        # Estimate normal: min scale axis rotated to world space
        min_scale_idx = argmin(splat.S)
        local_normal = unit_vector([0,0,0])
        local_normal[min_scale_idx] = 1.0
        world_normal = splat.R @ local_normal
        splat.n_i = world_normal
        
        # Find corresponding mesh face f_j
        face = get_closest_face(splat.P, Mesh)
        n_f = face.normal
        
        # Cosine similarity loss
        L_nc += (1.0 - dot(splat.n_i, n_f))
        
    # 3. Scale Regularization for Tightly-Bound Splats
    for splat in G_t:
        s = splat.S
        min_s = min(s)
        max_s = max(s)
        L_scale += lambda_min * abs(min_s) + lambda_max * abs(max_s - rho)
        
    # 4. Projection Loss for Tightly-Bound Splats
    for splat in G_t:
        face = get_closest_face(splat.P, Mesh)
        # Source formula: s_i = P_i + ((P_i - v_i) . f_j) * f_j
        # Note: f_j in source likely refers to the face normal or a specific vector.
        # We use the face normal as the projection direction based on context "closest point on mesh surface".
        # However, strictly following source notation where f_j is used in dot product:
        # If f_j is the normal n_f:
        n_f = face.normal
        v_i = face.vertices[0] # Pick a vertex from the face
        projection_offset = dot(splat.P - v_i, n_f) * n_f
        s_i = splat.P + projection_offset
        L_proj += norm(s_i - splat.P)
        
    # 5. Total Loss
    L = L_img + lambda_nc * L_nc + L_scale + lambda_proj * L_proj
    
    # 6. Densification
    new_splats = []
    
    # Loosely-bound: Original 3DGS strategy
    for splat in G_l:
        if should_clone(splat):
            new_splats.append(clone_splat(splat))
        elif should_split(splat):
            new_splats.append(split_splat(splat))
            
    # Tightly-bound: Mesh-aligned strategy
    for splat in G_t:
        face = get_closest_face(splat.P, Mesh)
        # Sample point from triangle
        sampled_point = sample_triangle(face)
        
        if should_clone(splat):
            new_splat = clone_splat(splat)
            new_splat.P = sampled_point
            # Copy additional attributes
            new_splats.append(new_splat)
        elif should_split(splat):
            new_splat = split_splat(splat)
            new_splat.P = sampled_point
            # Scale divided by 1.6
            new_splat.S = new_splat.S / 1.6
            new_splats.append(new_splat)
            
    Gaussians.extend(new_splats)
    return L
```

### Parameters
*   $\lambda_{img} = 0.001$: Weight for SSIM in image loss [S1].
*   $\lambda_{nc} = 0.1$: Weight for normal consistency loss [S1].
*   $\lambda_{min} = 0.1$: Weight for minimum scale regularization [S1].
*   $\lambda_{max} = 10$: Weight for maximum scale regularization [S1].
*   $\lambda_{proj} = 50$: Weight for projection loss [S1].
*   $\rho = 0.1$: Threshold for maximum scale in regularization [S1].
*   $d_{th} = 0.01$: Distance threshold for classifying tightly-bound splats [S1].
*   Training iterations: 30,000 [S1].

### Edge Cases and Pitfalls
*   **Projection Vector Ambiguity**: The source formula for $L_{proj}$ uses $f_j$ in the term $((P_i - v_i) \cdot f_j) \cdot f_j$. In standard geometry, projecting onto a surface requires the surface normal. If $f_j$ is interpreted as the face normal, the formula represents the distance to the plane. If $f_j$ is a tangent vector, the formula is incorrect for surface projection. The implementation should use the face normal $n_f$ for the projection direction to ensure $s_i$ lies on the mesh surface [S1].
*   **Scale Regularization**: The term $|\max(s_1, s_2, s_3) - \rho|$ penalizes scales larger than $\rho$. If scales are in log-space, this must be converted to linear space before comparison [S1].
*   **Normal Estimation**: Estimating normals from the minimum scale axis assumes the Gaussian is elongated along the surface tangent or normal. If the Gaussian is spherical, this axis is unstable. This method assumes tightly-bound splats are flattened against the mesh [S1].

## Subtopics

## Operating characteristics

*   **Time Complexity**: The addition of mesh projection and normal consistency losses increases the per-iteration cost compared to standard 3DGS. Finding the closest mesh face and computing projections for all tightly-bound splats adds an $O(N \log M)$ or $O(N)$ overhead depending on mesh traversal acceleration structures (e.g., BVH), where $N$ is the number of splats and $M$ is the number of mesh faces [S1].
*   **Memory Complexity**: Requires storing the mesh geometry (vertices, faces, normals) in addition to the Gaussian attributes. The number of splats is constrained by the mesh alignment, potentially reducing the total count compared to unconstrained 3DGS if the mesh is a good representation of the scene [S1].
*   **Performance**: Evaluated on Mip-NeRF360 and Deep Blending datasets. The method achieves high-quality rendering by preventing splat drift and excessive scaling [S1].
*   **Strengths**: Explicitly enforces geometric alignment, preventing the "floating" artifacts common in standard 3DGS. The scale regularization keeps splats compact and aligned with the mesh resolution.
*   **Limitations**: Relies on the quality of the input mesh. If the mesh is noisy or incomplete, the tightly-bound splats will be constrained to incorrect geometry. The projection loss weight $\lambda_{proj}=50$ is high, which may dominate the gradient if not balanced carefully [S1].

## Background

*   **Origin**: MeshGS (Adaptive Mesh-Aligned Gaussian Splatting) was introduced in 2024 [S1].
*   **History**: It builds upon the original 3D Gaussian Splatting [3DGS] and methods like SuGaR which also bind Gaussians to meshes. MeshGS differentiates itself by using a specific projection loss and scale regularization tailored for tightly-bound splats, while retaining the original densification for loosely-bound ones [S1].
*   **Usage**: Used for high-quality rendering of large-scale scenes where explicit mesh geometry is available or can be extracted (e.g., from neural fields) [S1].

## Sources

[S1] MeshGS: Adaptive Mesh-Aligned Gaussian Splatting for High-Quality Rendering, https://arxiv.org/abs/2410.08941v1, fetched 2026-10-26
