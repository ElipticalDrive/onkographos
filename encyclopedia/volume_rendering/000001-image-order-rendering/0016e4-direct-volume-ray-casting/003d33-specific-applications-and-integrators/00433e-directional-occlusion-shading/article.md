# Directional Occlusion Shading

Also called: DOS, Directional Occlusion, Directional Occlusion Shading Model

## Overview
Directional Occlusion Shading (DOS) is an image-order shading technique for interactive direct volume rendering that approximates in-scattered radiance using a backward-peaked cone phase function [S2]. It provides perceptual cues similar to ambient occlusion for solid and transparent surface-like features without requiring precomputation [S2]. Unlike [Half Angle Slicing](8414-cutting-plane-integration.md) which adapts the slicing axis to the light position, DOS is bound to view-aligned slices and assumes a single light source at the camera position [S1]. It differs from [Kniss et al.](5291-customization-volume-integration.md) by using a backward-peaked function to accentuate semi-transparent features rather than a forward-peaked function for translucency [S2].

## How it works
**Inputs:**
*   Volume dataset with transfer function mapping density to absorption ($\sigma_a$), scattering ($\sigma_s$), and color.
*   Camera parameters (view and projection matrices).
*   Aperture angle $\theta \in [0, \pi/2)$ controlling the cone of occlusion [S2].
*   Inter-slice distance $d$ [S2].

**Outputs:**
*   Final color buffer with occlusion shading applied.

**Data Structures:**
*   **Eye Buffer:** 16-bit floating-point texture storing the composited color and opacity of the volume.
*   **Occlusion Buffers:** Two 16-bit floating-point textures (Current and Next) storing the fractional visibility (occlusion factor) for each fragment. These are swapped after each slice [S2].
*   **Proxy Geometry:** View-aligned slices (planes) intersecting the volume bounding box, generated on the CPU [S2].

**Formulas:**
The radiance $L(\vec{x}, \vec{\omega})$ at point $\vec{x}$ along viewing direction $\vec{\omega}$ is:
$$ L(\vec{x}, \vec{\omega}) = T(\vec{x}, \vec{x}_0) L_b(\vec{x}_0, \vec{\omega}) + L_m(\vec{x}, \vec{\omega}) $$
where $T$ is transmittance, $L_b$ is background radiance, and $L_m$ is medium radiance [S2].

Medium radiance $L_m$ integrates scattering:
$$ L_m(\vec{x}, \vec{\omega}) = \int_{\vec{x}}^{\vec{x}_0} T(\vec{x}, \vec{x}') \sigma_s(\vec{x}') L_i(\vec{x}', \vec{\omega}) d\vec{x}' $$
where $\sigma_s$ is the scattering coefficient and $L_i$ is in-scattered radiance [S2].

In-scattered radiance is approximated as:
$$ L_i(\vec{x}', \vec{\omega}) = L_a V(\vec{x}', \vec{\omega}) $$
where $L_a$ is constant ambient radiance and $V$ is fractional visibility [S2].

Fractional visibility $V$ is computed using the cone phase function $\Phi_\theta$:
$$ V(\vec{x}', \vec{\omega}) = \int_{4\pi} T(\vec{x}', \vec{x}_{0,i}) \Phi_\theta(\vec{\omega}, \vec{\omega}_i) d\vec{\omega}_i $$
where $\vec{x}_{0,i}$ is the point where the ray leaves the volume [S2].

The cone phase function is:
$$ \Phi_\theta(\vec{\omega}, \vec{\omega}_i) = \begin{cases} \frac{1}{2\pi(1-\cos \theta)} & \text{if } \langle \vec{\omega}, \vec{\omega}_i \rangle > \cos(\theta) \\ 0 & \text{otherwise} \end{cases} $$
where $\vec{\omega}$ is the viewing direction and $\vec{\omega}_i$ are directions within the cone [S2].

Transmittance is:
$$ T(\vec{x}_a, \vec{x}_b) = e^{-\tau(\vec{x}_a, \vec{x}_b)} $$
where optical thickness $\tau(\vec{x}_a, \vec{x}_b) = \int_{\vec{x}_a}^{\vec{x}_b} \sigma_t(\vec{x}) d\vec{x}$ and $\sigma_t = \sigma_a + \sigma_s$ [S2].

**Algorithm Steps:**
1.  **Preparation:** Compute world space proxy geometry (view-aligned slices) on the CPU by intersecting planes with the volume bounding box [S2].
2.  **Initialize:** Clear Eye Buffer and both Occlusion Buffers. Set Current Occlusion Buffer to all zeros (full visibility) [S2].
3.  **Slice Loop:** For each view-aligned slice from front to back:
    a.  **Render to Eye Buffer:**
        i.   Render the slice geometry from the camera's viewpoint.
        ii.  In the fragment shader, read the occlusion factor $V$ from the Current Occlusion Buffer at the fragment's texture coordinates.
        iii. Compute the sample color $C_{sample}$ and opacity $\alpha_{sample}$ from the volume transfer function.
        iv.  Modulate the ambient contribution: $C_{ambient} = C_{sample} \cdot V \cdot L_a$.
        v.   Perform standard front-to-back compositing into the Eye Buffer:
             $$ C_{out} = C_{in} + C_{ambient} \cdot \alpha_{sample} \cdot (1 - \alpha_{in}) $$
             $$ \alpha_{out} = \alpha_{in} + \alpha_{sample} \cdot (1 - \alpha_{in}) $$
             (Note: Direct background transmission is handled by the standard ray casting loop or initial clear; DOS specifically adds the occlusion-modulated ambient term) [S2].
    b.  **Update Occlusion Buffer:**
        i.   Render the same slice geometry again, targeting the Next Occlusion Buffer.
        ii.  Compute the correction factor $\vec{c}$ in the vertex shader to handle perspective distortion (see Pitfalls).
        iii. In the fragment shader, generate sample positions $\vec{p}$ on a disk with radius $r = d \tan(\theta)$ using a uniform grid [S2].
        iv.  Calculate NDC sample positions: $\vec{p}_d = \vec{x}_y d + \vec{c} \otimes \vec{p}$, where $\vec{x}_y d$ is the fragment's NDC position, $\otimes$ is component-wise multiplication, and $\vec{c}$ is the correction factor [S2].
        v.   Convert to texture coordinates: $\vec{p}_t = 0.5 \cdot \vec{p}_d + [0.5, 0.5]^T$ [S2].
        vi.  Read the previous occlusion factor $V_{prev}$ from the Current Occlusion Buffer at $\vec{p}_t$.
        vii. Compute transmittance $T$ between the current slice and the sample point. The sources state that approximating the distance with the inter-slice distance $d$ is sufficient and increases performance [S2].
        viii. Accumulate the weighted sample: $V_{new} = \frac{1}{N} \sum V_{prev} \cdot T$ [S2].
        ix.  Write $V_{new}$ to the Next Occlusion Buffer.
    c.  **Swap:** Swap Current and Next Occlusion Buffers [S2].
4.  **Finalize:** Copy the Eye Buffer to the frame buffer, converting from floating-point to fixed-point [S2].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def directional_occlusion_shading(volume, camera, aperture_angle, inter_slice_distance, num_samples):
    """
    Renders volume with Directional Occlusion Shading.
    """
    # 1. Setup Buffers
    eye_buffer = create_floating_point_texture(width, height)
    occlusion_current = create_floating_point_texture(width, height, initial_value=1.0) # 1.0 = full visibility
    occlusion_next = create_floating_point_texture(width, height)
    
    # 2. Generate Slices
    slices = generate_view_aligned_slices(volume.bounding_box, camera.view_direction, inter_slice_distance)
    
    # 3. Render Slices
    for slice in slices:
        # Pass A: Update Eye Buffer
        render_slice_to_eye(slice, camera, eye_buffer, occlusion_current, aperture_angle)
        
        # Pass B: Update Occlusion Buffer
        render_slice_to_occlusion(slice, camera, occlusion_current, occlusion_next, aperture_angle, inter_slice_distance, num_samples)
        
        # Swap buffers
        swap(occlusion_current, occlusion_next)
        
    # 4. Final Copy
    copy_to_framebuffer(eye_buffer)

def render_slice_to_eye(slice, camera, eye_buffer, occlusion_buffer, aperture_angle):
    """
    Renders a slice, compositing color modulated by occlusion.
    """
    # Vertex Shader: Compute correction factor c
    # c = P * [x_scale, y_scale, z_scale, 1]^T projected appropriately
    # This is handled by the GPU pipeline internally based on P and view-space Z
    
    for fragment in slice.rasterize(camera):
        # Read occlusion factor from previous slice's occlusion buffer
        # Note: In the actual algorithm, the occlusion buffer stores the accumulated visibility
        # from the *previous* slice's update pass.
        v = read_texture(occlusion_buffer, fragment.texture_coords)
        
        # Sample volume
        color_sample, alpha_sample = sample_volume(fragment.world_position)
        
        # Ambient radiance (constant)
        L_a = vec3(1.0, 1.0, 1.0) # Simplified
        
        # Modulate ambient color by occlusion
        ambient_color = color_sample * v * L_a
        
        # Front-to-back compositing
        # C_out = C_in + C_ambient * alpha_sample * (1 - alpha_in)
        # alpha_out = alpha_in + alpha_sample * (1 - alpha_in)
        
        c_in, a_in = read_texture(eye_buffer, fragment.texture_coords)
        c_out = c_in + ambient_color * alpha_sample * (1.0 - a_in)
        a_out = a_in + alpha_sample * (1.0 - a_in)
        
        write_texture(eye_buffer, fragment.texture_coords, (c_out, a_out))

def render_slice_to_occlusion(slice, camera, occlusion_current, occlusion_next, aperture_angle, d, num_samples):
    """
    Updates the occlusion buffer by integrating visibility from the previous buffer.
    """
    # Precompute occlusion extent radius
    r = d * tan(aperture_angle)
    
    # Generate uniform grid sample positions on a disk of radius r
    samples = generate_disk_samples(num_samples, r)
    
    for fragment in slice.rasterize(camera):
        # Get correction factor c from vertex shader output
        c = fragment.correction_factor
        
        # Fragment NDC position
        x_y_d = fragment.ndc_position.xy
        
        total_weighted_visibility = 0.0
        total_weight = 0.0
        
        for p in samples:
            # Calculate NDC sample position
            # p_d = x_y_d + c * p (component-wise multiplication)
            p_d = x_y_d + c * p
            
            # Convert to texture coordinates
            p_t = 0.5 * p_d + vec2(0.5, 0.5)
            
            # Read previous occlusion factor
            v_prev = read_texture(occlusion_current, p_t)
            
            # Approximate transmittance using inter-slice distance d
            # T = exp(-sigma_t * d)
            # Since sigma_t is not explicitly passed to this shader in the notes,
            # we assume the transmittance is either pre-computed or approximated.
            # The notes say: "attenuating each sample by the transmittance computed using the distance...
            # approximating it with the inter-slice distance d".
            # However, sigma_t varies. The source [S2] implies T is computed.
            # For a self-contained implementation, we need sigma_t at the fragment.
            # Let's assume we sample sigma_t from the volume at the fragment's position.
            sigma_t = sample_sigma_t(fragment.world_position)
            t = exp(-sigma_t * d)
            
            total_weighted_visibility += v_prev * t
            total_weight += t
            
        # Average
        if total_weight > 0:
            v_new = total_weighted_visibility / total_weight
        else:
            v_new = 1.0
            
        write_texture(occlusion_next, fragment.texture_coords, v_new)
```

**Parameters:**
*   **Aperture Angle ($\theta$):** Controls the influence of features at different distances. Range $[0, \pi/2)$. Typical value: $80^\circ$ [S2].
*   **Inter-slice Distance ($d$):** Distance between view-aligned slices. Typical value: $0.001$ [S2].
*   **Sample Count:** Number of samples in the uniform grid for occlusion integration. Higher counts improve quality but reduce performance [S2].

**Pitfalls:**
*   **Perspective Distortion:** The cone base size varies with z-coordinate in screen space. Must be corrected using the projection matrix and view-space z in the vertex shader [S2].
*   **Inter-frame Artifacts:** Camera movement can cause artifacts. Mitigated by using a symmetric uniform grid for sample positions aligned with occlusion buffer axes [S2].
*   **MRT Undefined Behavior:** Reading from and writing to the same layered render target is undefined by OpenGL spec but works if reads/writes do not reference the same layer [S2].
*   **Light Source Limitation:** Bound to a single light source at the camera position [S1].

## Operating characteristics
*   **Time Complexity:** Depends on the number of slices $N_s$, resolution $W \times H$, and sample count $N_{samples}$. Complexity is roughly $O(N_s \cdot W \cdot H \cdot N_{samples})$ [S2].
*   **Memory Complexity:** Requires two additional floating-point buffers (Eye and Occlusion) of size $W \times H$. Low memory footprint compared to precomputation methods [S1].
*   **Performance:** Interactive frame rates achieved. Examples: 10.6 FPS for Stag Beetle (1046 slices, $80^\circ$ aperture) [S2]; 48.3 FPS for Carp CT (220 slices) [S2]; 59.0 FPS for Tree CT (1563 slices) [S2]. Performance is strongly dependent on grid resolution [S2].
*   **Strengths:** No precomputation required; supports interactive clipping planes and transfer function changes; accentuates concavities and depth discontinuities better than Lambertian shading [S2].
*   **Limitations:** Only supports single light source at camera; bound to slice-based rendering; artifacts may appear on regular structures [S2].
*   **MRT vs SRT:** Multiple Render Targets (MRT) increases performance by 44-99% compared to Single Render Target (SRT), especially for low sample counts. Bottleneck shifts to memory bandwidth for high sample counts [S2].

## Background
Directional Occlusion Shading was introduced by Mathias Schott, Vincent Pegoraro, Charles Hansen, Kévin Boulanger, and Kadi Bouatouch in 2009 at the Eurographics/ IEEE-VGTC Symposium on Visualization [S2]. It was published in the Journal of Computer Graphics Forum (Eurographics) [S2]. The method is derived from the radiative transport equation using a specialized backward-peaked cone phase function [S2]. It is used in interactive direct volume rendering to provide perceptual cues similar to ambient occlusion [S2].

## Sources
[S1] Interactive Volume Rendering with Volumetric Illumination, https://doi.org/10.2312/conf/eg2012/stars/053-074, fetched 2026-10-26
[S2] A Directional Occlusion Shading Model for Interactive Direct Volume Rendering, https://cgl.ethz.ch/teaching/scivis_common/Literature/Schott2009.pdf, fetched 2026-10-29
