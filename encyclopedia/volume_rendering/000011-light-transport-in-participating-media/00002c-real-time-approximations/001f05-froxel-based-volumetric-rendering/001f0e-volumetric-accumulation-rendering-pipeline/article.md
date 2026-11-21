# Volumetric Accumulation Rendering Pipeline

Also called: Froxel-based Temporal Accumulation, Real-time Froxel Volumetric Rendering

## Overview
The Volumetric Accumulation Rendering Pipeline is a real-time approximation for rendering participating media (such as fog, smoke, or volumetric clouds) within a camera frustum using a low-resolution 3D grid of "froxels" [S1, unofficial]. It solves the computational intractability of full volumetric light transport by decoupling the expensive per-froxel radiance calculation from the final depth-wise integration, utilizing temporal accumulation to reuse historical data across frames [S1, unofficial]. It differs from [Froxel TAA Reprojection](1920-froxel-taa-reprojection.md) by defining the complete rendering pipeline including the integration and shading stages, and from [Froxel Depth Distribution](1922-froxel-depth-distribution.md) by focusing on temporal stability and accumulation rather than depth-aware sampling distributions [S1, unofficial].

## How it works
**Inputs:** Camera view/projection matrices, world-to-camera transformation, froxel grid resolution, volumetric effect range, per-frame jitter seed, and previous frame's history buffer [S1, unofficial].
**Outputs:** A 2D screen-space image with volumetric lighting blended over geometry [S1, unofficial].

**Data Structures:**
1.  **Froxel Grid:** A 3D texture or buffer of size $W \times H \times D$. Fields typically store in-scattered radiance and participating media density [S1, unofficial].
2.  **History Buffer:** Same dimensions as the froxel grid, storing historical radiance values $f_{t-1}$ [S1, unofficial].
3.  **Integrated 3D Texture:** Same size as the froxel grid. Each slice $z$ contains the accumulated volumetric transmittance up to that depth [S1, unofficial].

**Parameters:**
*   **Froxel Resolution:** Varies by platform. Common values: $160 \times 90 \times 64$ or $160 \times 90 \times 128$ (Wronski), $160 \times 88 \times 64$ (Red Dead Redemption 2), $240 \times 135 \times 64$ (Unity HDRP) [S1, unofficial]. Aspect ratio is typically $\sim 16:9$ [S1, unofficial].
*   **Volumetric Effect Range:** Depth extent of the grid. Typical values: 50–128 meters (Wronski), $\sim 160$ meters (Red Dead Redemption 2) [S1, unofficial].
*   **Blending Factor ($\alpha$):** Exponential moving average weight, $\alpha \in [0, 1]$ [S1, unofficial].
*   **Jitter Offset:** Per-frame sub-froxel offset, often derived from a Halton sequence [S1, unofficial].

**Steps:**

1.  **Grid Creation:** Create a low-resolution froxel grid in the camera frustum for the current frame [S1, unofficial].
2.  **Per-Froxel Radiance Calculation:** In a compute shader, compute local in-scattered radiance and participating media density for each froxel in parallel [S1, unofficial].
3.  **Jittering:** Acquire the per-frame sub-froxel jitter offset and apply it to the sampling point [S1, unofficial].
4.  **Reprojection:** Reproject the history buffer into the current frame using a reprojection vector derived from camera movement [S1, unofficial]. If reprojection fails (e.g., vector points outside the buffer), discard history and use the current sample [S1, unofficial].
5.  **Temporal Accumulation:** Blend the current frame sample $s_t(p)$ with the reprojected history $f_{t-1}(\pi(p))$ using an exponential moving average:
    $$ f_t(p) = \alpha s_t(p) + (1 - \alpha) f_{t-1}(\pi(p)) $$
    where $p$ is the froxel position and $\pi(p)$ is the reprojected position [S1, unofficial].
6.  **Smoothing (Optional):** Apply a low-pass filter (e.g., Gaussian blur with radius 3) to smooth sharp changes between nearby froxels [S1, unofficial].
7.  **Depth-wise Integration:** Integrate volumetric radiance and transmittance through the froxel grid depth-wise from the camera outward. This is performed in a low-resolution 2D shader (corresponding to the $x$ and $y$ dimensions) by stepping through the $z$ dimension and accumulating transmitted radiance into an output 3D texture [S1, unofficial].
8.  **Shading:** Render participating media by transforming world-space fragment positions into froxel space and sampling the integrated 3D texture [S1, unofficial].
9.  **Filtering:** Filter the participating media texture (e.g., quadrilinear filtering) to prevent a jagged/pixelated appearance [S1, unofficial].
10. **Blending:** Blend the result with previously rendered geometry using pre-multiplied color/alpha blending [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Step 1-3: Compute Shader Pass
def compute_froxel_radiance(froxel_grid, jitter_offset):
    for each froxel p in froxel_grid:
        # Apply jitter to sampling point
        sample_point = p.position + jitter_offset
        # Compute local in-scattered radiance and density
        radiance = compute_in_scattered_radiance(sample_point)
        density = compute_media_density(sample_point)
        froxel_grid[p].radiance = radiance
        froxel_grid[p].density = density

# Step 4-5: Temporal Accumulation Pass
def accumulate_history(history_buffer, current_froxel_grid, reprojection_vector, alpha):
    for each froxel p in history_buffer:
        # Reproject previous position to current frame
        prev_p = p.position - reprojection_vector
        if is_valid(prev_p):
            history_sample = history_buffer[prev_p].radiance
        else:
            history_sample = 0.0
        
        # Exponential Moving Average
        current_sample = current_froxel_grid[p].radiance
        history_buffer[p].radiance = alpha * current_sample + (1 - alpha) * history_sample

# Step 6: Optional Blur Pass
def smooth_froxels(froxel_grid, radius=3):
    froxel_grid = gaussian_blur(froxel_grid, radius)

# Step 7: Integration Pass (2D Shader stepping through Z)
def integrate_volumetrics(froxel_grid):
    output_texture = create_3d_texture(size=froxel_grid.size)
    for each pixel (x, y) in screen:
        accumulated_transmittance = 1.0
        accumulated_radiance = 0.0
        for z from 0 to froxel_grid.depth:
            froxel = froxel_grid[x, y, z]
            # Accumulate transmittance and radiance
            # Note: Exact integration formula depends on specific medium model
            # Standard volume rendering equation approximation
            transmittance = exp(-froxel.density * step_size)
            accumulated_radiance += accumulated_transmittance * froxel.radiance * (1 - transmittance)
            accumulated_transmittance *= transmittance
            output_texture[x, y, z].radiance = accumulated_radiance
            output_texture[x, y, z].transmittance = accumulated_transmittance
    return output_texture

# Step 8-10: Final Render Pass
def render_participating_media(integrated_texture, geometry_buffer):
    for each fragment f in geometry:
        # Transform world position to froxel space
        froxel_pos = world_to_froxel(f.world_position)
        # Sample integrated texture
        volumetric_color = quadrilinear_sample(integrated_texture, froxel_pos)
        # Blend with geometry
        final_color = pre_multiply_blend(volumetric_color, geometry_buffer[f].color)
        return final_color
```

## Subtopics

### [Accumulation Buffer Composite](2368-accumulation-buffer-composite.md)
The Accumulation Buffer Composite is a post-processing step that averages accumulated color samples from a stochastic volume renderer and blends the result with the background using a reveal factor.

### [Empirical Volume Shading with Indirect Light Approximation](4092-empirical-volume-shading-with-indirect-light-approximation.md)
Empirical Volume Shading with Indirect Light Approximation is a real-time shading model for participating media that approximates global illumination by blurring direct light within a local neighborhood cone and applying a separate indirect attenuation term. It solves the computational intractability of full light transport in participating media by decoupling indirect lighting from high-resolution volume data, allowing for interactive frame rates without pre-computed global illumination caches. It differs from traditional volume rendering by adding indirect attenuation and local light diffusion, and from full global illumination by restricting scattering to a local blur rather than computing transport from all directions.

### SNeRG Deferred Shading Model
SNeRG decomposes the radiance field into a 3D field of densities, diffuse RGB colors, and feature vectors to reduce MLP evaluations to one per ray. It volumetrically accumulates diffuse colors and features, then computes view-dependent color using a small MLP [S2].
**Steps:**
1.  Volumetrically accumulate diffuse colors $C_d$ and features $F$ along the ray using weights $w_i$: $C_d = \sum_i w_i c_{d,i}$ and $F = \sum_i w_i f_i$ [S2].
2.  Compute final ray color $C$ by adding the accumulated diffuse color $C_d$ to the view-dependent color output from a small MLP $h$, which takes $C_d$, $F$, and viewing direction $d$ as input [S2].
**Formula:**
$$ C = C_d + h(C_d, F, d) $$
where $h$ is a small MLP [S2].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of froxels and the depth resolution of the integration pass [S1, unofficial]. The cost is independent of screen resolution, making it scalable to high-res displays [S1, unofficial].
*   **Memory Complexity:** Cubic with respect to the froxel grid dimensions ($W \times H \times D$) [S1, unofficial].
*   **Performance:** Performance is determined by the froxel grid resolution and effect range. Lower-end devices may halve depth precision to save memory [S1, unofficial].
*   **Strengths:** Real-time performance on consumer hardware; decouples expensive radiance calculation from final integration; temporal accumulation reduces noise [S1, unofficial].
*   **Limitations:** Ghosting artifacts if reprojection fails or objects move rapidly; loss of sharp features if low-pass filtering is applied; fixed grid resolution may not capture fine details at far distances without large effect ranges [S1, unofficial].
*   **When to use:** Real-time games and applications requiring volumetric fog, smoke, or clouds with dynamic lighting [S1, unofficial].
*   **When not to use:** Offline rendering requiring high-fidelity global illumination; scenes with extremely large unbounded volumes where fixed frustum grids are insufficient [S1, unofficial].

## Background
The method originates from real-time volumetric rendering techniques for participating media, specifically those utilizing froxel-based representations [S1, unofficial]. It was popularized by implementations in commercial engines and games, including Wronski's work, Red Dead Redemption 2, and Unity's High Definition Render Pipeline (HDRP) [S1, unofficial]. The approach builds on [Froxel-based Volumetric Rendering](1919-froxel-based-volumetric-rendering.md) by adding temporal accumulation to stabilize the noisy samples inherent in real-time volumetric calculations [S1, unofficial].

## Sources
[S1] Temporal Anti-Aliasing for Real-Time Froxel-Based Volumetric Rendering - Advances in volumetric temporal anti-aliasing for real-time ren dering of participating media, https://openalex.org/W7218067689, fetched 2026-11-01 (unofficial)
[S2] MERF: Memory-Efficient Radiance Fields for Real-time View Synthesis in Unbounded Scenes, https://doi.org/10.1145/3592426, fetched 2026-10-26
