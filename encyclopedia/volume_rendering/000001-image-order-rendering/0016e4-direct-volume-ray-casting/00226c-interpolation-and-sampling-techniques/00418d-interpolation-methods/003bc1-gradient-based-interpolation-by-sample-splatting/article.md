# Gradient-Based Interpolation by Sample Splatting

Also called: Gradient-Based Interpolation, Gradient-based Interpolation for Source Radiance

## Overview
Gradient-Based Interpolation by Sample Splatting is a real-time technique for reconstructing source radiance $L_x$ in inhomogeneous, single-scattering media using a sparse set of sample points [S1]. It solves the problem of severe rendering artifacts (e.g., in light shafts and volumetric shadows) that occur when standard interpolation methods, such as Radial Basis Functions (RBF), fail to capture high-frequency shading variations from local or directional illumination [S1]. Unlike RBF methods that rely solely on radiance values, this method uses both radiance values and their spatial gradients at sample points to achieve higher accuracy with fewer samples [S1].

## How it works
The method operates in three main phases: sample evaluation, gradient-based interpolation via splatting, and final ray marching.

### Inputs and Outputs
*   **Inputs:**
    *   Density field $D(x)$ defined on a 3D volume $\mathbb{V}$.
    *   Light source parameters (e.g., point source intensity $I_0$, position $s$).
    *   Extinction coefficient $\sigma_t$.
    *   A set of sample points $\{x_j\}$ determined by a dynamic sampling strategy [S1].
*   **Outputs:**
    *   A 3D volume texture containing the interpolated source radiance $L_x$ at every voxel.
    *   Final image radiance $L$ computed via ray marching through the volume texture.

### Data Structures
*   **Sample $j$:** Defined by a tuple $(x_j, L_{x_j}, \nabla L_{x_j}, R_j)$.
    *   $x_j$: 3D position in the media volume.
    *   $L_{x_j}$: Scalar source radiance at $x_j$.
    *   $\nabla L_{x_j}$: 3D vector gradient of source radiance at $x_j$.
    *   $R_j$: Valid radius (scalar) defining the sphere of influence.
*   **Volume Texture:** 3D texture storing interpolated $L_x$ values.

### Parameters
*   $\Delta_1$: User-defined step size for volume tracing to compute $L_x$. Controls precision of radiance evaluation [S1].
*   $\Delta_2$: User-defined step size for numerical gradient estimation. Controls precision of gradient evaluation [S1].
*   $R_j$: Valid radius. For comparison with RBF methods, set $R_j = 3r_j$, where $r_j$ is the culling radius of the corresponding Gaussian center [S1].

### Step-by-Step Method

1.  **Evaluate Source Radiance and Gradient at Samples**
    For each sample point $x_j$:
    a.  Compute source radiance $L_{x_j}$ by tracing from $x_j$ to the light source $s$. Discretize the path into intervals of $\Delta_1$. Let $\mathbb{U} = \{u_k : u_k = x_j + v k \Delta_1, k = 0, 1, \dots, \lfloor \|s - x_j\| / \Delta_1 \rfloor \}$, where $v = (s - x_j) / \|s - x_j\|$ and $u_k \in \mathbb{V}$.
        $$ L_{x_j} = \frac{I_0}{4\pi \|s - x_j\|^2} \exp\left(-\sigma_t \Delta_1 \sum_{u \in \mathbb{U}} D(u)\right) $$
        [S1]
    b.  Compute the gradient $\nabla L_{x_j}$ using central differences at six points surrounding $x_j$ along the X, Y, and Z axes with interval $\Delta_2$:
        $$ \nabla L_{x_j} = \left( \frac{L_{x_j + \Delta_2 \hat{x}} - L_{x_j - \Delta_2 \hat{x}}}{2\Delta_2}, \frac{L_{x_j + \Delta_2 \hat{y}} - L_{x_j - \Delta_2 \hat{y}}}{2\Delta_2}, \frac{L_{x_j + \Delta_2 \hat{z}} - L_{x_j - \Delta_2 \hat{z}}}{2\Delta_2} \right) $$
        [S1]
        *Note: $L_{x_j \pm \Delta_2 \hat{axis}}$ are computed by repeating step 1a for the offset positions.*

2.  **Interpolate Source Radiance via Sample Splatting**
    Reconstruct $L_x$ for all points $x$ in the volume using the weighted average of first-order Taylor approximations from contributing samples [S1].
    
    The interpolation formula is:
    $$ L_x = \frac{\sum_{j \in S} W_j(x) \left( L_{x_j} + (x - x_j) \cdot \nabla L_{x_j} \right)}{\sum_{j \in S} W_j(x)} $$
    where $S = \{j : \|x - x_j\| < R_j\}$ is the set of samples whose valid sphere contains $x$, and the weight function is:
    $$ W_j(x) = \frac{R_j}{\|x - x_j\|} $$
    [S1]

    **GPU Implementation (Splatting):**
    To efficiently compute this on the GPU, the method uses alpha blending to accumulate the numerator and denominator of the fraction above [S1].
    a.  Align the volume such that the viewing axis is +Z.
    b.  For each sample $j$, intersect its valid sphere (center $x_j$, radius $R_j$) with each X-Y slice of the volume.
    c.  Find the bounding quad of the intersection circle for each slice.
    d.  Group bounding quads by slice.
    e.  For each slice, render the bounding quads with alpha blending enabled.
    f.  For each pixel covered by a quad, evaluate and accumulate:
        *   Numerator term: $W_j(x) \left( L_{x_j} + (x - x_j) \cdot \nabla L_{x_j} \right)$
        *   Denominator term: $W_j(x)$
        *   *Note: $x$ is the world position of the pixel in the current slice.*
    g.  After rendering all quads for a slice, compute $L_x$ at each pixel by dividing the accumulated numerator by the accumulated denominator.
    h.  Render the resulting $L_x$ values into a 3D volume texture [S1].

3.  **Final Ray Marching**
    Perform a ray march to discretely compute the integral in Eq. (2) for final radiance $L$ at the viewer [S1]:
    $$ L = \frac{I_0}{d_{sv}^2} \tau_{sv} + \frac{\Omega}{4\pi} \int_{x_{out}}^{x_{in}} D(x) L_x \tau_{xv} dx $$
    where the first term is direct transmission and the second is single scattered radiance [S1]. $L_x$ is retrieved from the interpolated volume texture.

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_sample_radiance_and_gradient(x_j, density_field, light_source, sigma_t, delta_1, delta_2):
    """
    Computes L_xj and gradient nabla L_xj for a sample point x_j.
    """
    # 1. Compute Source Radiance L_xj
    # Direction from sample to light
    v = normalize(light_source.position - x_j)
    dist = length(light_source.position - x_j)
    
    # Discretize path
    num_steps = floor(dist / delta_1)
    density_sum = 0.0
    for k in range(num_steps + 1):
        u_k = x_j + v * (k * delta_1)
        if u_k is inside volume:
            density_sum += density_field.sample(u_k)
    
    # Transmittance and Radiance
    transmittance = exp(-sigma_t * delta_1 * density_sum)
    L_xj = (light_source.intensity / (4 * pi * dist**2)) * transmittance
    
    # 2. Compute Gradient nabla L_xj using central differences
    # Offsets
    offset_x = vec3(delta_2, 0, 0)
    offset_y = vec3(0, delta_2, 0)
    offset_z = vec3(0, 0, delta_2)
    
    # Evaluate radiance at neighbors
    L_pos_x = compute_sample_radiance_and_gradient(x_j + offset_x, density_field, light_source, sigma_t, delta_1, delta_2)[0]
    L_neg_x = compute_sample_radiance_and_gradient(x_j - offset_x, density_field, light_source, sigma_t, delta_1, delta_2)[0]
    L_pos_y = compute_sample_radiance_and_gradient(x_j + offset_y, density_field, light_source, sigma_t, delta_1, delta_2)[0]
    L_neg_y = compute_sample_radiance_and_gradient(x_j - offset_y, density_field, light_source, sigma_t, delta_1, delta_2)[0]
    L_pos_z = compute_sample_radiance_and_gradient(x_j + offset_z, density_field, light_source, sigma_t, delta_1, delta_2)[0]
    L_neg_z = compute_sample_radiance_and_gradient(x_j - offset_z, density_field, light_source, sigma_t, delta_1, delta_2)[0]
    
    grad_x = (L_pos_x - L_neg_x) / (2 * delta_2)
    grad_y = (L_pos_y - L_neg_y) / (2 * delta_2)
    grad_z = (L_pos_z - L_neg_z) / (2 * delta_2)
    
    return L_xj, vec3(grad_x, grad_y, grad_z)

def gradient_based_interpolation_splatting(samples, volume_dimensions, density_field, light_source, sigma_t, delta_1, delta_2):
    """
    Reconstructs source radiance L_x throughout the volume using splatting.
    Returns a 3D volume texture of L_x.
    """
    # Initialize accumulators for numerator and denominator
    # Using two 3D textures: one for weighted radiance sum, one for weight sum
    numerator_texture = init_3d_texture(volume_dimensions, 0.0)
    denominator_texture = init_3d_texture(volume_dimensions, 0.0)
    
    # For each sample, splat into volume
    for sample in samples:
        x_j = sample.position
        L_xj = sample.radiance
        grad_L_xj = sample.gradient
        R_j = sample.valid_radius
        
        # Determine bounding quads for each X-Y slice
        # Intersection of sphere (x_j, R_j) with plane z = z_slice
        # Circle center in XY plane: (x_j.x, x_j.y)
        # Radius of circle at slice z: r_slice = sqrt(R_j^2 - (z_slice - x_j.z)^2)
        # If r_slice is real, compute bounding quad
        
        z_min = max(0, x_j.z - R_j)
        z_max = min(volume_dimensions.z, x_j.z + R_j)
        
        for z_slice in range(floor(z_min), ceil(z_max)):
            dz = z_slice - x_j.z
            r_slice_sq = R_j**2 - dz**2
            
            if r_slice_sq < 0:
                continue
                
            r_slice = sqrt(r_slice_sq)
            
            # Bounding quad for the circle in this slice
            # Center: (x_j.x, x_j.y, z_slice)
            # Extent: r_slice in X and Y
            quad_center = vec3(x_j.x, x_j.y, z_slice)
            quad_extent = r_slice
            
            # Render quad to accumulators
            # For each pixel (x, y) in the quad:
            #   x_world = pixel_position(x, y, z_slice)
            #   dist_sq = length(x_world - x_j)^2
            #   dist = sqrt(dist_sq)
            #   if dist < R_j:
            #       W = R_j / dist
            #       approx_L = L_xj + dot(x_world - x_j, grad_L_xj)
            #       numerator_texture[x, y, z_slice] += W * approx_L
            #       denominator_texture[x, y, z_slice] += W
            
            render_quad_to_accumulators(quad_center, quad_extent, z_slice, 
                                        x_j, L_xj, grad_L_xj, R_j,
                                        numerator_texture, denominator_texture)

    # Compute final L_x from accumulators
    result_texture = init_3d_texture(volume_dimensions, 0.0)
    for x, y, z in volume_dimensions:
        num = numerator_texture[x, y, z]
        den = denominator_texture[x, y, z]
        if den > 0:
            result_texture[x, y, z] = num / den
        else:
            result_texture[x, y, z] = 0.0 # Or default value
            
    return result_texture

def render_quad_to_accumulators(quad_center, quad_extent, z_slice, 
                                x_j, L_xj, grad_L_xj, R_j,
                                numerator_texture, denominator_texture):
    """
    Renders a bounding quad to the accumulator textures.
    """
    # Define pixel range covered by the quad
    x_min = quad_center.x - quad_extent
    x_max = quad_center.x + quad_extent
    y_min = quad_center.y - quad_extent
    y_max = quad_center.y + quad_extent
    
    for x in range(floor(x_min), ceil(x_max)):
        for y in range(floor(y_min), ceil(y_max)):
            # Check if pixel is inside the circle
            dx = x - quad_center.x
            dy = y - quad_center.y
            dist_sq = dx**2 + dy**2
            
            if dist_sq <= quad_extent**2:
                # World position of pixel
                x_world = vec3(x, y, z_slice)
                dist = sqrt(dist_sq)
                
                # Avoid division by zero
                if dist < 1e-6:
                    continue
                
                W = R_j / dist
                displacement = x_world - x_j
                approx_L = L_xj + dot(displacement, grad_L_xj)
                
                numerator_texture[x, y, z_slice] += W * approx_L
                denominator_texture[x, y, z_slice] += W
```

### Edge Cases and Pitfalls
*   **Sharp Shading Variations:** RBF-based methods fail with sharp shading variations (e.g., light shafts, volumetric shadows) caused by local/directional/high-frequency illumination, producing severe artifacts [S1]. Gradient-based interpolation handles these better [S1].
*   **Precision:** The precision of numerical evaluation is controlled by user-defined intervals $\Delta_1$ and $\Delta_2$. Smaller values increase accuracy but also computation cost [S1].
*   **Boundary Conditions:** When computing gradients, points outside the volume must be handled (e.g., clamping density to 0 or reflecting). The pseudocode assumes `density_field.sample` handles out-of-bounds queries appropriately.
*   **Division by Zero:** In the interpolation step, if a point $x$ is exactly at a sample location $x_j$, the distance is zero. The weight formula $W_j(x) = R_j / \|x - x_j\|$ becomes undefined. In practice, this is handled by either excluding the exact sample point from the splat accumulation (since the sample value is known) or using a small epsilon. The pseudocode includes a check `if dist < 1e-6: continue`.

## Operating characteristics
*   **Time Complexity:** The method is designed for real-time performance. The sample evaluation is parallelized on the GPU. The splatting step involves rendering bounding quads, which is efficient on modern GPUs. The final ray marching is a standard volume rendering step.
*   **Memory Complexity:** Requires storing the density field, the sample data, and the interpolated volume texture. The number of samples is dynamically determined to minimize shading error, balancing memory and accuracy.
*   **Measured Performance:** In the source, the method achieves 23.4 fps for a dynamic scene with a $128 \times 128 \times 128$ volume [S1].
*   **Accuracy:** Gradient-based interpolation yields a relative error of 4.3% compared to 14.7% for RBF-based interpolation at comparable performance levels [S1].
*   **Strengths:** Captures all-frequency shading effects (glows, light shafts, volumetric shadows) in real-time. No precomputation of light transport is needed. Supports dynamic lighting, media, and scene configurations [S1].
*   **Limitations:** Assumes single scattering. Accuracy depends on the dynamic sampling strategy and the validity of the first-order Taylor approximation within the valid sphere.

## Background
*   **Origin:** The method was proposed in a paper by [S1] to address the limitations of RBF-based interpolation in real-time volumetric rendering.
*   **History:** It builds on the idea of using gradients for interpolation, similar to techniques in computer graphics for texture synthesis and image processing, but adapted for volume rendering. The splatting technique is analogous to [Gautron et al. 2005] [S1].
*   **Usage:** Used in real-time rendering of inhomogeneous, single-scattering media, particularly for visual effects involving light shafts and volumetric shadows.

## Sources
[S1] Gradient‐based Interpolation and Sampling for Real‐time Rendering of Inhomogeneous, Single‐scattering Media, https://doi.org/10.1111/j.1467-8659.2008.01343.x, fetched 2026-10-27
