# Training Objective and Densification

Also called: MeshGS Training, Tightly/Loosely-bound Training

## Overview
This method defines the loss function and densification strategy for **MeshGS** (Mesh-Aligned Gaussian Splatting), which integrates an explicit mesh representation into the 3D Gaussian Splatting optimization pipeline [S1]. It solves the problem of Gaussian primitives drifting away from known geometry or expanding uncontrollably by binding a subset of splats ("tightly-bound") to a mesh surface while allowing others ("loosely-bound") to follow standard differentiable rendering optimization [S1]. It is distinguished from standard 3DGS by the introduction of mesh-aware losses (projection, normal consistency, scale regularization) and geometry-constrained densification for tightly-bound primitives [S1].

## How it works

### Inputs and Outputs
*   **Inputs:**
    *   Rendered image $\hat{I}_i$ and ground truth image $I_i$ [S1].
    *   Set of tightly-bound Gaussians $G^t$ and loosely-bound Gaussians $G^l$ [S1].
    *   Mesh $M$ consisting of faces $f_j$ and vertices $v_i$ with face normals $n_f$ [S1].
    *   Scale parameters $S_i = (s_1, s_2, s_3)$ and rotation matrices $R_i$ for each Gaussian [S1].
*   **Outputs:**
    *   Total loss $L$ used for gradient-based optimization [S1].
    *   Updated set of Gaussians after densification (splitting/cloning) [S1].

### Data Structures
*   **Gaussian Primitive:** Contains mean position $P_i$, scale $S_i$, rotation $R_i$, opacity, and color attributes [S1].
*   **Mesh:** Defined by a set of triangular faces $F$, where each face $f_j$ has vertices $v_i$ and a normal $n_f$ [S1].
*   **Classification:** Gaussians are classified as $G^t$ (tightly-bound) or $G^l$ (loosely-bound) based on their distance to the mesh surface. A distance threshold $d_{th} = 0.01$ is used for this classification [S1].

### Formulas
The total loss $L$ is a weighted sum of four terms [S1]:

$$ L = L_{img} + \lambda_{nc} L_{nc} + L_{scale} + \lambda_{proj} L_{proj} $$

1.  **Image Loss ($L_{img}$):** Combines L1 loss and D-SSIM [S1]:
    $$ L_{img} = (1 - \lambda_{img})L_{1}(\hat{I}_{i}, I_{i}) + \lambda_{img}SSIM(\hat{I}_{i}, I_{i}) $$
    Where $\lambda_{img}$ is the SSIM weight [S1].

2.  **Normal Consistency Loss ($L_{nc}$):** Aligns tightly-bound splat normals with mesh face normals [S1]:
    $$ L_{nc} = \sum_{i \in G^{t}, f \in F} (1 - n_{i} \cdot n_{f}) $$
    The splat normal $n_i$ is estimated by identifying the minimum scale axis among $S_i$ and applying the rotation matrix $R_i$ to transform it into world coordinates [S1].

3.  **Scale Regularization Loss ($L_{scale}$):** Constrains the scales of tightly-bound splats to align with the mesh surface geometry [S1]:
    $$ L_{scale} = \sum_{i \in G^{t}} (\lambda_{min}|\text{min}(s_{1}, s_{2}, s_{3})| + \lambda_{max}|\text{max}(s_{1}, s_{2}, s_{3}) - \rho|) $$
    Where $\rho$ is the threshold for the maximum scale [S1].

4.  **Projection Loss ($L_{proj}$):** Ensures tightly-bound splat centers remain close to the mesh surface [S1]:
    $$ s_{i} = P_{i} + ((P_{i} - v_{i}) \cdot f_{j}) \cdot f_{j} $$
    $$ L_{proj} = \sum_{i \in G^{t}} \| s_{i} - P_{i} \|_{2} $$
    Here, $s_i$ is the projection of the Gaussian center $P_i$ onto the closest mesh face $f_j$ defined by vertex $v_i$ [S1].

### Parameters
*   $\lambda_{img} = 0.001$ [S1]
*   $\lambda_{nc} = 0.1$ [S1]
*   $\lambda_{min} = 0.1$ [S1]
*   $\lambda_{max} = 10$ [S1]
*   $\lambda_{proj} = 50$ [S1]
*   $\rho = 0.1$ (scale threshold) [S1]
*   $d_{th} = 0.01$ (distance threshold for binding classification) [S1]
*   Training iterations: 30,000 [S1]

### Densification Strategy
Densification is applied differently based on the Gaussian type [S1]:

1.  **Loosely-bound Gaussians ($G^l$):** Follow the original 3DGS densification strategy [S1]. This involves evaluating metrics (e.g., gradient norm, screen size) and performing split, clone, or prune operations as defined in the parent article [S1].
2.  **Tightly-bound Gaussians ($G^t$):** Use geometry-aware densification [S1]:
    *   **Trigger:** Evaluated every 500 training iterations [S1].
    *   **Splitting:** The primitive is replaced by two new primitives. The scale of the new primitives is divided by a factor of 1.6 [S1]. The new mean positions are initialized by sampling points from the corresponding mesh triangle [S1].
    *   **Cloning:** A copy of the primitive is created. The new mean position is initialized by sampling a point from the corresponding mesh triangle [S1]. Additional attributes are copied to the new Gaussian [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_loss(rendered_image, ground_truth_image, gaussians, mesh):
    """
    Computes the total loss for MeshGS training.
    """
    # 1. Image Loss
    l1_loss = L1(rendered_image, ground_truth_image)
    ssim_loss = SSIM(rendered_image, ground_truth_image)
    lambda_img = 0.001
    l_img = (1 - lambda_img) * l1_loss + lambda_img * ssim_loss

    # Separate tightly-bound (Gt) and loosely-bound (Gl) Gaussians
    # Classification based on distance to mesh surface < d_th (0.01)
    Gt, Gl = classify_gaussians(gaussians, mesh, d_th=0.01)

    l_nc = 0.0
    l_scale = 0.0
    l_proj = 0.0

    if Gt is not empty:
        # 2. Normal Consistency Loss
        # Estimate normal ni for each Gt: min scale axis rotated by Ri
        normals_t = [estimate_normal(g) for g in Gt]
        # Find corresponding mesh face normals nf for each Gt
        face_normals_t = [get_closest_face_normal(g, mesh) for g in Gt]
        lambda_nc = 0.1
        l_nc = sum(1.0 - dot(ni, nf) for ni, nf in zip(normals_t, face_normals_t))

        # 3. Scale Regularization Loss
        lambda_min = 0.1
        lambda_max = 10.0
        rho = 0.1
        for g in Gt:
            scales = g.scales # (s1, s2, s3)
            min_s = min(scales)
            max_s = max(scales)
            l_scale += lambda_min * abs(min_s) + lambda_max * abs(max_s - rho)

        # 4. Projection Loss
        lambda_proj = 50.0
        for g in Gt:
            # Project center Pi to closest point si on mesh face fj
            si = project_point_to_mesh(g.mean, mesh)
            l_proj += norm(si - g.mean)

    total_loss = l_img + lambda_nc * l_nc + l_scale + lambda_proj * l_proj
    return total_loss

def densify_tightly_bound_gaussians(Gt, mesh, iteration):
    """
    Densifies tightly-bound Gaussians using mesh sampling.
    Triggered every 500 iterations.
    """
    if iteration % 500 != 0:
        return Gt

    new_gaussians = []
    for g in Gt:
        # Determine if split or clone is needed based on standard metrics
        # (Assuming standard 3DGS metrics are used to decide action)
        action = evaluate_densification_metric(g)

        if action == 'split':
            # Sample two points from the corresponding mesh triangle
            p1 = sample_point_from_triangle(g.closest_triangle)
            p2 = sample_point_from_triangle(g.closest_triangle)
            
            # Create two new Gaussians
            g1 = Gaussian(mean=p1, scale=g.scale / 1.6, attributes=g.attributes)
            g2 = Gaussian(mean=p2, scale=g.scale / 1.6, attributes=g.attributes)
            new_gaussians.extend([g1, g2])
            
        elif action == 'clone':
            # Sample one point from the corresponding mesh triangle
            p1 = sample_point_from_triangle(g.closest_triangle)
            
            # Create one new Gaussian with copied attributes
            g1 = Gaussian(mean=p1, scale=g.scale, attributes=g.attributes)
            new_gaussians.append(g1)
            
        else:
            # Keep original
            new_gaussians.append(g)
            
    return new_gaussians
```

### Edge Cases and Pitfalls
*   **Precision in Projection:** The projection formula $s_i = P_i + ((P_i - v_i) \cdot f_j) \cdot f_j$ assumes $f_j$ is a unit normal vector. If the mesh normal is not normalized, the projection will be incorrect [S1].
*   **Scale Regularization:** The term $|\text{max}(s_1, s_2, s_3) - \rho|$ can become unstable if scales are very large or negative (though scales are typically positive). The absolute value handles sign, but the threshold $\rho$ must be chosen carefully relative to the scene scale [S1].
*   **Normal Estimation:** Estimating normals from the minimum scale axis assumes the Gaussian is elongated along the surface. If a Gaussian is spherical or misaligned, the normal estimate may be noisy [S1].

## Operating characteristics

*   **Time Complexity:** The addition of mesh-related losses (normal consistency, projection) increases the per-step computation cost compared to standard 3DGS, as it requires finding closest mesh faces and computing projections for all tightly-bound Gaussians [S1]. Densification for $G^t$ requires mesh sampling, which is more expensive than random perturbation [S1].
*   **Memory Complexity:** No significant additional memory overhead beyond storing the mesh and the classification of Gaussians [S1].
*   **Performance:** The method is evaluated on Mip-NeRF360 and Deep Blending datasets over 30k iterations [S1]. It aims to improve rendering quality by maintaining geometric fidelity [S1].
*   **Strengths:** Prevents Gaussian drift from the mesh surface; enforces geometric consistency; allows for high-quality rendering of mesh-based scenes [S1].
*   **Limitations:** Relies on the quality of the input mesh; tightly-bound Gaussians are constrained to the mesh, potentially limiting their ability to represent complex occlusions or non-manifold geometry not captured by the mesh [S1].

## Background

*   **Origin:** MeshGS (Adaptive Mesh-Aligned Gaussian Splatting for High-Quality Rendering) was introduced in 2024 [S1].
*   **History:** It builds upon the original 3D Gaussian Splatting (3DGS) [S1] and other methods like SuGaR that integrate mesh representations [S1]. It specifically addresses the limitation of standard 3DGS in ignoring existing 3D geometry [S1].
*   **Usage:** Used for high-quality rendering of large-scale real-world scenes where an explicit mesh representation is available or can be extracted (e.g., from neural fields) [S1].

## Sources

[S1] MeshGS: Adaptive Mesh-Aligned Gaussian Splatting for High-Quality Rendering, https://arxiv.org/abs/2410.08941v1, fetched 2026-10-26
