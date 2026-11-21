# Gaussian Frosting

Also called: Frosting

## Overview
Gaussian Frosting is a mesh-based radiance field representation that augments a base triangle mesh with an adaptive layer of 3D Gaussians, termed the "Frosting," to capture fine volumetric details such as hair or grass while maintaining the editability of a surface mesh [S1]. It solves the inefficiency of unconstrained densification in fuzzy areas and the loss of volumetric quality in surface-only methods by allowing users to specify a fixed Gaussian budget ($N$) and adapting the layer thickness locally to material properties [S1]. Unlike SuGaR, which flattens Gaussians onto the mesh surface, Frosting preserves rendering quality for complex materials by maintaining a variable-thickness layer [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of input images with known camera poses (estimated via COLMAP for real scenes or provided for synthetic scenes).
*   **Outputs:** A base triangle mesh, a set of $N$ 3D Gaussians constrained within a frosting layer, and their associated parameters (position, rotation, scale, opacity, spherical harmonics, and barycentric coordinates).

**Data Structures:**
1.  **Base Mesh:** Defined by vertices $V$ and faces $F$.
2.  **Frosting Layer:** Defined by an inner bound and an outer bound at each mesh vertex, creating prismatic cells.
3.  **Gaussian Attributes:** For each Gaussian $g$:
    *   Position $\mu_g$ (3D vector).
    *   Rotation $q_g$ (quaternion).
    *   Scaling factors $s_g$ (3D vector).
    *   Opacity $\alpha_g$ (scalar).
    *   Spherical Harmonics coefficients $c_g$ (view-dependent color).
    *   Barycentric coordinates $\lambda_g$ relative to the prismatic cell containing the Gaussian.
    *   *Note:* The mesh vertices are stored to enable deformation [S1].

**Algorithm Steps:**

1.  **Initialization (Unconstrained 3DGS):**
    *   Initialize a point cloud (random for synthetic, COLMAP-based for real).
    *   Optimize unconstrained 3D Gaussians for 7,000 iterations using standard photometric loss [S1].

2.  **Surface Extraction (SuGaR Regularization):**
    *   Apply SuGaR regularization to the Gaussians from Step 1 for iterations 7,000 to 15,000.
    *   The regularization term uses a parameter $\gamma = 100$ [S1].
    *   Compute an optimal depth parameter $\bar{D}$ [S1].
    *   Extract the base mesh using Poisson surface reconstruction on the regularized Gaussians [S1].

3.  **Frosting Layer Definition:**
    *   For each vertex $v_i$ of the extracted mesh, estimate the inner and outer bounds of the frosting layer based on the distribution of the initial Gaussians retrieved by 3DGS around the mesh [S1].
    *   Define the layer thickness locally: thin for flat surfaces, thick for fuzzy materials [S1].

4.  **Gaussian Population:**
    *   Select a budget $N$ (Typical values: $N=5$ million for real scenes, $N=2$ million for synthetic scenes) [S1].
    *   Randomly sample $N$ Gaussians within the defined frosting layer bounds [S1].
    *   Reset opacities, scaling factors, and rotations to help Gaussians avoid local minima encountered by previous unconstrained Gaussians [S1].

5.  **Frosting Optimization:**
    *   Optimize the $N$ Gaussians for 15,000 additional iterations (Total 30,000 iterations) [S1].
    *   Use the photometric loss from Vanilla 3DGS [S1].
    *   Constrain Gaussians to stay within the frosting layer bounds [S1].
    *   Keep the total number of Gaussians constant during this phase [S1].

6.  **Occlusion Culling (Rendering & Training):**
    *   **Training:** Precompute mesh rasterization results for all viewpoints to accelerate optimization [S1].
    *   **Inference:** Rasterize the base mesh using a differentiable renderer (e.g., Nvdiffrast) to identify visible triangles from the current viewpoint [S1].
    *   Splat only the Gaussians belonging to the Frosting cells corresponding to those visible triangles [S1].

7.  **Deformation and Animation:**
    *   When the base mesh is deformed, Gaussian positions are adjusted using their barycentric coordinates [S1].
    *   **Rotation/Scaling Adjustment:** For a prismatic cell with center $c$ and vertices $v_i$ ($0 \le i < 5$):
        1.  Estimate local transformation at each vertex $v_i$ by computing the rotation and rescaling of the vector $(c - v_i)$ [S1].
        2.  Use the barycentric coordinates of Gaussian $g$ to compute an average transformation at point $\mu_g$ from the transformations of all 6 vertices [S1].
        3.  Adjust the rotation, scaling factors, and spherical harmonics of $g$ by applying this average transformation [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def gaussian_frosting_training(image_paths, camera_poses):
    # Step 1: Unconstrained 3DGS Initialization
    gaussians = initialize_gaussians_from_point_cloud(image_paths, camera_poses)
    for iter in range(7000):
        optimize_gaussians(gaussians, image_paths, camera_poses, loss="photometric")

    # Step 2: Surface Extraction via SuGaR
    for iter in range(7000, 15000):
        # Apply SuGaR regularization with gamma=100
        loss = photometric_loss(gaussians, image_paths, camera_poses) + \
               sugar_regularization(gaussians, gamma=100)
        optimize_gaussians(gaussians, image_paths, camera_poses, loss=loss)
    
    # Extract mesh using Poisson reconstruction
    base_mesh = poisson_surface_reconstruction(gaussians)

    # Step 3: Define Frosting Layer Bounds
    # Estimate inner/outer bounds at each vertex based on initial Gaussian distribution
    frosting_bounds = estimate_layer_bounds(base_mesh, gaussians)

    # Step 4: Populate Frosting Layer
    N = 5000000  # For real scenes; 2000000 for synthetic
    frosting_gaussians = random_sample_gaussians(frosting_bounds, count=N)
    reset_gaussian_params(frosting_gaussians) # Reset opacity, scale, rotation

    # Step 5: Optimize Frosting
    for iter in range(15000):
        # Precompute occlusion culling for training speedup
        visible_triangles = precompute_mesh_rasterization(base_mesh, camera_poses)
        
        # Render only visible Gaussians
        rendered_images = render_frosting(frosting_gaussians, base_mesh, visible_triangles, camera_poses)
        
        loss = photometric_loss(rendered_images, ground_truth_images)
        optimize_gaussians(frosting_gaussians, loss=loss, constraints=frosting_bounds)

    return base_mesh, frosting_gaussians

def render_frosting(gaussians, mesh, visible_triangles, camera):
    # Occlusion Culling
    # Rasterize mesh to find visible triangles
    # Identify Gaussians belonging to cells of visible triangles
    visible_gaussians = filter_gaussians_by_visible_triangles(gaussians, mesh, visible_triangles)
    
    # Splat visible Gaussians
    image = splat_gaussians(visible_gaussians, camera)
    return image

def deform_mesh_and_gaussians(mesh, deformation_function):
    # Deform mesh
    deformed_mesh = deformation_function(mesh)
    
    # Adjust Gaussians
    for g in frosting_gaussians:
        # Get barycentric coordinates of g in its prismatic cell
        lambdas = get_barycentric_coords(g.position, g.cell_vertices)
        
        # Compute average transformation based on vertex transformations
        avg_transform = compute_average_transformation(g.cell_center, g.cell_vertices, lambdas, deformed_mesh)
        
        # Apply transformation to Gaussian parameters
        g.position = apply_transform(avg_transform, g.position)
        g.rotation = apply_rotation(avg_transform, g.rotation)
        g.scale = apply_scale(avg_transform, g.scale)
        g.sh_coeffs = apply_sh_rotation(avg_transform, g.sh_coeffs)
        
    return deformed_mesh, frosting_gaussians
```

**Parameters:**
*   **$N$ (Number of Gaussians):** The budget for the Frosting layer. Typical values: $N=5$ million for real scenes, $N=2$ million for synthetic scenes [S1].
*   **$\gamma$ (Gamma):** Regularization parameter for SuGaR. Value: 100 [S1].
*   **Optimization Iterations:** 7,000 for unconstrained 3DGS, followed by SuGaR regularization until iteration 15,000, then 15,000 additional iterations for Frosting optimization (Total 30,000) [S1].
*   **Frosting Layer Thickness:** A local parameter adapting to material. Thin for flat surfaces, thicker for fuzzy materials [S1].

**Edge Cases and Pitfalls:**
*   **Deformation Limitation:** The current deformation model is piecewise linear and may not capture complex physics-based deformations accurately [S1].
*   **Model Size:** Models are larger than vanilla 3DGS due to the storage of barycentric coordinates and mesh vertices [S1].
*   **Occlusion Culling Dependency:** Performance gains from occlusion culling depend heavily on scene complexity; small, object-centric scenes may show limited gains [S1].

## Operating characteristics
*   **Time Complexity:** Optimization takes 45-90 minutes on a single Nvidia Tesla V100 SXM2 32GB GPU [S1]. This is significantly faster than Adaptive Shells (8 hours on a single GPU for synthetic scenes) [S1].
*   **Memory Complexity:** Higher than vanilla 3DGS due to the storage of mesh vertices and barycentric coordinates for each Gaussian [S1].
*   **Performance:**
    *   **Training Speed:** Occlusion culling provides a 100% increase in training speed [S1].
    *   **Inference Speed:** 30% average increase on low-resolution meshes, ~60% for some views, and at least 10% average increase with high-resolution meshes [S1].
    *   **Quality:** On the Shelly dataset, Frosting achieves 39.84 PSNR, outperforming Vanilla 3DGS (37.66) and Adaptive Shells (36.02) [S1].
*   **Strengths:** Real-time rendering, editable mesh structure, efficient densification in fuzzy areas, faster optimization than comparable surface-based methods [S1].
*   **Limitations:** Piecewise linear deformation model, larger model size than vanilla 3DGS, occlusion culling gains vary by scene [S1].
*   **When to use:** When high-quality rendering of volumetric/fuzzy materials is required alongside mesh editability and real-time performance [S1].
*   **When not to use:** When physics-based deformation accuracy is critical, or when minimal model size is the primary constraint [S1].

## Background
Gaussian Frosting was proposed by A. Guédon and V. Lepetit [S1]. It builds upon the 3D Gaussian Splatting framework [S1] and the SuGaR method for mesh extraction [S1]. It is designed to address the limitations of surface-based methods like SuGaR (loss of volumetric quality) and Adaptive Shells (slow optimization, reliance on neural SDFs) [S1]. The method was presented in the context of real-time rendering and editing of complex 3D effects [S1].

## Sources
[S1] Gaussian Frosting: Editable Complex Radiance Fields with Real-Time Rendering, https://doi.org/10.1007/978-3-031-73650-6_24, fetched 2026-10-26
