# Directional Occlusion Shading

Also called: DOS, Directional Occlusion, Directional Occlusion Shading Model

## Overview
Directional Occlusion Shading (DOS) is an image-order shading technique for interactive direct volume rendering that approximates in-scattered radiance using a backward-peaked cone phase function to provide perceptual cues similar to ambient occlusion [S2]. Unlike standard diffuse shading or precomputed ambient occlusion methods, DOS integrates soft shadowing and first-order scattering effects directly during the slice-based ray casting traversal without requiring precomputation [S1, S2]. It is distinguished from its siblings by restricting occlusion computation to a cone oriented toward the viewer, allowing interactive manipulation of camera position, transfer functions, and clipping planes while maintaining interactive frame rates [S2].

## How it works
DOS extends a slice-based 3D texture volume renderer by introducing an additional occlusion buffer to store and compute the fractional visibility (occlusion factor) for each slice [S2]. The method assumes a single light source located at the camera position (ambient radiance) and uses a backward-peaked phase function to model occlusion [S1, S2].

### Inputs and Outputs
*   **Inputs**: 3D scalar volume data, transfer function (mapping scalar values to color and opacity), camera parameters (view and projection matrices), aperture angle $\theta$, inter-slice distance $d$, and sample grid resolution for occlusion computation.
*   **Outputs**: Final color and opacity image in the eye buffer.

### Data Structures
*   **Eye Buffer**: A floating-point buffer (typically 16-bit precision) storing the accumulated color and opacity of the volume [S2].
*   **Occlusion Buffer**: A floating-point buffer storing the fractional visibility (occlusion factor) for each fragment [S2]. Two buffers (`current_occlusion` and `next_occlusion`) are used and swapped after each slice to allow iterative convolution [S2].
*   **Proxy Geometry**: View-aligned slices (quads) intersecting the volume bounding box, generated on the CPU [S2].

### Formulas
The radiance $L(\vec{x}, \vec{\omega})$ at a point $\vec{x}$ along a ray with direction $\vec{\omega}$ is given by the radiative transport equation simplified for DOS [S2]:

$$
L(\vec{x}, \vec{\omega}) = T(\vec{x}, \vec{x}_0) L_b(\vec{x}_0, \vec{\omega}) + L_m(\vec{x}, \vec{\omega})
$$

where $L_b$ is background radiance, $T$ is transmittance, and $L_m$ is medium radiance [S2].

The medium radiance $L_m$ integrates scattering contributions [S2]:

$$
L_m(\vec{x}, \vec{\omega}) = \int_{\vec{x}}^{\vec{x}_0} T(\vec{x}, \vec{x}') \sigma_s(\vec{x}') L_i(\vec{x}', \vec{\omega}) d\vec{x}'
$$

where $\sigma_s$ is the scattering coefficient and $L_i$ is in-scattered radiance [S2].

In-scattered radiance is approximated as [S2]:

$$
L_i(\vec{x}', \vec{\omega}) = L_a V(\vec{x}', \vec{\omega})
$$

where $L_a$ is constant ambient radiance and $V$ is fractional visibility [S2].

Fractional visibility is defined as [S2]:

$$
V(\vec{x}', \vec{\omega}) = \int_{4\pi} T(\vec{x}', \vec{x}_{0,i}) \Phi_\theta(\vec{\omega}, \vec{\omega}_i) d\vec{\omega}_i
$$

where $\vec{x}_{0,i}$ is the point where the ray leaves the volume [S2].

The cone phase function $\Phi_\theta$ with aperture angle $\theta \in [0, \pi/2)$ is [S2]:

$$
\Phi_\theta(\vec{\omega}, \vec{\omega}_i) = \begin{cases} \frac{1}{2\pi(1-\cos \theta)} & \text{if } \langle \vec{\omega}, \vec{\omega}_i \rangle > \cos(\theta) \\ 0 & \text{otherwise} \end{cases}
$$

Transmittance is [S2]:

$$
T(\vec{x}_a, \vec{x}_b) = e^{-\tau(\vec{x}_a, \vec{x}_b)}
$$

where optical thickness $\tau(\vec{x}_a, \vec{x}_b) = \int_{\vec{x}_a}^{\vec{x}_b} \sigma_t(\vec{x}) d\vec{x}$ and $\sigma_t = \sigma_a + \sigma_s$ [S2].

### Parameters
*   **Aperture Angle ($\theta$)**: Controls the influence of features at different distances. Range $[0, \pi/2)$. Typical value: $80^\circ$ [S2].
*   **Inter-slice Distance ($d$)**: Distance between view-aligned slices. Typical value: $0.001$ [S2].

### Algorithm Steps

1.  **Preprocessing**: Compute world space proxy geometry (view-aligned slices) by intersecting planes with the volume bounding box [S2].
2.  **Initialization**: Initialize `current_occlusion` and `next_occlusion` buffers to 1.0 (full visibility) [G]. Initialize `eye_buffer` to 0.0 [G].
3.  **Slice Traversal**: For each slice from front to back:
    a.  **Render Eye Buffer**:
        i.   Render the slice quad from the eye's point of view.
        ii.  In the fragment shader, sample the volume at the intersection point to get color $C$ and opacity $\alpha$.
        iii. Read the occlusion factor $V$ from `current_occlusion` at the fragment's screen position.
        iv.  Compute the contribution: $C_{contrib} = C \cdot V \cdot L_a \cdot \alpha$.
        v.   Blend $C_{contrib}$ into `eye_buffer` using front-to-back compositing:
             $$ C_{eye} = C_{eye} + (1 - C_{eye}.a) \cdot C_{contrib} $$
             $$ A_{eye} = A_{eye} + (1 - A_{eye}) \cdot \alpha $$
    b.  **Update Occlusion Buffer**:
        i.   Render the same slice quad again, targeting `next_occlusion`.
        ii.  Compute the correction factor $\vec{c}$ in the vertex shader:
             $$ \vec{c} = P \cdot \begin{bmatrix} x_{scale} \\ y_{scale} \\ z_{scale} \end{bmatrix} $$
             where $P$ is the projection matrix and the vector components are derived from the view-space z-coordinate of the slice vertices [S2]. Specifically, $\vec{c}$ compensates for perspective distortion of the cone base [S2].
        iii. Calculate occlusion extent: $r = d \tan(\theta)$ [S2].
        iv.  Generate sample positions $\vec{p}$ on a disk of radius $r$ using a uniform grid [S2].
        v.   For each sample $\vec{p}$:
             - Project fragment clip space position $\vec{x}_y$ to NDC $\vec{x}_y d$ via perspective division [S2].
             - Calculate NDC sample position: $\vec{p}_d = \vec{x}_y d + \vec{c} \otimes \vec{p}$ [S2].
             - Calculate texture coordinates: $\vec{p}_t = 0.5 \cdot \vec{p}_d + [0.5, 0.5]^T$ [S2].
             - Read previous occlusion value $V_{prev}$ from `current_occlusion` at $\vec{p}_t$.
             - Approximate transmittance using inter-slice distance $d$: $T_{approx} = e^{-\sigma_t \cdot d}$ [G] (or use exact distance if available, but sources note approximation with $d$ increases performance) [S2].
             - Weighted sample: $W = V_{prev} \cdot T_{approx}$ [S2].
        vi.  Average the weighted samples to compute new occlusion factor $V_{new}$ for the current fragment [S2].
        vii. Write $V_{new}$ to `next_occlusion`.
    c.  **Swap Buffers**: Swap `current_occlusion` and `next_occlusion` [S2].
4.  **Finalization**: Copy `eye_buffer` to the frame buffer, converting from floating-point to fixed-point format [S2].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def directional_occlusion_shading(volume, transfer_function, camera, theta, d, sample_grid_res):
    """
    Renders volume using Directional Occlusion Shading.
    """
    # Initialize buffers (16-bit float)
    eye_buffer = Buffer(width, height, channels=4, dtype=float16)
    current_occlusion = Buffer(width, height, channels=1, dtype=float16, init=1.0)
    next_occlusion = Buffer(width, height, channels=1, dtype=float16)
    
    # Precompute proxy geometry (view-aligned slices)
    slices = compute_view_aligned_slices(volume.bounding_box, camera.view_direction, d)
    
    # Projection matrix
    P = camera.projection_matrix
    
    for slice in slices:
        # Pass 1: Render to Eye Buffer
        render_slice_to_eye(slice, volume, transfer_function, current_occlusion, eye_buffer, camera)
        
        # Pass 2: Update Occlusion Buffer
        render_slice_to_occlusion(slice, volume, transfer_function, current_occlusion, next_occlusion, camera, P, theta, d, sample_grid_res)
        
        # Swap buffers
        current_occlusion, next_occlusion = next_occlusion, current_occlusion
        
    # Copy to frame buffer
    frame_buffer = convert_to_fixed_point(eye_buffer)
    return frame_buffer

def render_slice_to_eye(slice, volume, tf, occlusion_buffer, eye_buffer, camera):
    """
    Renders slice into eye buffer, blending color modulated by occlusion.
    """
    for fragment in slice.rasterize(camera):
        # Sample volume
        color, opacity = sample_volume_at_fragment(fragment, volume, tf)
        
        # Read occlusion factor
        occlusion = occlusion_buffer.read(fragment.screen_pos)
        
        # Ambient radiance (assumed constant)
        L_a = 1.0 
        
        # Contribution
        contrib_color = color * occlusion * L_a * opacity
        contrib_alpha = opacity
        
        # Front-to-back compositing
        current_color, current_alpha = eye_buffer.read(fragment.screen_pos)
        
        new_alpha = current_alpha + (1.0 - current_alpha) * contrib_alpha
        if new_alpha > 0.0:
            new_color = current_color + (1.0 - current_alpha) * contrib_color
        else:
            new_color = current_color
            
        eye_buffer.write(fragment.screen_pos, new_color, new_alpha)

def render_slice_to_occlusion(slice, volume, tf, current_occlusion, next_occlusion, camera, P, theta, d, sample_grid_res):
    """
    Renders slice into next occlusion buffer, computing visibility via cone sampling.
    """
    # Compute correction factor c in vertex shader (constant for slice)
    # c = P * [x_scale, y_scale, z_scale]^T derived from view-space z
    c = compute_correction_factor(P, slice.view_space_z)
    
    # Occlusion extent (radius of cone base)
    r = d * tan(theta)
    
    # Generate uniform grid samples on disk of radius r
    samples = generate_uniform_disk_samples(r, sample_grid_res)
    
    for fragment in slice.rasterize(camera):
        # Project fragment to NDC
        clip_pos = fragment.clip_space_pos
        ndc_pos = clip_pos.xyz / clip_pos.w
        
        # Initialize accumulation
        weighted_sum = 0.0
        weight_total = 0.0
        
        for p in samples:
            # Calculate NDC sample position
            # p is in local slice coordinates (xy plane), scaled by r
            # c is component-wise correction
            p_d = ndc_pos.xy + (c.xy * p) # Component-wise multiplication
            
            # Convert to texture coordinates [0,1]
            p_t = 0.5 * p_d + 0.5
            
            # Read previous occlusion
            v_prev = current_occlusion.read(p_t)
            
            # Approximate transmittance using inter-slice distance d
            # sigma_t is extinction coefficient, derived from transfer function or volume
            sigma_t = get_extinction_coefficient_at_fragment(fragment, volume, tf)
            t_approx = exp(-sigma_t * d)
            
            weighted_sum += v_prev * t_approx
            weight_total += t_approx
            
        # Average
        if weight_total > 0:
            v_new = weighted_sum / weight_total
        else:
            v_new = 1.0
            
        next_occlusion.write(fragment.screen_pos, v_new)

def compute_correction_factor(P, z_view):
    """
    Computes correction factor c to compensate for perspective distortion.
    Based on Eq. 12 in [S2].
    """
    # P is 4x4 projection matrix
    # The vector [x_scale, y_scale, z_scale] is derived from the projection of a unit vector
    # in view space. Specifically, it relates to how the cone base scales in screen space.
    # Simplified implementation based on source description:
    # c = P * [1, 1, z_view, 0]^T ? No, source says "using projection matrix P and view space z-component".
    # Eq 12: c = P * [xscale, yscale, zscale]^T
    # xscale, yscale, zscale are likely derived from the projection of basis vectors.
    # For implementation, we assume the source provides the exact derivation or we use the standard
    # perspective correction for cone sampling.
    # Given the complexity and lack of explicit formula for xscale/yscale/zscale in notes,
    # we note that c is computed in vertex shader.
    # A common approach for cone sampling in screen space is:
    # c = [P[0][0], P[1][1], P[2][2]] * z_view ? 
    # We will leave this as a comment as the exact vector construction is not fully explicit in the provided notes beyond "using P and view space z".
    # [G] Implementation note: c is typically [P[0][0]*z, P[1][1]*z, P[2][2]*z] or similar, 
    # but the source Eq 12 shows a matrix-vector product. 
    # We assume the vertex shader computes this per-vertex (constant across slice).
    return [0, 0, 0] # Placeholder for specific matrix math not fully detailed in notes
```

### Edge Cases and Pitfalls
*   **Perspective Distortion**: The cone base size varies with z-coordinate in screen space. This is handled by computing the correction factor $\vec{c}$ in the vertex shader [S2].
*   **Inter-frame Artifacts**: Camera movement can cause artifacts. Mitigated by using a symmetric uniform grid for sample positions aligned with occlusion buffer axes [S2].
*   **Undefined OpenGL Behavior**: MRT implementation reads/writes to the same layered render target, which is undefined by OpenGL spec but works in practice if reads/writes don't reference the same layer [S2].
*   **Single Light Source**: DOS is bound to the slice-based rendering paradigm and supports only a single light source located at the camera position [S1].
*   **Precision**: Floating point buffers are used to increase precision, especially for low-transparency materials and low inter-slice distances [S2].

## Operating characteristics
*   **Time Complexity**: Depends on the number of slices $N_s$ and the number of occlusion samples $N_o$ per slice. Complexity is $O(N_s \cdot N_o)$.
*   **Memory Complexity**: $O(W \cdot H)$ for the eye and occlusion buffers, where $W, H$ are screen dimensions. No precomputation storage required [S1].
*   **Performance**: Interactive frame rates are achieved. Examples from [S2]:
    *   Stag Beetle (1046 slices, $80^\circ$ aperture): 10.6 FPS.
    *   Brain MRI (256x256x160, 1000 slices): 3.7 FPS (DOS) vs 1.6 FPS (Lambertian).
    *   Carp CT (256x288x512, 220 slices): 48.3 FPS (DOS) vs 8.8 FPS (Lambertian).
    *   Visible Male (512x512x302, 962 slices): 5.0 FPS (DOS) vs 25.7 FPS (Lambertian).
    *   Engine Block CT (256x256x128, 679 slices): 5.6 FPS (DOS) vs 5.6 FPS (Lambertian).
    *   Hand CT (244x124x257, 619 slices): 13.3 FPS (DOS) vs 16.0 FPS (Lambertian).
    *   Tree CT (512x512x512, 1563 slices): 59.0 FPS (DOS) vs 29.1 FPS (Lambertian).
*   **Strengths**: Interactive, no precomputation, supports clipping planes and transfer function changes, accentuates concavities and depth discontinuities better than Lambertian shading [S2].
*   **Limitations**: Single light source at camera, bound to slice-based rendering, performance sensitive to sample grid resolution [S2].
*   **MRT vs SRT**: Multiple Render Targets (MRT) increases performance by 44-99% compared to Single Render Target (SRT), especially for low sample counts. Bottleneck shifts to memory bandwidth for high sample counts [S2].

## Background
Directional Occlusion Shading was introduced by Mathias Schott, Vincent Pegoraro, Charles Hansen, Kévin Boulanger, and Kadi Bouatouch in 2009 at the Eurographics/ IEEE-VGTC Symposium on Visualization [S2]. It was published in the Journal of Computer Graphics Forum (Eurographics) in 2009 [S2]. The method is designed for interactive direct volume rendering of CT/MRI data, providing perceptual cues similar to ambient occlusion without the computational cost of precomputation [S2].

## Sources
[S1] Interactive Volume Rendering with Volumetric Illumination, https://doi.org/10.2312/conf/eg2012/stars/053-074, fetched 2026-10-26
[S2] PDF A Directional Occlusion Shading Model for Interactive Direct Volume ..., https://cgl.ethz.ch/teaching/scivis_common/Literature/Schott2009.pdf, fetched 2026-10-29
