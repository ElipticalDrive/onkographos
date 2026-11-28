# Combined Implicit and Explicit Inverse Rendering Pipeline

Also called: Two-stage Inverse Rendering Pipeline

## Overview
This method solves physics-based inverse rendering problems by combining implicit and explicit geometric representations to support topology changes and differentiable rendering of complex light-transport effects [S1]. It separates the reconstruction into a coarse implicit stage for robust topology handling and a fine explicit stage for high-fidelity appearance and shape refinement using mesh-based differentiable rendering [S1]. Unlike purely implicit methods (e.g., DVR, IDR) that struggle with complex light transport, or purely explicit methods that require good initialization, this pipeline leverages the topological flexibility of implicit fields and the rendering efficiency of explicit meshes [S1].

## How it works
The pipeline consists of two sequential stages: an implicit optimization stage followed by an explicit refinement stage [S1].

### Inputs and Outputs
**Inputs:**
- A collection of $N$ geometrically calibrated input images $I_1, \dots, I_N$ [S1].
- Camera poses and intrinsics for each image [S1].
- Initial geometric parameters $\theta_g$ for the implicit field [S1].
- Initial optical parameters $\theta_a$ for the appearance field [S1].

**Outputs:**
- A reconstructed 3D mesh $M$ representing the object surface [S1].
- SVBRDF maps (diffuse albedo $k_d$, specular albedo $k_s$, roughness $\alpha$) associated with the mesh [S1].

### Data Structures
1.  **Implicit Field ($\phi$):** A function $\phi: \mathbb{R}^3 \times \Theta_g \to \mathbb{R}$ defining the surface as the zero level set [S1]. In practice, this is implemented as a Signed Distance Function (SDF) controlled by parameters $\theta_g$ [S1].
2.  **Explicit Mesh ($M$):** A triangle mesh defined by vertex positions $V$ and connectivity $F$ [S1]. Vertices are updated during the explicit stage [S1].
3.  **Appearance Field ($B$):** A reflectance field mapping surface points to BRDF parameters [S1]. Implemented as SVBRDF maps storing $k_d, k_s, \alpha$, controlled by parameters $\theta_a$ [S1].
4.  **Lighting Model:** The incident radiance $L_i(x, \omega_i)$ supports environmental illumination, soft shadows, and interreflections [S1].

### Numbered Steps

**Stage 1: Implicit Optimization**
1.  **Initialize:** Define the object surface $M(\theta_g)$ as the zero level set of $\phi(x; \theta_g) = 0$ [S1].
2.  **Render:** For each input image, compute the rendered radiance $L_{rendered}$ using a differentiable renderer that supports implicit surfaces [S1]. The source notes that Monte Carlo sampling over implicit surfaces is challenging, but this stage focuses on coarse shape recovery where topology changes are critical [S1].
3.  **Compute Loss:** Calculate a reconstruction loss between $L_{rendered}$ and the target image [G].
4.  **Optimize:** Update geometric parameters $\theta_g$ to minimize the loss [S1]. This stage allows topology changes without remeshing [S1].
5.  **Extract:** Once convergence is reached, obtain the coarse implicit geometry [S1].

**Stage 2: Explicit Refinement**
1.  **Iso-surface Extraction:** Apply a non-differentiable iso-surface extraction algorithm (specifically **MeshSDF**, a differentiable marching cubes variant) to convert the implicit surface $\phi(x; \theta_g) = 0$ into an explicit triangle mesh $M$ [S1].
2.  **Initialize Appearance:** Initialize SVBRDF maps ($k_d, k_s, \alpha$) for the mesh [S1].
3.  **Differentiable Rendering:** Render the mesh $M$ using physics-based differentiable rendering [S1]. This involves solving the rendering equation and its differential form [S1].
    -   **Rendering Equation:**
        $$ L(x, \omega_o) = \int_{S^2} L_i(x, \omega_i) f_r(x, \omega_i, \omega_o) d\sigma(\omega_i) $$
        where $L_i$ is incident radiance, $f_r$ is the cosine-weighted BRDF, and $\sigma$ is the solid-angle measure [S1].
    -   **Differential Rendering Equation:**
        $$ \frac{d}{d\theta} L(x, \omega_o) = \int_{S^2} \frac{d}{d\theta}[L_i f_r] d\sigma(\omega_i) + \int_{\partial S^2} v_\perp(\omega_i) \Delta L_i f_r d\ell(\omega_i) $$
        where $\partial S^2$ is the domain of jump discontinuity points of $L_i$, $v_\perp$ captures the change rate of the discontinuity boundary, and $\Delta L_i$ is the difference in $L_i$ across the boundary [S1].
4.  **Optimize:** Jointly optimize:
    -   **Shape:** Update mesh vertex positions [S1].
    -   **Appearance:** Update pixel values in SVBRDF maps ($\theta_a$) [S1].
5.  **Output:** Return the final mesh and SVBRDF maps [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def combined_inverse_rendering_pipeline(input_images, camera_poses, initial_theta_g, initial_theta_a, lr_g, lr_a, max_iter_g, max_iter_a):
    """
    Implements the two-stage inverse rendering pipeline.
    
    Args:
        input_images: List of calibrated input images.
        camera_poses: List of camera poses corresponding to input_images.
        initial_theta_g: Initial parameters for the implicit field phi.
        initial_theta_a: Initial parameters for the appearance field B.
        lr_g: Learning rate for geometric optimization.
        lr_a: Learning rate for appearance optimization.
        max_iter_g: Maximum iterations for implicit stage.
        max_iter_a: Maximum iterations for explicit stage.
        
    Returns:
        final_mesh: Reconstructed triangle mesh.
        final_svbrdf: SVBRDF maps (k_d, k_s, alpha).
    """
    
    # --- Stage 1: Implicit Optimization ---
    theta_g = initial_theta_g
    
    for iteration in range(max_iter_g):
        total_loss = 0.0
        grad_theta_g = 0.0
        for image, pose in zip(input_images, camera_poses):
            # Render using implicit representation
            rendered_radiance = render_implicit(phi, theta_g, pose)
            # Compute loss against target image
            image_loss = compute_loss(rendered_radiance, image)
            total_loss += image_loss
            # Accumulate gradient for theta_g
            grad_theta_g += compute_gradient(image_loss, theta_g)
            
        # Standard gradient descent update
        theta_g = theta_g - lr_g * grad_theta_g
        
    # Extract coarse mesh from implicit field using MeshSDF
    coarse_mesh = mesh_sdf_extraction(phi, theta_g)
    
    # --- Stage 2: Explicit Refinement ---
    mesh = coarse_mesh
    theta_a = initial_theta_a
    
    for iteration in range(max_iter_a):
        total_loss = 0.0
        grad_mesh = 0.0
        grad_theta_a = 0.0
        for image, pose in zip(input_images, camera_poses):
            # Render using explicit mesh with physics-based differentiable rendering
            rendered_radiance = render_explicit(mesh, theta_a, pose)
            # Compute loss
            image_loss = compute_loss(rendered_radiance, image)
            total_loss += image_loss
            # Compute gradients for shape and appearance
            grad_mesh += compute_shape_gradient(image_loss, mesh)
            grad_theta_a += compute_appearance_gradient(image_loss, theta_a)
            
        # Update mesh vertices and SVBRDF maps
        mesh = update_mesh_vertices(mesh, grad_mesh)
        theta_a = theta_a - lr_a * grad_theta_a
        
    return mesh, theta_a

# Helper definitions (standard operations)
def render_implicit(phi, theta_g, pose):
    # Evaluates phi at ray-surface intersections or samples volume
    # Returns radiance L(x, omega_o) per pixel
    pass

def compute_loss(rendered, target):
    # Computes reconstruction error (e.g., L2 or perceptual loss)
    pass

def compute_gradient(loss, theta):
    # Computes d(loss)/d(theta) via automatic differentiation
    pass

def mesh_sdf_extraction(phi, theta_g):
    # Applies marching cubes to phi(x; theta_g) = 0
    # Returns triangle mesh with vertices V and faces F
    pass

def render_explicit(mesh, theta_a, pose):
    # Performs physics-based differentiable rendering on mesh
    # Solves rendering equation and differential equation
    pass

def compute_shape_gradient(loss, mesh):
    # Computes d(loss)/d(vertex_positions) using boundary integral term
    pass

def compute_appearance_gradient(loss, theta_a):
    # Computes d(loss)/d(SVBRDF_pixels)
    pass

def update_mesh_vertices(mesh, gradient):
    # Moves vertices along negative gradient direction
    mesh.vertices = mesh.vertices - learning_rate * gradient
    return mesh
```

### Parameters and Typical Values
-   **Max Implicit Iterations:** Not given in the sources.
-   **Max Explicit Iterations:** Not given in the sources.
-   **BRDF Model:** Microfacet BRDF model is used [S1].
-   **Iso-surface Extraction:** MeshSDF is specified [S1].

### Edge Cases and Pitfalls
-   **Topology Changes:** Explicit mesh representations suffer from difficulty in handling topology changes during gradient-based optimization, requiring good initial geometry or frequent remeshing [S1]. The implicit stage mitigates this [S1].
-   **Implicit Rendering Difficulty:** Implicit representations are difficult to use for physics-based differentiable rendering because Monte Carlo sampling over implicitly expressed surfaces is challenging due to difficulty in parameterization [S1]. The pipeline avoids this by switching to explicit meshes for the final rendering stage [S1].
-   **Discontinuity Boundaries:** In differentiable rendering, the boundary integral term in the differential rendering equation is crucial when the parameter $\theta$ controls scene geometry (e.g., mesh vertex positions), as it can be the major contributor to the resulting gradient [S1].

## Operating characteristics
-   **Time Complexity:** Not given in the sources.
-   **Memory Complexity:** Not given in the sources.
-   **Measured Performance:** The pipeline is computationally more efficient than existing differentiable-sphere-tracing based methods (e.g., DVR, IDR) [S1].
-   **Strengths:**
    -   Supports topology changes during optimization [S1].
    -   Supports complex light-transport effects such as environmental illumination, soft shadow, and interreflection [S1].
    -   Enables high-fidelity 3D reconstruction of objects with complex topologies [S1].
-   **Limitations:**
    -   Requires two stages, potentially increasing implementation complexity [G].
    -   Implicit stage may be less accurate for fine details compared to explicit methods [G].
-   **When to Use:** When reconstructing objects with complex topologies that may change during optimization, or when high-fidelity appearance with complex lighting effects is required [G].
-   **When Not to Use:** When computational resources are extremely limited and topology changes are not expected, or when simple lighting models suffice [G].

## Background
-   **Origin:** The method was introduced in the paper "Physics-Based Inverse Rendering using Combined Implicit and Explicit Geometries" [S1].
-   **History:** Inverse rendering has been a long-standing problem in computer vision and graphics [S1]. Previous mesh-based techniques struggled with topology changes, while implicit methods struggled with differentiable rendering [S1]. This method bridges the gap by combining both [S1].
-   **Usage:** Demonstrated on several synthetic and real examples [S1].

## Sources
[S1] Physics‐Based Inverse Rendering using Combined Implicit and Explicit Geometries, https://doi.org/10.1111/cgf.14592, fetched 2026-10-26
