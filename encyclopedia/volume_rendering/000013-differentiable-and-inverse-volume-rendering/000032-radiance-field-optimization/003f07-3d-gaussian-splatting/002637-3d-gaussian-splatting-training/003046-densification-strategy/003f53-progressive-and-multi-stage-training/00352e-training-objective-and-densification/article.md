# Training Objective and Densification

Also called: MeshGS Training, Tightly/Loosely-bound Training

## Overview
This method defines the loss function and densification strategy for **MeshGS** (Adaptive Mesh-Aligned Gaussian Splatting), a variant of 3D Gaussian Splatting that aligns primitives with an explicit mesh surface [S1]. It solves the problem of Gaussian splats drifting away from geometry or expanding excessively by introducing a projection loss and scale regularization for "tightly-bound" splats [S1]. It differs from standard 3DGS by decoupling the densification strategy: loosely-bound splats follow the original gradient-based 3DGS rules, while tightly-bound splats are densified via mesh-sampling and constrained by mesh-aware losses [S1].

## How it works
**Inputs:**
*   Rendered image $\hat{I}_i$ and ground truth image $I_i$.
*   Set of tightly-bound Gaussian splats $G^t$ and loosely-bound splats $G^l$.
*   Mesh $M$ consisting of faces $F$, where each face $f_j$ has vertices $v_i$ and normal $n_f$.
*   For each Gaussian $i \in G^t$: center $P_i$, scale parameters $S_i = (s_1, s_2, s_3)$, and rotation matrix $R_i$.

**Outputs:**
*   Total loss $L$ for backpropagation.
*   Updated set of Gaussian splats (via densification).

**Data Structures:**
*   **Gaussian Splat:** Contains position $P_i$, scale $S_i$, rotation $R_i$, opacity, and color.
*   **Mesh Face:** Defined by three vertices and a computed normal vector.

**Parameters:**
*   $\lambda_{img} = 0.001$ [S1]
*   $\lambda_{nc} = 0.1$ [S1]
*   $\lambda_{min} = 0.1$ [S1]
*   $\lambda_{max} = 10$ [S1]
*   $\lambda_{proj} = 50$ [S1]
*   $\rho = 0.1$ (scale threshold) [S1]
*   $d_{th} = 0.01$ (distance threshold to distinguish tightly/loosely bound) [S1]
*   Split scale factor: $1.6$ [S1]
*   Training iterations: $30,000$ [S1]
*   Densification frequency: Every $500$ iterations [S1]

**Loss Functions:**

1.  **Image Loss ($L_{img}$):** Combines L1 loss and D-SSIM [S1].
    $$L_{img} = (1-\lambda_{img})L_{1}(\hat{I}_{i},I_{i}) + \lambda_{img}SSIM(\hat{I}_{i}, I_{i})$$

2.  **Normal Consistency Loss ($L_{nc}$):** Aligns splat normals with mesh face normals [S1].
    *   Estimate splat normal $n_i$: Identify the minimum scale axis in $S_i$. Apply rotation $R_i$ to transform this axis into world coordinates.
    $$L_{nc} = \sum_{i \in G^{t},f \in F} (1 - n_{i} \cdot n_{f})$$

3.  **Scale Regularization Loss ($L_{scale}$):** Constrains the scale of tightly-bound splats to prevent excessive expansion relative to the mesh [S1].
    $$L_{scale} = \sum_{i \in G^{t}}(\lambda_{min}|\text{min}(s_{1}, s_{2}, s_{3})| + \lambda_{max}|\text{max}(s_{1}, s_{2}, s_{3}) - \rho|)$$

4.  **Projection Loss ($L_{proj}$):** Pulls splat centers to the closest point on the mesh surface [S1].
    *   Calculate projection $s_i$ of center $P_i$ onto face $f_j$ with vertex $v_i$:
    $$s_{i} = P_{i} + ((P_{i} - v_{i}) \cdot f_{j}) \cdot f_{j}$$
    $$L_{proj} = \sum_{i \in G^{t}}\parallel s_{i} - P_{i} \parallel _{2}$$

5.  **Total Loss ($L$):**
    $$L = L_{img} + \lambda_{nc} L_{nc} + L_{scale} +\lambda_{proj} L_{proj}$$

**Densification Strategy:**
Executed every 500 iterations [S1].

1.  **Classification:** Classify splats into $G^t$ (tightly-bound) or $G^l$ (loosely-bound) based on distance to mesh. If distance $> d_{th}$, classify as $G^l$ [S1].
2.  **Loosely-bound Splats ($G^l$):** Apply the original 3DGS densification strategy (gradient-based cloning and splitting) [S1].
3.  **Tightly-bound Splats ($G^t$):**
    *   **Cloning:** Identify splats requiring cloning. Sample a new point from the corresponding mesh triangle. Initialize a new Gaussian at this sampled position. Copy additional attributes (opacity, color, rotation) from the parent [S1].
    *   **Splitting:** Identify splats requiring splitting. Sample a new point from the corresponding mesh triangle. Initialize a new Gaussian at this position. Set the scale of the new splat to the parent's scale divided by $1.6$ [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_loss(rendered_image, gt_image, splats, mesh):
    # 1. Image Loss
    l1_loss = L1(rendered_image, gt_image)
    ssim_loss = SSIM(rendered_image, gt_image)
    l_img = (1 - 0.001) * l1_loss + 0.001 * ssim_loss

    l_nc = 0.0
    l_scale = 0.0
    l_proj = 0.0

    for splat in splats:
        if splat.is_tightly_bound(mesh, threshold=0.01):
            # 2. Normal Consistency Loss
            # Find min scale axis index
            min_scale_idx = argmin(splat.scale)
            # Get local axis (e.g., [1,0,0] for x, [0,1,0] for y, etc.)
            local_axis = get_basis_vector(min_scale_idx)
            # Transform to world coordinates
            normal = rotate_vector(local_axis, splat.rotation)
            
            # Find closest mesh face and its normal
            face, face_normal = mesh.get_closest_face_and_normal(splat.position)
            l_nc += (1.0 - dot(normal, face_normal))

            # 3. Scale Regularization
            min_s = min(splat.scale)
            max_s = max(splat.scale)
            l_scale += 0.1 * abs(min_s) + 10.0 * abs(max_s - 0.1)

            # 4. Projection Loss
            # Project center to face plane
            # s_i = P_i + ((P_i - v_i) . f_j) * f_j
            # Note: f_j in formula represents the face normal vector for projection
            # Assuming 'face' object provides normal 'n_f' and a vertex 'v'
            projection_vector = dot(splat.position - face.vertex, face.normal)
            projected_point = splat.position + projection_vector * face.normal
            l_proj += norm(projected_point - splat.position)

    total_loss = l_img + 0.1 * l_nc + l_scale + 50.0 * l_proj
    return total_loss

def densify_splats(splats, mesh, iteration):
    if iteration % 500 != 0:
        return

    for splat in splats:
        if splat.is_loosely_bound(mesh, threshold=0.01):
            # Follow original 3DGS strategy
            if splat.should_clone():
                clone_3dgs(splat)
            if splat.should_split():
                split_3dgs(splat)
        else:
            # Tightly-bound strategy
            if splat.should_clone():
                # Sample point from corresponding mesh triangle
                new_pos = sample_point_from_triangle(splat.closest_mesh_triangle)
                new_splat = clone_attributes(splat)
                new_splat.position = new_pos
                add_splat(new_splat)
            
            if splat.should_split():
                # Sample point from corresponding mesh triangle
                new_pos = sample_point_from_triangle(splat.closest_mesh_triangle)
                new_splat = clone_attributes(splat)
                new_splat.position = new_pos
                new_splat.scale = splat.scale / 1.6
                add_splat(new_splat)
```

## Subtopics

## Operating characteristics
*   **Time Complexity:** The projection loss and normal consistency loss require finding the closest mesh face for each tightly-bound splat, which adds overhead compared to standard 3DGS. This is typically done via spatial indexing (e.g., BVH) on the mesh [S1].
*   **Memory Complexity:** No significant additional memory overhead compared to standard 3DGS, aside from storing the mesh geometry.
*   **Performance:** Evaluated on Mip-NeRF360 and Deep Blending datasets [S1]. Trained for 30k iterations [S1].
*   **Strengths:** Prevents splats from drifting away from the surface and controls scale expansion, leading to higher quality rendering aligned with explicit geometry [S1].
*   **Limitations:** Relies on the quality of the input mesh. The original 3DGS densification strategy is still used for loosely-bound splats, which may not align with the mesh [S1].

## Background
This method originates from **MeshGS** (Adaptive Mesh-Aligned Gaussian Splatting for High-Quality Rendering), published in 2024 [S1]. It builds upon the original 3D Gaussian Splatting [22] and integrates concepts from mesh-based representations like LTM [7] and SuGaR [13]. The method addresses the limitation of standard 3DGS in explicitly representing geometry by binding splats to a mesh and using mesh-aware losses.

## Sources
[S1] MeshGS: Adaptive Mesh-Aligned Gaussian Splatting for High-Quality Rendering, https://arxiv.org/abs/2410.08941v1, fetched 2026-10-26
