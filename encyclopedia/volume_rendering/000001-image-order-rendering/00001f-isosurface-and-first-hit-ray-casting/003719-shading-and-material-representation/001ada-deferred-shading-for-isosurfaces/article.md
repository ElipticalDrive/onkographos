# Deferred Shading for Isosurfaces

Also called: Curvature Estimation, Deferred Shading with Differential Properties, Differential Surface Properties Computation, Gradient and Hessian Estimation, Image-Space Shading

## Overview
Deferred Shading for Isosurfaces is a multi-pass image-space technique that computes surface properties (position, gradient, Hessian, curvature) in separate rendering passes and uses them in a final shading pass [S1]. It solves the problem of high-quality isosurface visualization by decoupling shading from volume sampling, calculating lighting only for visible pixels using pre-computed differential property images [S1]. It distinguishes itself from direct volume rendering by applying shading only after the opaque volume is determined, and from immediate shading methods by storing full surface attributes (gradient, Hessian, curvature) in intermediate floating-point buffers rather than computing them on-the-fly during a single ray-march [S1].

## How it works
The method operates on the intersection point $\mathbf{p}$ found by the parent method. It extends the parent's principles by computing first-order (gradient) and second-order (Hessian) differential properties in image space to derive surface normals and principal curvures for isosurfaces [S1].

### Inputs and Outputs
*   **Input:** Scalar volume data $f(\mathbf{x})$, typically stored in a 3D texture.
*   **Output:** Final shaded image.
*   **Intermediate Buffers:**
    *   Position image ($\mathbf{pos}$): 3D coordinates of the isosurface intersection.
    *   Gradient image ($\mathbf{g}$): RGB floating-point image storing $\nabla f$.
    *   Hessian image ($\mathbf{H}$): Stored in two RGB floating-point images, storing the six unique components of $\nabla^2 f$.
    *   Curvature images: Two floating-point targets storing principal curvatures $\kappa_1, \kappa_2$ and directions $\mathbf{e}_1, \mathbf{e}_2$.

### Data Structures and Layout
1.  **Gradient ($\mathbf{g}$):** Stored in a single RGB floating-point image. Each pixel $(u,v)$ contains $(\frac{\partial f}{\partial x}, \frac{\partial f}{\partial y}, \frac{\partial f}{\partial z})$ evaluated at the hit point [S1].
2.  **Hessian ($\mathbf{H}$):** Due to symmetry ($H_{ij} = H_{ji}$), only six unique components are calculated. These are stored in two RGB floating-point images [S1].
3.  **Curvature:** Magnitudes and directions are rendered to two floating-point targets [S1].

### Numbered Steps

1.  **Pass 1: Ray-Casting (Position)**
    *   **Input:** Volume texture.
    *   **Output:** Position image ($\mathbf{pos}$).
    *   **Action:** Perform ray-casting to find the isosurface intersection $\mathbf{p}$ for each pixel. Store $\mathbf{p}$ in the position buffer [S1].

2.  **Pass 2: Gradient Computation**
    *   **Input:** Position image ($\mathbf{pos}$), Volume texture.
    *   **Output:** Gradient image ($\mathbf{g}$).
    *   **Action:** Compute the gradient $\mathbf{g} = \nabla f$ in three rendering passes (one per component). In each pass, evaluate a tri-cubic B-spline convolution sum to compute one first-order partial derivative. This requires eight texture fetches from the 3D volume texture and three fetches from 1D filter weight textures [S1].
    *   **Formula:** $\mathbf{n} = \frac{\mathbf{g}}{|\mathbf{g}|}$ (used for normal in shading) [S1].

3.  **Pass 3: Hessian Computation**
    *   **Input:** Position image ($\mathbf{pos}$), Volume texture.
    *   **Output:** Hessian image ($\mathbf{H}$).
    *   **Action:** Compute the Hessian $\mathbf{H} = \nabla \mathbf{g}$ in six rendering passes (one per unique second partial derivative). Each pass uses either eleven or fourteen texture fetches [S1].

4.  **Pass 4: Curvature Computation**
    *   **Input:** Gradient image ($\mathbf{g}$), Hessian image ($\mathbf{H}$).
    *   **Output:** Principal curvatures $\kappa_1, \kappa_2$ and directions $\mathbf{e}_1, \mathbf{e}_2$.
    *   **Action:** Compute principal curvature magnitudes and directions in a single rendering pass using the gradient and Hessian images [S1].
    *   **Formula:** The shape operator $\mathbf{S}$ is defined as the tangent space projection of the normalized Hessian:
        $$ \mathbf{S} = \mathbf{P}^T \frac{\mathbf{H}}{|\mathbf{g}|} \mathbf{P}, \quad \mathbf{P} = \mathbf{I} - \frac{\mathbf{g}\mathbf{g}^T}{|\mathbf{g}|^2} $$
        The principal curvatures $\kappa_{1,2}$ are the eigenvalues of $\mathbf{S}$, computed as:
        $$ \kappa_{1,2} = \frac{1}{2} \left( \text{trace}(\mathbf{A}) \pm \sqrt{\text{trace}(\mathbf{A})^2 - 4 \det(\mathbf{A})} \right) $$
        where $\mathbf{A}$ is the $2 \times 2$ matrix representation of $\mathbf{S}$ in the tangent basis [S1].
        The principal directions $\mathbf{e}_i$ are computed as:
        $$ \mathbf{e}_i = \kappa_i \mathbf{u} + (\kappa_i + a_{22} - a_{11}) \mathbf{v} $$
        where $\mathbf{u}, \mathbf{v}$ are basis vectors of the tangent plane and $a_{11}, a_{22}$ are elements of $\mathbf{A}$ [S1].

5.  **Pass 5: Shading**
    *   **Input:** Position ($\mathbf{pos}$), Gradient ($\mathbf{g}$), Curvatures ($\kappa_{1,2}, \mathbf{e}_{1,2}$).
    *   **Output:** Final Image.
    *   **Action:** Render the shaded isosurface using the property images. Apply shading models such as Blinn-Phong or tone shading using the normal vector derived from $\mathbf{g}$ [S1]. Apply curvature-based effects (color mapping, flow advection, NPR) [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pass 1: Ray-Casting
def pass1_ray_casting(volume_texture):
    # Initialize position buffer
    pos_buffer = create_floating_point_image(width, height, channels=3)
    
    for each pixel (u, v):
        ray = generate_ray(u, v)
        hit_point = ray_cast_isosurface(ray, volume_texture)
        if hit_point is not None:
            pos_buffer[u, v] = hit_point
        else:
            pos_buffer[u, v] = 0.0 # Or transparent marker
    return pos_buffer

# Pass 2: Gradient (3 passes, one per component)
def pass2_gradient(pos_buffer, volume_texture):
    grad_buffer = create_floating_point_image(width, height, channels=3)
    
    # Compute dx
    grad_buffer[:,:,0] = compute_tricubic_derivative(pos_buffer, volume_texture, axis=0)
    # Compute dy
    grad_buffer[:,:,1] = compute_tricubic_derivative(pos_buffer, volume_texture, axis=1)
    # Compute dz
    grad_buffer[:,:,2] = compute_tricubic_derivative(pos_buffer, volume_texture, axis=2)
    
    return grad_buffer

def compute_tricubic_derivative(pos_buffer, volume_texture, axis):
    # Single pass for one component
    output = create_floating_point_image(width, height)
    for each pixel (u, v):
        p = pos_buffer[u, v]
        # Evaluate tri-cubic B-spline convolution sum
        # 8 fetches from 3D volume, 3 fetches from 1D filter weights
        derivative = tricubic_convolution_gradient(p, volume_texture, axis)
        output[u, v] = derivative
    return output

# Pass 3: Hessian (6 passes)
def pass3_hessian(pos_buffer, volume_texture):
    # Hessian has 6 unique components: Hxx, Hxy, Hxz, Hyy, Hyz, Hzz
    # Stored in two RGB images
    hess1 = create_floating_point_image(width, height, channels=3)
    hess2 = create_floating_point_image(width, height, channels=3)
    
    # Passes for Hxx, Hxy, Hxz
    hess1[:,:,0] = compute_tricubic_second_derivative(pos_buffer, volume_texture, axis1=0, axis2=0)
    hess1[:,:,1] = compute_tricubic_second_derivative(pos_buffer, volume_texture, axis1=0, axis2=1)
    hess1[:,:,2] = compute_tricubic_second_derivative(pos_buffer, volume_texture, axis1=0, axis2=2)
    
    # Passes for Hyy, Hyz, Hzz
    hess2[:,:,0] = compute_tricubic_second_derivative(pos_buffer, volume_texture, axis1=1, axis2=1)
    hess2[:,:,1] = compute_tricubic_second_derivative(pos_buffer, volume_texture, axis1=1, axis2=2)
    hess2[:,:,2] = compute_tricubic_second_derivative(pos_buffer, volume_texture, axis1=2, axis2=2)
    
    return hess1, hess2

def compute_tricubic_second_derivative(pos_buffer, volume_texture, axis1, axis2):
    # Single pass for one component
    # 11 or 14 texture fetches
    output = create_floating_point_image(width, height)
    for each pixel (u, v):
        p = pos_buffer[u, v]
        second_deriv = tricubic_convolution_hessian(p, volume_texture, axis1, axis2)
        output[u, v] = second_deriv
    return output

# Pass 4: Curvature
def pass4_curvature(grad_buffer, hess1, hess2):
    # Inputs: grad (RGB), Hessian components in hess1 (RGB) and hess2 (RGB)
    # Output: Curvature magnitudes and directions
    
    kappa1_buffer = create_floating_point_image(width, height)
    kappa2_buffer = create_floating_point_image(width, height)
    e1_buffer = create_floating_point_image(width, height, channels=3)
    e2_buffer = create_floating_point_image(width, height, channels=3)
    
    for each pixel (u, v):
        g = grad_buffer[u, v]
        # Reconstruct Hessian matrix from components
        H = reconstruct_hessian(hess1[u, v], hess2[u, v])
        
        # Compute Shape Operator S
        g_norm = length(g)
        if g_norm < epsilon:
            kappa1_buffer[u, v] = 0
            kappa2_buffer[u, v] = 0
            continue
            
        P = I - (g * g^T) / (g_norm^2)
        S = P^T * (H / g_norm) * P
        
        # Project S to 2x2 tangent basis matrix A
        A = project_to_tangent_basis(S, g)
        
        # Compute eigenvalues (principal curvatures)
        tr = trace(A)
        det = determinant(A)
        discriminant = tr^2 - 4 * det
        if discriminant < 0:
            discriminant = 0
        sqrt_disc = sqrt(discriminant)
        
        kappa1 = 0.5 * (tr + sqrt_disc)
        kappa2 = 0.5 * (tr - sqrt_disc)
        
        kappa1_buffer[u, v] = kappa1
        kappa2_buffer[u, v] = kappa2
        
        # Compute principal directions
        # e_i = kappa_i * u + (kappa_i + a22 - a11) * v
        # u, v are tangent basis vectors derived from g
        u, v = compute_tangent_basis(g)
        a11 = A[0,0]
        a22 = A[1,1]
        
        e1 = kappa1 * u + (kappa1 + a22 - a11) * v
        e2 = kappa2 * u + (kappa2 + a22 - a11) * v
        
        e1_buffer[u, v] = normalize(e1)
        e2_buffer[u, v] = normalize(e2)
        
    return kappa1_buffer, kappa2_buffer, e1_buffer, e2_buffer

# Pass 5: Shading
def pass5_shading(pos_buffer, grad_buffer, kappa1_buffer, kappa2_buffer, e1_buffer, e2_buffer):
    final_image = create_color_image(width, height)
    
    for each pixel (u, v):
        p = pos_buffer[u, v]
        g = grad_buffer[u, v]
        k1 = kappa1_buffer[u, v]
        k2 = kappa2_buffer[u, v]
        e1 = e1_buffer[u, v]
        e2 = e2_buffer[u, v]
        
        if p == 0: # No hit
            final_image[u, v] = background_color
            continue
            
        n = normalize(g)
        
        # Example: Blinn-Phong Shading
        L = compute_light_direction(p)
        V = compute_view_direction(p)
        H = normalize(L + V)
        
        diffuse = max(dot(n, L), 0)
        specular = pow(max(dot(n, H), 0), shininess)
        
        color = ambient + diffuse * light_color + specular * light_color
        
        # Example: Curvature Color Mapping
        # Map k1, k2 to colors via lookup texture
        curvature_color = lookup_curvature_color(k1, k2)
        
        # Combine
        final_color = color + curvature_color
        
        final_image[u, v] = final_color
        
    return final_image
```

### Parameters and Typical Values
*   **Filter:** Tri-cubic B-spline convolution [S1].
*   **Texture Fetches:** 8 volume + 3 weight for gradient; 11 or 14 for Hessian [S1].
*   **Floating Point Precision:** Used for intermediate buffers (gradient, Hessian, curvature) [S1].

### Edge Cases and Pitfalls
*   **Flow Advection Seams:** Advecting flow along curvature directions may produce visible seams because the orientation of curvature direction vectors is not uniquely defined [S1].
*   **Zero Gradient:** If $|\mathbf{g}| \approx 0$, the normal and shape operator are undefined. Guard against division by zero [G].
*   **Negative Discriminant:** In curvature calculation, if $\text{trace}(\mathbf{A})^2 - 4 \det(\mathbf{A}) < 0$, the eigenvalues are complex. Clamp to 0 or handle as flat surface [G].

## Subtopics

### [Surface shading for thresholded volumes](1159-surface-shading-for-thresholded-volumes.md)
Surface shading for thresholded volumes is a post-processing technique that applies illumination models to isosurfaces extracted from scalar volume data by thresholding. It solves the problem of visualizing the 3D shape and orientation of opaque boundaries within a volume without the computational cost of full ray-marching integration. It distinguishes itself from direct volume rendering by applying shading only after the opaque volume is determined, and from deferred shading methods by often relying on immediate geometric estimates (z-buffer slopes or voxel gradients) rather than storing full surface attributes in intermediate buffers.

### [Normal-based Contextual Shading](6617-normal-based-contextual-shading.md)
Normal-based Contextual Shading is an object-space shading technique applied to isosurfaces to compute surface normals more accurately than image-space methods like Depth-Gradient Shading. It solves the problem of inaccurate normal estimation at voxel boundaries by fitting a local surface approximation to the set of intersection points belonging to the same iso-surface. Unlike its parent, Deferred Shading for Isosurfaces, which computes gradients in volume space, this method operates on the discrete set of closest intersection points stored per pixel. It is distinct from its sibling, Surface shading for thresholded volumes, by explicitly using a larger voxel neighborhood to estimate surface inclination.

## Operating characteristics
*   **Time Complexity:** Image-space complexity, independent of grid size [S1]. Total passes: 1 (Ray-Cast) + 3 (Gradient) + 6 (Hessian) + 1 (Curvature) + 1 (Shading) = 12 passes [S1].
*   **Memory Complexity:** Requires multiple floating-point images (Position, Gradient, Hessian x2, Curvature x2) [S1].
*   **Performance:** On a NVIDIA GeForce 6800, each property calculation pass renders a single screen-aligned quad [S1].
*   **Strengths:** High-quality differential properties (curvature) computed accurately via tri-cubic filtering; decoupled shading allows for complex effects (flow advection, NPR) [S1].
*   **Limitations:** High number of rendering passes; seams in flow advection due to non-unique curvature direction orientation [S1].
*   **When to use:** When high-fidelity curvature-based shading and visualization of isosurface geometry is required, and performance allows for multiple passes.
*   **When not to use:** When real-time performance with limited GPU memory is critical, or when simple normal-based shading suffices.

## Background
The method was introduced in "Real-Time Ray-Casting and Advanced Shading of Discrete Isosurfaces" by [S1]. It builds on the principles of deferred shading and differential geometry applied to volume data. The technique exploits GPU capabilities for image-space processing to compute complex surface properties efficiently.

## Sources
[S1] Real-Time Ray-Casting and Advanced Shading of Discrete Isosurfaces, https://cgl.ethz.ch/people/archive/siggc/publications/eg05.pdf, fetched 2026-10-26
[S2] sebastianlipponer/surface_splatting, https://github.com/sebastianlipponer/surface_splatting, fetched 2026-10-28 (unofficial)
