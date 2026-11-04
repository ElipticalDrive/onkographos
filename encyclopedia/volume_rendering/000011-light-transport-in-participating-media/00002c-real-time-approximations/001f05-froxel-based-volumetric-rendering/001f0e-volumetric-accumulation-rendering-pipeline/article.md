# Volumetric Accumulation Rendering Pipeline

Also called: Froxel-based Temporal Accumulation, TAA for Volumetric Fog

## Overview
The Volumetric Accumulation Rendering Pipeline is a real-time approximation method for light transport in participating media that uses a low-resolution froxel grid to compute in-scattered radiance and applies temporal anti-aliasing (TAA) to stabilize the result [S1, unofficial]. It solves the high computational cost and memory requirements of full volumetric ray-marching by decoupling the expensive scattering calculation from the final high-resolution rendering pass, relying on history reprojecting and exponential smoothing to reduce noise [S1, unofficial]. It differs from deterministic ray-marching compositing by accepting stochastic noise in exchange for performance, and from static froxel shading by introducing temporal coherence to handle camera movement and dynamic scenes [S1, unofficial].

## How it works
**Inputs:** Camera view/projection matrices, previous frame's history buffer, current frame's geometry depth/position data, volumetric effect range, froxel grid resolution.
**Outputs:** Final composited image with volumetric fog.

**Data Structures:**
1.  **Froxel Grid:** A 3D texture of size $W \times H \times D$.
    *   $W, H$: Screen-space width/height (e.g., 160x90). Aspect ratio $\approx 16:9$.
    *   $D$: Depth slices covering the volumetric effect range (e.g., 64 or 128).
    *   Layout: Linear memory or 3D texture format. Each cell stores computed in-scattered radiance and participating media density.
2.  **History Buffer:** Same dimensions as the Froxel Grid. Stores the accumulated radiance value $f_t(p)$ for each froxel $p$ at frame $t$.
3.  **Integrated 3D Texture:** Same dimensions as the Froxel Grid. Stores accumulated volumetric transmittance and radiance integrated from the camera outward.

**Parameters:**
*   **Froxel Resolution:** Varies by platform. Typical: $160 \times 90 \times 64$ or $160 \times 90 \times 128$ [S1, unofficial]. Unity HDRP default: $240 \times 135 \times 64$ [S1, unofficial]. RDR2: $160 \times 88 \times 64$ [S1, unofficial].
*   **Volumetric Effect Range:** Depth extent of the grid. Typical: 50–128 meters (Wronski), ~160 meters (RDR2) [S1, unofficial].
*   **Blending Factor ($\alpha$):** Exponential moving average weight, $\alpha \in [0, 1]$ [S1, unofficial].
*   **Jitter Offset:** Per-frame sub-froxel offset, typically from a Halton sequence [S1, unofficial].

**Steps:**

1.  **Grid Creation:** Create a low-resolution froxel grid aligned with the camera frustum for the current frame. The grid covers the defined volumetric effect range [S1, unofficial].
2.  **Compute In-Scattered Radiance:** In a GPU compute shader, compute the local in-scattered radiance and participating media density for each froxel in parallel [S1, unofficial].
3.  **Jitter Sampling:** Acquire the per-frame sub-froxel jitter offset. Apply this offset to the sampling point within the froxel. This offset is pre-calculated and passed as a shader parameter [S1, unofficial].
4.  **Reprojection:** Reproject the history buffer from the previous frame into the current frame's froxel grid.
    *   Use a reprojection vector derived from the camera's movement and rotation between frames [S1, unofficial].
    *   For each froxel $p$, calculate the reprojected coordinate $\pi(p)$ using the vector and the froxel's position [S1, unofficial].
    *   If $\pi(p)$ is outside the history buffer bounds, discard history and use the current sample only [S1, unofficial].
5.  **Temporal Blending:** Blend the current frame's sample $s_t(p)$ with the reprojected history $f_{t-1}(\pi(p))$ using an exponential moving average:
    $$ f_t(p) = \alpha s_t(p) + (1 - \alpha) f_{t-1}(\pi(p)) $$
    where $f_t(p)$ is the blended radiance for froxel $p$ at frame $t$ [S1, unofficial]. Store $f_t(p)$ in the history buffer for the next frame [S1, unofficial].
6.  **Low-Pass Filtering (Optional):** Apply a low-pass filter (e.g., Gaussian blur with radius 3) to the froxel grid to smooth sharp changes between nearby froxels. Note: This may remove sharp features [S1, unofficial].
7.  **Depth-Wise Integration:** Integrate volumetric radiance and transmittance through the froxel grid depth-wise from the camera outward.
    *   Perform this in a low-resolution 2D shader corresponding to the $x$ and $y$ dimensions of the grid [S1, unofficial].
    *   Accumulate transmitted radiance into an output 3D texture, where each slice contains the accumulated volumetric transmittance up to that depth [S1, unofficial].
8.  **Final Rendering:** Render participating media onto the scene.
    *   Transform world-space fragment positions into froxel space [S1, unofficial].
    *   Sample the integrated 3D texture at the transformed position to acquire the volumetric contribution [S1, unofficial].
    *   Apply filtering (e.g., quadrilinear filtering) to the sampled texture to prevent jagged/pixelated appearance [S1, unofficial].
    *   Blend the result with previously rendered geometry using pre-multiplied color/alpha blending [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Step 1-3: Compute Shader Pass
def compute_froxel_radiance(froxel_grid, history_buffer, camera_matrix, prev_camera_matrix, jitter_offset, alpha):
    # 1. Grid is implicitly created based on camera frustum and effect range
    
    # 2. Compute local in-scattered radiance and density for each froxel
    for froxel in froxel_grid:
        # Sample participating media density and compute in-scattered radiance
        # This is computationally expensive, done in parallel on GPU
        current_sample = compute_in_scattered_radiance(froxel.position + jitter_offset)
        
        # 4. Reprojection
        # Calculate reprojection vector from camera movement
        reprojection_vector = get_camera_movement_vector(camera_matrix, prev_camera_matrix)
        
        # Reproject previous history to current froxel position
        prev_pos = froxel.position + reprojection_vector
        if is_within_bounds(prev_pos, history_buffer):
            history_value = history_buffer.sample(prev_pos)
        else:
            history_value = 0.0 # Reset history if out of bounds
            
        # 5. Temporal Blending (Exponential Moving Average)
        blended_radiance = alpha * current_sample + (1 - alpha) * history_value
        
        # Store result in history buffer for next frame
        history_buffer.set(froxel.position, blended_radiance)
        
        # 6. Optional Low-Pass Filter (Separate Pass)
        # froxel_grid[froxel.position] = gaussian_blur(froxel_grid, radius=3)

# Step 7: Integration Pass
def integrate_volumetric_data(froxel_grid):
    # Output 3D texture same size as froxel grid
    integrated_texture = create_3d_texture(froxel_grid.dimensions)
    
    # Integrate depth-wise from camera outward
    for x in range(froxel_grid.width):
        for y in range(froxel_grid.height):
            accumulated_transmittance = 1.0
            accumulated_radiance = 0.0
            for z in range(froxel_grid.depth):
                froxel = froxel_grid[x, y, z]
                # Accumulate transmitted radiance
                # Note: Exact integration formula depends on density/transmittance model
                transmittance = compute_transmittance(froxel.density)
                accumulated_radiance += froxel.radiance * accumulated_transmittance
                accumulated_transmittance *= transmittance
                integrated_texture[x, y, z] = (accumulated_radiance, accumulated_transmittance)
    return integrated_texture

# Step 8: Final Rendering Pass
def render_volumetric_fog(scene_geometry, integrated_texture, froxel_grid):
    for fragment in scene_geometry:
        # Transform world-space position to froxel space
        froxel_pos = transform_to_froxel_space(fragment.world_position)
        
        # Sample integrated 3D texture
        # Use quadrilinear filtering to smooth result
        volumetric_contribution = integrated_texture.quadrilinear_sample(froxel_pos)
        
        # Blend with previously rendered geometry
        final_color = blend_pre_multiplied(fragment.color, volumetric_contribution)
        return final_color
```

## Subtopics

### [Accumulation Buffer Composite](2368-accumulation-buffer-composite.md)
The final post-processing step that averages accumulated color samples and blends with the background, removing spot noise artifacts inherent in stochastic sampling.

### [Empirical Volume Shading with Indirect Light Approximation](4092-empirical-volume-shading-with-indirect-light-approximation.md)
A shading model that approximates global illumination by blurring direct light in a local neighborhood cone and applying indirect attenuation, decoupling indirect lighting from high-resolution volume data.

## Operating characteristics
**Time Complexity:** The core computation scales with the number of froxels ($W \times H \times D$), which is fixed and independent of screen resolution, making performance predictable [S1, unofficial]. The integration pass scales with $W \times H \times D$. The final rendering pass scales with screen resolution but is lightweight (texture sampling and blending) [S1, unofficial].
**Memory Complexity:** Scales cubically with froxel resolution. For $160 \times 90 \times 64$, this is manageable in modern GPUs, but higher resolutions (e.g., $160 \times 90 \times 128$) double the memory cost [S1, unofficial].
**Performance:** Real-time frame rates are achievable on modern hardware. RDR2 uses $160 \times 88 \times 64$ for ~160m range [S1, unofficial]. Unity HDRP uses $240 \times 135 \times 64$ [S1, unofficial].
**Strengths:** Performance is decoupled from screen resolution; temporal accumulation reduces noise without requiring high sample counts per frame; reprojection handles camera movement efficiently.
**Limitations:** Ghosting artifacts if reprojection fails or history validation is insufficient; blurring from low-pass filters or temporal smoothing can reduce sharpness; fixed froxel resolution may lead to aliasing at extreme distances or angles if not managed [S1, unofficial].
**When to Use:** Real-time applications requiring volumetric fog/lighting with dynamic cameras and limited computational resources.
**When Not to Use:** Offline rendering requiring high fidelity without temporal artifacts; scenes with very large, static volumetric effects where pre-computation might be better.

## Background
The method originates from real-time froxel-based volumetric rendering techniques developed for modern video games. Key implementations include Wronski's work (resolutions 160x90x64/128, range 50-128m) [S1, unofficial], Red Dead Redemption 2 (resolution 160x88x64, range ~160m) [S1, unofficial], and Unity's High Definition Render Pipeline (HDRP) (resolution 240x135x64) [S1, unofficial]. The approach leverages temporal anti-aliasing (TAA) principles, specifically exponential moving average blending and reprojection, to stabilize stochastic volumetric samples [S1, unofficial].

## Sources
[S1] Temporal Anti-Aliasing for Real-Time Froxel-Based Volumetric Rendering - Advances in volumetric temporal anti-aliasing for real-time rendering of participating media, https://openalex.org/W7218067689, fetched 2026-11-01 (unofficial)
