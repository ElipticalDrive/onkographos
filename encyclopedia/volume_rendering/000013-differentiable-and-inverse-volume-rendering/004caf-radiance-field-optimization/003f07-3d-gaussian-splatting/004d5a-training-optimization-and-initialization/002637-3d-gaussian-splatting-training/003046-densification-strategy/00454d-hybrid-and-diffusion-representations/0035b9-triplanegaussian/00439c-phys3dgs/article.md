# Phys3DGS

Also called: Physically-based 3D Gaussian Splatting for Inverse Rendering

## Overview
Phys3DGS is a physically-based inverse rendering method that resolves the "hidden Gaussian" problem, where Gaussians beneath the surface adversely affect pixel color in volume rendering, by employing deferred rendering and a mesh-3DGS hybrid representation [S1, unofficial]. It differs from standard relightable 3DGS methods by decoupling geometry estimation from material learning via a two-step training process, and from naive deferred approaches by introducing specific regularizations to stabilize BRDF learning and prevent mesh extraction artifacts [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of multi-view images.
*   **Outputs:** A hybrid scene representation consisting of a mesh and associated 3D Gaussians, with learned PBR parameters (albedo $\mathcal{A}$, metalness $\mathcal{M}$, roughness $\mathcal{R}$, normal $\mathcal{N}$) and baked visibility for real-time relighting.

**Data Structures:**
Each 3D Gaussian $i$ in the hybrid representation stores:
*   Center coordinate $\mu_i$.
*   Opacity $\alpha_i$.
*   Scale $s_i$ (and 3D covariance $\Sigma_i$ derived from scale and rotation).
*   Spherical Harmonics (SH) parameters for view-dependent color.
*   Normal vector $\mathbf{n}_i$.
*   BRDF parameters: Albedo $\mathbf{a}_i$, metalness $m_i$, roughness $r_i$.
*   Visibility parameters (SH coefficients for indirect illumination, used during training and baked post-training).

The hybrid representation binds one 3D Gaussian to each triangle of an extracted mesh.

**Numbered Steps:**

1.  **Initial Training:** Train a vanilla 3DGS model [4] on the input images.
2.  **Opacity Culling:** Remove low-opacity Gaussians from the initial model to reduce noise and computational load.
3.  **SDF Regularization:** Apply Signed Distance Function (SDF)-based regularization to the remaining Gaussians. This flattens the Gaussians and pushes their centers toward the implicit surface [S1, unofficial].
    *   *Pitfall:* This step may cause Gaussians near the surface to grow abnormally large, leading to large triangles in mesh extraction and visibility test interference [S1, unofficial].
    *   *Mitigation:* Apply **Masked Opacity Regularization** ($L_o$) to constrain the opacity of Gaussians near the surface, preventing excessive scale growth. The exact formula for $L_o$ is not fully provided in the sources, but it involves a mask $M$ and penalizes opacity deviations [S1, unofficial].
4.  **Mesh Extraction:** Extract a triangular mesh from the regularized 3DGS model using standard techniques (e.g., marching cubes on the SDF field implied by the Gaussians) [S1, unofficial].
5.  **Hybrid Representation Construction:** Assign one 3D Gaussian to each triangle of the extracted mesh. This creates the mesh-3DGS hybrid representation.
6.  **Two-Step PBR Training:**
    *   **Step 6a (Geometry & Base PBR):** Train PBR-related parameters on the hybrid representation. This step focuses on learning accurate geometry and base material properties.
    *   **Step 6b (Refinement):** Further refine the parameters. The sources describe a "deferred learning-based two-step training approach" but do not detail the specific loss functions or iteration counts for the second step beyond the general goal of robust inverse rendering [S1, unofficial].

**Deferred Rendering Pipeline (Inference/Training):**
For each pixel (ray):
1.  **Estimate Depth:** Calculate the estimated depth $\mathcal{D}$ to the surface by weighted summation of opacity and distance for all Gaussians $i$ intersecting the ray:
    $$ \mathcal{D} = \sum_{i \in N} T_i \alpha_i d_i $$
    where $T_i$ is the transmittance up to Gaussian $i$, $\alpha_i$ is the opacity, and $d_i$ is the distance from the camera center to Gaussian $i$ [S1, unofficial].
2.  **Estimate Surface Properties:** Calculate the estimated normal $\mathcal{N}$ and BRDF parameters (Albedo $\mathcal{A}$, Metalness $\mathcal{M}$, Roughness $\mathcal{R}$) using the same weighted summation:
    $$ \{\mathcal{A}, \mathcal{M}, \mathcal{R}, \mathcal{N}\} = \sum_{i \in N} T_i \alpha_i \{\mathbf{a}_i, m_i, r_i, \mathbf{n}_i\} $$
    where $\mathbf{a}_i, m_i, r_i, \mathbf{n}_i$ are the per-Gaussian albedo, metalness, roughness, and normal [S1, unofficial].
3.  **Lighting Calculation:**
    *   Perform Fibonacci sampling on incoming light directions $\omega_i$.
    *   Compute Global Illumination $L_{global}(\omega_i)$ using Spherical Harmonics (SH) approximation [S1, unofficial].
    *   Compute Local Indirect Illumination $L_{local}(\omega_i)$ using SH parameters stored on each Gaussian, aggregated at the estimated surface point similarly to Eqn. 3 [S1, unofficial].
    *   Determine Visibility $V(\omega_i)$:
        *   *During Training:* Compute visibility via ray tracing using a Bounding Volume Hierarchy (BVH). This computation is detached from back-propagation [S1, unofficial].
        *   *Post-Training (Baking):* Bake the visibility onto each Gaussian. Replace the SH coefficients of indirect illumination with the SH coefficients of the baked visibility. This reduces model size and boosts test-time speed by avoiding ray tracing [S1, unofficial].
    *   Combine lighting:
        $$ L(\omega_i) = V(\omega_i) \cdot L_{global}(\omega_i) + L_{local}(\omega_i) $$
4.  **Rendering Equation:** Solve the rendering equation once at the estimated surface point using the estimated BRDF parameters and the computed incoming light $L(\omega_i)$ to obtain the final pixel color [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def phys3dgs_render(ray_origin, ray_direction, gaussians, mesh, bvh, global_sh_coeffs):
    """
    Renders a single ray using the Phys3DGS deferred rendering pipeline.
    gaussians: List of Gaussian objects with attributes: mu, alpha, s, Sigma, n, a, m, r, sh_color, sh_indirect
    mesh: Triangle mesh for visibility baking (used in training) or baked visibility map (used in inference)
    bvh: Bounding Volume Hierarchy for ray tracing (used in training)
    global_sh_coeffs: SH coefficients for global illumination
    """
    
    # 1. Filter Gaussians intersecting the ray
    intersecting_gaussians = [g for g in gaussians if ray_intersects_gaussian(ray_origin, ray_direction, g)]
    
    if not intersecting_gaussians:
        return background_color

    # 2. Deferred Accumulation: Estimate Depth and Surface Properties
    total_transmittance = 1.0
    estimated_depth = 0.0
    estimated_albedo = [0.0, 0.0, 0.0]
    estimated_metalness = 0.0
    estimated_roughness = 0.0
    estimated_normal = [0.0, 0.0, 0.0]
    
    # Sort gaussians by distance to camera for correct transmittance calculation
    intersecting_gaussians.sort(key=lambda g: distance(ray_origin, g.mu))
    
    for g in intersecting_gaussians:
        d_i = distance(ray_origin, g.mu)
        alpha_i = g.alpha
        T_i = total_transmittance
        
        # Weighted sum for depth
        estimated_depth += T_i * alpha_i * d_i
        
        # Weighted sum for BRDF and Normal
        weight = T_i * alpha_i
        estimated_albedo += weight * g.a
        estimated_metalness += weight * g.m
        estimated_roughness += weight * g.r
        estimated_normal += weight * g.n
        
        # Update transmittance
        total_transmittance *= (1.0 - alpha_i)
        
        # Early exit if fully opaque
        if total_transmittance < 1e-4:
            break

    # Normalize accumulated properties
    # Note: The source uses direct summation for properties, implying the weights 
    # are normalized implicitly or the sum of weights is used as a divisor. 
    # Based on Eqn 3, it's a weighted average.
    total_weight_sum = sum(T_i * g.alpha for g in intersecting_gaussians) # Approximate
    
    if total_weight_sum > 1e-6:
        estimated_albedo /= total_weight_sum
        estimated_metalness /= total_weight_sum
        estimated_roughness /= total_weight_sum
        estimated_normal = normalize(estimated_normal)
    else:
        return background_color

    # 3. Lighting Calculation
    # Sample incoming light directions using Fibonacci sampling
    light_directions = fibonacci_sphere(num_samples=16)
    
    incoming_light = [0.0, 0.0, 0.0]
    
    for omega_i in light_directions:
        # Global Illumination via SH
        L_global = evaluate_sh(global_sh_coeffs, omega_i)
        
        # Visibility
        # In inference, use baked visibility. In training, use BVH ray tracing.
        # Assuming baked visibility is stored in Gaussian or looked up from mesh
        V = get_baked_visibility(omega_i, estimated_depth, mesh) # Simplified
        
        # Local Indirect Illumination
        # Aggregated from Gaussians similar to BRDF estimation
        L_local = aggregate_local_indirect_illumination(intersecting_gaussians, omega_i)
        
        # Combine
        L_i = V * L_global + L_local
        incoming_light += L_i
        
    incoming_light /= len(light_directions)
    
    # 4. Solve Rendering Equation
    # Simple diffuse/specular model for demonstration
    # F0 = mix(vec3(0.04), albedo, metalness)
    # ... (Standard PBR BRDF evaluation)
    color = evaluate_pbr_brdf(estimated_albedo, estimated_metalness, estimated_roughness, 
                              estimated_normal, ray_direction, incoming_light)
    
    return color

def train_phys3dgs(images, initial_gaussians):
    """
    Two-step training process.
    """
    # Step 1: Initial Vanilla 3DGS Training
    gaussians = train_vanilla_3dgs(images, initial_gaussians)
    
    # Step 2: Opacity Culling
    gaussians = opacity_cull(gaussians)
    
    # Step 3: SDF Regularization + Masked Opacity Regularization
    gaussians = apply_sdf_regularization(gaussians)
    gaussians = apply_masked_opacity_regularization(gaussians)
    
    # Step 4: Mesh Extraction
    mesh = extract_mesh(gaussians)
    
    # Step 5: Build Hybrid Representation
    hybrid_gaussians = assign_gaussians_to_mesh(mesh, gaussians)
    
    # Step 6: Two-Step PBR Training
    # Step 6a: Train PBR parameters
    hybrid_gaussians = train_pbr_parameters(hybrid_gaussians, images, mesh)
    
    # Step 6b: Refine (Details not specified in sources)
    hybrid_gaussians = refine_pbr_parameters(hybrid_gaussians, images, mesh)
    
    # Post-Training: Bake Visibility
    hybrid_gaussians = bake_visibility(hybrid_gaussians, mesh)
    
    return hybrid_gaussians, mesh
```

## Operating characteristics
*   **Time Complexity:** Training is more expensive than vanilla 3DGS due to mesh extraction, SDF regularization, and the two-step PBR training process. Inference is real-time capable, comparable to standard 3DGS, as the deferred rendering avoids per-Gaussian PBR evaluation and ray tracing is replaced by baked visibility [S1, unofficial].
*   **Memory Complexity:** Higher than vanilla 3DGS due to the storage of mesh data, additional PBR parameters (albedo, metalness, roughness, normal) per Gaussian, and visibility SH coefficients. However, baking visibility reduces the model size at test time by removing the need for dynamic visibility computation [S1, unofficial].
*   **Performance:** Offers significantly better rendering quality under relighting compared to existing relightable 3DGS methods. Provides better rendering quality with real-time rendering speeds compared to SOTA voxel grid-based inverse rendering methods (e.g., TensoIR) [S1, unofficial].
*   **Strengths:** Solves the hidden Gaussian problem; enables real-time relighting; robust geometry learning via mesh hybridization.
*   **Limitations:** Sensitive to mesh extraction quality; requires careful regularization to prevent abnormal Gaussian scales; training instability if deferred rendering is applied naively without the proposed two-step approach [S1, unofficial].

## Background
Phys3DGS was proposed by Euntae Choi and Sungjoo Yoo from the Department of Computer Science and Engineering, Seoul National University, South Korea. It was published on arXiv on September 16, 2024 [S1, unofficial]. The work is motivated by the limitations of existing relightable 3DGS methods, which suffer from quality degradation under novel lighting conditions due to the hidden Gaussian problem. It draws inspiration from SuGaR, which uses a hybrid mesh-Gaussian representation, but differs by demonstrating that simple mesh adoption fails for inverse rendering and proposing a novel deferred learning-based training approach [S1, unofficial].

## Sources
[S1] Phys3DGS: Physically-based 3D Gaussian Splatting for Inverse Rendering, https://doi.org/10.48550/arxiv.2409.10335, fetched 2026-10-26 (unofficial)
