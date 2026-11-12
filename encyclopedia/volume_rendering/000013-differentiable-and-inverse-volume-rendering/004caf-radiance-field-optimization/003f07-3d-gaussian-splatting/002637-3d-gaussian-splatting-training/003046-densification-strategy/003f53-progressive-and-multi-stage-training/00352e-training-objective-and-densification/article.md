# Training Objective and Densification

Also called: MeshGS Training, Tightly/Loosely-bound Training

## Overview
This method defines the loss functions and densification protocols for **MeshGS** [S1], a variant of 3D Gaussian Splatting that aligns primitives with an explicit mesh surface. It solves the problem of Gaussian splats drifting away from geometry or expanding uncontrollably by introducing surface-aware regularization and projection losses. It differs from standard 3DGS [22] by splitting the primitive population into **tightly-bound** ($G^t$) and **loosely-bound** ($G^l$) sets, applying distinct densification rules and loss terms to each, and enforcing proximity to the mesh surface [S1].

## How it works
**Inputs:**
*   Rendered image $\hat{I}_i$ and ground truth image $I_i$.
*   Set of tightly-bound Gaussians $G^t$ and loosely-bound Gaussians $G^l$.
*   Mesh $M$ consisting of faces $F$, where each face $f_j$ has a normal $n_f$ and vertices $v_i$.
*   Gaussian parameters for each $i$: center $P_i$, scale $S_i = (s_1, s_2, s_3)$, rotation $R_i$, opacity, and color.

**Outputs:**
*   Total loss $L$ for backpropagation.
*   Updated Gaussian set via densification (cloning/splitting).

**Data Structures:**
*   $G^t$: Gaussians within distance threshold $d_{th} = 0.01$ from the mesh surface [S1].
*   $G^l$: Gaussians outside distance threshold $d_{th}$ [S1].

**Parameters:**
*   $\lambda_{img} = 0.001$ [S1].
*   $\lambda_{nc} = 0.1$ [S1].
*   $\lambda_{min} = 0.1$ [S1].
*   $\lambda_{max} = 10$ [S1].
*   $\lambda_{proj} = 50$ [S1].
*   $\rho = 0.1$ (scale threshold) [S1].
*   Training iterations: 30,000 [S1].
*   Densification frequency: Every 500 iterations [S1].

**Steps:**

1.  **Compute Image Loss ($L_{img}$):**
    Calculate the combination of L1 loss and D-SSIM between the rendered image $\hat{I}_i$ and ground truth $I_i$ [S1]:
    $$ L_{img} = (1 - \lambda_{img})L_{1}(\hat{I}_i, I_i) + \lambda_{img}SSIM(\hat{I}_i, I_i) $$

2.  **Compute Normal Consistency Loss ($L_{nc}$):**
    For each tightly-bound Gaussian $i \in G^t$:
    a. Identify the minimum scale axis among $S_i = (s_1, s_2, s_3)$.
    b. Transform this axis into world coordinates using the rotation matrix $R_i$ to estimate the normal $n_i$ [S1].
    c. Find the corresponding mesh face $f \in F$ (typically the closest face or the one used for projection).
    d. Compute the cosine similarity loss between $n_i$ and the face normal $n_f$ [S1]:
    $$ L_{nc} = \sum_{i \in G^t, f \in F} (1 - n_i \cdot n_f) $$

3.  **Compute Scale Regularization Loss ($L_{scale}$):**
    For each tightly-bound Gaussian $i \in G^t$, regularize the scale parameters to prevent excessive expansion relative to the mesh face [S1]:
    $$ L_{scale} = \sum_{i \in G^t} \left( \lambda_{min}|\text{min}(s_1, s_2, s_3)| + \lambda_{max}|\text{max}(s_1, s_2, s_3) - \rho| \right) $$
    Here, $\text{min}(s_1, s_2, s_3)$ and $\text{max}(s_1, s_2, s_3)$ are the minimum and maximum values in the scale vector [S1].

4.  **Compute Projection Loss ($L_{proj}$):**
    For each tightly-bound Gaussian $i \in G^t$:
    a. Identify the closest mesh face $f_j$ with vertex $v_i$ (Note: The source formula uses $v_i$ as a vertex of the face $f_j$ associated with the projection, not necessarily the Gaussian's vertex) [S1].
    b. Project the Gaussian center $P_i$ onto the plane of face $f_j$ to find the closest point $s_i$. The source provides the formula:
    $$ s_i = P_i + ((P_i - v_i) \cdot f_j) \cdot f_j $$
    *Note: The source formula implies projecting along the face normal $f_j$ (assuming $f_j$ is normalized) from a point on the plane defined by $v_i$. Standard projection of point $P$ onto plane with normal $n$ passing through $v$ is $P - ((P-v)\cdot n)n$. The source formula has a sign difference or specific vector definition. We follow the source's explicit formula [S1].*
    c. Compute the distance loss:
    $$ L_{proj} = \sum_{i \in G^t} \| s_i - P_i \|_2 $$
    This ensures tightly-bound splats remain close to the mesh surface [S1].

5.  **Compute Total Loss:**
    $$ L = L_{img} + \lambda_{nc} L_{nc} + L_{scale} + \lambda_{proj} L_{proj} $$

6.  **Densification Strategy (Every 500 iterations):**
    *   **Loosely-bound Gaussians ($G^l$):** Follow the original 3DGS densification strategy (gradient-based splitting and cloning) [S1].
    *   **Tightly-bound Gaussians ($G^t$):**
        a. Identify the mesh triangle close to the Gaussian [S1].
        b. Perform point sampling from the triangle to initialize new Gaussian splats for both cloning and splitting operations [S1].
        c. **Cloning:** Copy additional attributes from the parent Gaussian to the new Gaussian [S1].
        d. **Splitting:** Divide the scale of the new Gaussians by a factor of 1.6 [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_loss(G_t, G_l, mesh, rendered_image, ground_truth_image, params):
    """
    G_t: List of tightly-bound Gaussian objects
    G_l: List of loosely-bound Gaussian objects
    mesh: Mesh object with faces (normal, vertices)
    rendered_image: Rendered RGB image
    ground_truth_image: Ground truth RGB image
    params: Dictionary containing lambda values and rho
    """
    # 1. Image Loss
    l1_loss = compute_l1_loss(rendered_image, ground_truth_image)
    ssim_loss = compute_ssim_loss(rendered_image, ground_truth_image)
    l_img = (1 - params['lambda_img']) * l1_loss + params['lambda_img'] * ssim_loss
    
    # 2. Normal Consistency Loss
    l_nc = 0.0
    for gauss in G_t:
        # Estimate normal: min scale axis rotated by R
        min_scale_idx = argmin(gauss.scale)
        local_normal = unit_vector(get_axis(min_scale_idx)) # e.g., [1,0,0] for x-axis
        world_normal = rotate_vector(local_normal, gauss.rotation)
        
        # Find corresponding mesh face (closest face logic implied)
        face = mesh.get_closest_face(gauss.position)
        face_normal = face.normal
        
        l_nc += (1.0 - dot_product(world_normal, face_normal))
        
    # 3. Scale Regularization Loss
    l_scale = 0.0
    for gauss in G_t:
        scales = gauss.scale # (s1, s2, s3)
        min_s = min(scales)
        max_s = max(scales)
        
        term_min = params['lambda_min'] * abs(min_s)
        term_max = params['lambda_max'] * abs(max_s - params['rho'])
        l_scale += term_min + term_max
        
    # 4. Projection Loss
    l_proj = 0.0
    for gauss in G_t:
        P_i = gauss.position
        face = mesh.get_closest_face(P_i)
        v_i = face.vertices[0] # Using first vertex as reference point from source formula
        f_j = face.normal      # Face normal used as projection direction in source formula
        
        # Source formula: s_i = P_i + ((P_i - v_i) . f_j) * f_j
        # Note: This formula projects P_i onto the line passing through v_i with direction f_j,
        # or potentially implies a specific coordinate system. We implement exactly as source.
        diff = vector_subtract(P_i, v_i)
        projection_scalar = dot_product(diff, f_j)
        projection_vector = vector_scale(f_j, projection_scalar)
        s_i = vector_add(P_i, projection_vector)
        
        l_proj += vector_norm(vector_subtract(s_i, P_i))
        
    # 5. Total Loss
    total_loss = l_img + params['lambda_nc'] * l_nc + l_scale + params['lambda_proj'] * l_proj
    return total_loss

def densify_gaussians(G_t, G_l, mesh, iteration, params):
    """
    Perform densification every 500 iterations.
    """
    if iteration % 500 != 0:
        return G_t, G_l
        
    # Loosely-bound: Standard 3DGS densification
    G_l = densify_loosely_bound(G_l, params)
    
    # Tightly-bound: Mesh-aligned densification
    new_G_t = []
    for gauss in G_t:
        # Identify close mesh triangle
        face = mesh.get_closest_face(gauss.position)
        
        # Sample point from triangle for new Gaussian position
        new_pos = sample_point_from_triangle(face.vertices)
        
        # Cloning: Copy attributes
        cloned_gauss = Gaussian()
        cloned_gauss.position = new_pos
        cloned_gauss.rotation = gauss.rotation
        cloned_gauss.opacity = gauss.opacity
        cloned_gauss.color = gauss.color
        cloned_gauss.scale = gauss.scale # Attributes copied
        new_G_t.append(cloned_gauss)
        
        # Splitting: Divide scale by 1.6
        split_gauss = Gaussian()
        split_gauss.position = new_pos
        split_gauss.rotation = gauss.rotation
        split_gauss.opacity = gauss.opacity
        split_gauss.color = gauss.color
        split_gauss.scale = vector_scale(gauss.scale, 1.0 / 1.6)
        new_G_t.append(split_gauss)
        
    # Replace or merge G_t with new_G_t (implementation detail: typically replace or add)
    # The source says "initialize new Gaussian splats", implying addition or replacement.
    # Standard 3DGS replaces the parent with children.
    G_t = new_G_t
    
    return G_t, G_l
```

## Subtopics

### [Budget-Centered Training](9682-budget-centered-training.md)
Constrains primitive count to a target knee of the capacity curve, replacing gradient-based pruning with a redundancy map.

## Operating characteristics
*   **Time Complexity:** Densification occurs every 500 iterations [S1]. The projection loss requires finding the closest mesh face for each tightly-bound Gaussian, which adds overhead proportional to the number of tightly-bound Gaussians and mesh complexity [S1].
*   **Memory Complexity:** Requires storing the mesh geometry and maintaining two sets of Gaussians ($G^t$ and $G^l$) [S1].
*   **Performance:** Evaluated on Mip-NeRF360 and Deep Blending datasets [S1]. Trained for 30,000 iterations [S1].
*   **Strengths:** Prevents Gaussian splats from drifting away from the mesh surface and controls scale expansion, leading to high-quality rendering aligned with explicit geometry [S1].
*   **Limitations:** Relies on the quality of the input mesh (extracted via LTM [S1]). The projection loss formula in the source is non-standard and may require careful interpretation of vector definitions [S1].

## Background
Originated in **MeshGS: Adaptive Mesh-Aligned Gaussian Splatting for High-Quality Rendering** [S1]. It builds on the original 3DGS [22] and SuGaR [13] by introducing a dual-strategy for Gaussian splats based on their proximity to a mesh surface. The method uses LTM [7] for mesh extraction and decimation [S1].

## Sources
[S1] MeshGS: Adaptive Mesh-Aligned Gaussian Splatting for High-Quality Rendering, https://arxiv.org/abs/2410.08941v1, fetched 2026-10-26
