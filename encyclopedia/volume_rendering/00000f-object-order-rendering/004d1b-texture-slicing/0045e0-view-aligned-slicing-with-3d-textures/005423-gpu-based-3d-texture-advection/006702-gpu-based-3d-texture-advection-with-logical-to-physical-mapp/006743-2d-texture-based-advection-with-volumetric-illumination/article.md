# 2D Texture-Based Advection with Volumetric Illumination

Also called: 2D texture advection, Dense texture-based visualization of unsteady 3D flow

## Overview
This method provides an interactive technique for the dense texture-based visualization of unsteady 3D flow by achieving efficient 3D texture advection through the mapping of logical 3D memory to physical 2D memory [S1]. It sets itself apart from its sibling [2D Texture-Based Advection with Tile Reordering](19839-2d-texture-based-advection-with-tile-reordering.md) by utilizing a generic volumetric importance function to modify the transfer function, thereby addressing clutter and occlusion by emphasizing important flow regions and reducing visual complexity in less interesting parts [S1]. Unlike the parent [GPU-based 3D Texture Advection with Logical-to-Physical Mapping](19774-gpu-based-3d-texture-advection-with-logical-to-physical-mapp.md) which focuses on the general mapping strategy, this specific implementation integrates volumetric illumination and particle injection schemes directly into the 2D texture pipeline to optimize for real-time performance on specific hardware architectures [S1].

## How it works
The method visualizes unsteady 3D flow by advecting a property field $\rho(\mathbf{x}, t)$ along a vector field $\mathbf{v}(\mathbf{x}, t)$ using a semi-Lagrangian approach. The core innovation is the storage of the advected property and gradient fields in physical 2D textures, while the vector field and particle injection data remain in 3D textures because they are not modified on the GPU [S1].

### Inputs and Outputs
*   **Inputs:**
    *   Unsteady 3D vector field $\mathbf{v}(\mathbf{x}, t)$ stored in a 3D texture.
    *   Particle injection texture stored in a 3D texture.
    *   Current property field $\rho(\mathbf{x}, t)$ stored in a physical 2D texture.
    *   Current gradient field $\nabla \rho(\mathbf{x}, t)$ stored in a physical 2D texture.
    *   Viewport dimensions (typically $600^2$ pixels) [S1].
*   **Outputs:**
    *   Updated property field $\rho(\mathbf{x}, t+1)$ in physical 2D texture.
    *   Updated gradient field $\nabla \rho(\mathbf{x}, t+1)$ in physical 2D texture.
    *   Rendered image with volumetric illumination.

### Data Structures and Layout
*   **Property and Gradient Fields:** Stored in physical 2D textures. Channels use 8-bit fixed-point color representation [S1].
*   **Vector Field:** Stored in 3D textures. Uses 8-bit fixed-point or 16-bit floating-point resolution depending on accuracy requirements. The examples and performance measurements in the source use 8-bit vector fields [S1].
*   **Particle Injection Texture:** Stored in 3D textures. Uses 8-bit color channels [S1].

### Logical-to-Physical Mapping
The method maps logical 3D grid coordinates $(i, j, k)$ to physical 2D texture coordinates $(u, v)$. While the specific bijection function is not explicitly detailed in the provided notes, the text confirms that "mapping between logical and physical memory from Eqs. (3) and (4) can be directly mapped to GPU instructions" [S1]. This mapping allows the GPU to treat the 3D volume as a collection of 2D slices, leveraging the fast read/write access of 2D textures [S1].

### Algorithm Steps
The advection and visualization process proceeds as follows:

1.  **Particle Integration:** For each particle or texel, perform backward advection to find the source position. The integration uses first-order Euler integration:
    $$ \mathbf{x}_{source} = \mathbf{x}_{current} - \Delta t \cdot \mathbf{v}(\mathbf{x}_{current}, t) $$
    First-order Euler integration is deemed sufficient for short streaklets. Higher-order methods can be substituted at the cost of decreased advection speed [S1].
2.  **Coordinate Mapping:** Map the logical 3D source coordinates $\mathbf{x}_{source}$ to physical 2D texture coordinates $(u, v)$ using the logical-to-physical mapping function [S1].
3.  **Texture Fetch:** Sample the property field $\rho$ and vector field $\mathbf{v}$ from the respective textures at the mapped coordinates.
4.  **Advection Update:** Update the property value at the current position based on the sampled source value. This is performed on a texel-by-texel level using fragment programs [S1].
5.  **Gradient Computation:** Compute the gradient field $\nabla \rho$ on the GPU. This is done analogously to the advection step, using fragment programs to calculate spatial derivatives of the property field [S1].
6.  **Volumetric Illumination:** Apply shading to improve the perception of orientation and depth. This includes:
    *   Phong illumination.
    *   Cool/warm shading.
    *   Halos.
    *   Color-based depth cues.
    *   A generic volumetric importance function that modifies the transfer function to address clutter and occlusion [S1].
7.  **Rendering:** Render the updated fields to the viewport. Ping-pong rendering and render-to-texture functionality (e.g., DirectX) are used to update the textures [S1].

### Pseudocode
The following pseudocode represents the core advection and gradient update loop executed on the GPU via fragment programs. It assumes the existence of helper functions for logical-to-physical mapping and texture sampling.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Constants
DOMAIN_SIZE = 128  # Example domain size, typically up to 256^3 for interactive viz
VIEWPORT_SIZE = 600

# Data Structures
# PropertyField: 2D Texture, 8-bit fixed-point
# GradientField: 2D Texture, 8-bit fixed-point
# VectorField: 3D Texture, 8-bit fixed-point or 16-bit float
# ParticleInjection: 3D Texture, 8-bit color

def logical_to_physical_2d(logical_x, logical_y, logical_z):
    """
    Maps logical 3D grid coordinates to physical 2D texture coordinates.
    Implementation details depend on the specific stacking strategy.
    Returns (u, v) in [0, 1] range.
    """
    # The specific bijection is defined in the source's Eqs. (3) and (4)
    # [G] Placeholder for the specific mapping logic derived from the source's
    # logical-to-physical memory mapping equations.
    pass

def sample_vector_field(x, y, z):
    """
    Samples the 3D vector field texture at logical coordinates (x, y, z).
    """
    # [G] Standard 3D texture sampling with linear interpolation
    pass

def sample_property_field(u, v):
    """
    Samples the 2D property field texture at physical coordinates (u, v).
    """
    # [G] Standard 2D texture sampling
    pass

def euler_integrate_backward(x, y, z, vx, vy, vz, dt):
    """
    Performs first-order Euler integration backward in time.
    """
    new_x = x - dt * vx
    new_y = y - dt * vy
    new_z = z - dt * vz
    return new_x, new_y, new_z

def compute_gradient(u, v, property_field):
    """
    Computes the gradient of the property field at physical coordinates (u, v).
    Uses finite differences on the 2D texture.
    """
    # [G] Standard finite difference gradient computation on 2D texture
    pass

def apply_volumetric_importance(property_value, gradient_magnitude):
    """
    Modifies the transfer function using a generic volumetric importance function.
    Emphasizes important flow regions and reduces visual complexity.
    """
    # [G] Implementation of the importance function described in [S1]
    pass

def advect_and_render():
    """
    Main loop for advection and rendering.
    """
    # 1. Advect Property Field
    for each texel (u, v) in PropertyField:
        # Map physical 2D to logical 3D
        (x, y, z) = physical_to_logical_2d(u, v)
        
        # Sample vector field at current position
        (vx, vy, vz) = sample_vector_field(x, y, z)
        
        # Backward Euler integration
        (x_src, y_src, z_src) = euler_integrate_backward(x, y, z, vx, vy, vz, dt)
        
        # Map source logical 3D to physical 2D
        (u_src, v_src) = logical_to_physical_2d(x_src, y_src, z_src)
        
        # Sample property at source
        rho_src = sample_property_field(u_src, v_src)
        
        # Update property field
        PropertyField[u, v] = rho_src

    # 2. Compute Gradients
    for each texel (u, v) in GradientField:
        grad = compute_gradient(u, v, PropertyField)
        GradientField[u, v] = grad

    # 3. Render with Volumetric Illumination
    for each pixel in Viewport:
        # Ray casting or slice-based rendering
        # Apply Phong, halos, depth cues, and importance function
        color = render_with_illumination(PropertyField, GradientField)
        output_pixel(color)
```

### Parameters
*   **Viewport Size:** $600^2$ pixels [S1].
*   **Domain Size:** Performance measurements use $128^3$. Maximum property field size for interactive visualization is $256^3$ [S1].
*   **Integration Order:** First-order Euler integration [S1].
*   **Texture Precision:** 8-bit fixed-point for property/gradient fields; 8-bit fixed-point or 16-bit floating-point for vector fields [S1].

### Edge Cases and Pitfalls
*   **Stacking Direction Reordering:** Reordering the stacking direction is slow, causing a rendering bottleneck. However, switching stacks occurs infrequently in interactive applications, so it typically does not disturb the user [S1].
*   **Hardware Dependency:** The OpenGL implementation of the 3D texture-based method is significantly slower on NVIDIA GPUs due to slow `glCopyTexSubImage3D`. The 2D texture-based method avoids this by using 2D textures, which have fast read/write access independent of GPU architecture [S1].

## Operating characteristics
*   **Time Complexity:** Advection speed is roughly proportional to the number of texels in the domain [S1].
*   **Memory Complexity:** Property and gradient fields are stored in 2D textures, while vector fields are stored in 3D textures. The exact memory footprint depends on the domain size and channel precision (8-bit vs 16-bit) [S1].
*   **Measured Performance:**
    *   **ATI Radeon X800 (600^2 viewport, 128^3 domain, unsteady flow):**
        *   Advection only: 34.5 fps [S1].
        *   Advection & Phong: 22.5 fps [S1].
        *   Advection & Volrend w/ early ray: 34.5 fps [S1].
        *   Advection & Volrend w/o early ray: 19.7 fps [S1].
    *   **NVIDIA GeForce 6800 Ultra (600^2 viewport, 128^3 domain, unsteady flow):**
        *   Advection only: 29.1 fps [S1].
        *   Advection & Phong: 19.7 fps [S1].
        *   Advection & Volrend w/ early ray: 29.1 fps [S1].
        *   Advection & Volrend w/o early ray: 12.7 fps [S1].
    *   **Comparison with 3D Texture Advection:** On ATI Radeon X800, the 2D texture-based method outperforms the 3D texture-based method [WE04]. On NVIDIA GeForce 6800 Ultra, the OpenGL implementation of the 3D method is significantly slower (e.g., 2.0 fps for 256^3 field, advection only) due to `glCopyTexSubImage3D` [S1].
*   **Strengths:** Fast read/write access to 2D textures outweighs the overhead of mapping texture coordinates from logical 3D space to physical 2D space. Interactive visualization is feasible with property fields up to $256^3$ [S1].
*   **Limitations:** Reordering the stacking direction is slow. Unsteady flow requires transferring a new 3D texture for the vector field for each frame, which reduces rendering speed compared to steady flow [S1].

## Background
The method was presented by D. Weiskopf, T. Schafhitzel, and T. Ertl in 2005 as part of the Eurographics Symposium on Visualization [S1]. It builds on the parent technique of GPU-based 3D texture advection with logical-to-physical mapping, specifically optimizing for dense texture-based visualization of unsteady 3D flow. The work addresses the perceptual issues of occlusion and spatial perception by incorporating real-time volumetric illumination and interactive capabilities [S1].

## Sources
[S1] Real-Time Advection and Volumetric Illumination for the Visualization of 3D Unsteady Flow, https://doi.org/10.2312/vissym/eurovis05/013-020, fetched 2026-10-30
