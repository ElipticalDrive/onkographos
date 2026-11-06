# Single-pass Raycasting

Also called: Direct Volume Rendering, Direct Volume Rendering via Raycasting, Single Pass GPU Ray Casting

## Overview
Single-pass raycasting is a direct volume rendering technique that generates one ray per pixel, sampling a 3D texture along the ray and compositing the results in a single GPU rendering pass to produce a 2D image [S1, unofficial][S2]. It distinguishes itself from multi-pass approaches by avoiding temporary framebuffers for intermediate ray results, thereby addressing the low efficiency of multi-pass pipelines on modern graphics hardware [S2].

## How it works

### Inputs and Outputs
**Inputs:**
*   **Volume Data:** A 3D grid of scalar values. In the reference implementation, data is loaded from a `Uint16Array` where the first three elements are dimensions ($W, H, D$) and the remaining elements are voxel values normalized by dividing by 4095.0 [S1, unofficial]. The data is stored in a `THREE.Data3DTexture` with format `THREE.RedFormat`, type `THREE.FloatType`, and linear filtering [S1, unofficial].
*   **Camera:** Eye position and projection parameters.
*   **Lighting:** Light position (often fixed at camera position in basic implementations) and color [S1, unofficial].
*   **Parameters:** Uniforms controlling rendering mode, thresholds, step size, and transfer functions (see Parameters section).

**Outputs:**
*   **2D Image:** A color buffer where each pixel contains the accumulated color and opacity from the ray traversal [S1, unofficial].

### Data Structures
The implementation relies on the following structures and uniforms:

1.  **Volume Texture (`sampler3D volume_data`):**
    *   Dimensions: $W \times H \times D$.
    *   Coordinate space: Unit cube $[0,1]^3$.
2.  **Transfer Function Textures (`sampler2D`):**
    *   Lookup tables for color and opacity based on density and gradient magnitude [S3].
    *   Resolution controlled by `TransferSize` [S3].
3.  **Ray Entry/Exit Textures (FBOs):**
    *   Textures storing ray entry and exit positions for each fragment.
    *   Resolution controlled by `FBOSize` [S3].
4.  **Uniforms:**
    *   `volume_size`: `vec3` containing the dimensions of the volume [S1, unofficial].
    *   `render_mode`: `int` (0 for First Hit, 1 for MIP) [S1, unofficial].
    *   `u_renderthreshold`: `float` iso-value threshold for First Hit mode [S1, unofficial].
    *   `StepSize`: `float` spatial distance between samples [S3].
    *   `MaxSamples`: `int` limit on samples per ray [S3].
    *   `Minclip`: `float` threshold to discard low-density samples [S3].
    *   `Apply Noise`: `bool` toggle for stochastic ray shifting [S3].
    *   `GradDelta`: `vec3` offsets for gradient computation [S3].
    *   `Threshold (Shadows)`: `float` occlusion threshold for shadow rays [S3].
    *   `Quality (Shadows)`: `int` (0-100) controlling shadow ray sample count [S3].

### Numbered Steps

1.  **Initialization and Coordinate Transformation:**
    *   Transform the camera eye position into the volume's unit cube space. The transformed eye position is calculated as:
        $$ \text{transformed\_eye} = \frac{\text{cameraPosition}}{\text{volume\_size}} + (0.5, 0.5, 0.5) $$
        [S1, unofficial]
    *   For each fragment (pixel), calculate the ray direction in unit cube space. The fragment position is normalized to $[0,1]$. The direction is:
        $$ \vec{vray\_dir} = \frac{\text{position}}{\text{volume\_size}} + (0.5, 0.5, 0.5) - \text{transformed\_eye} $$
        [S1, unofficial]
    *   *Note:* In some implementations, ray entry and exit positions are pre-rendered to textures (FBOs) and looked up in the fragment shader to compute directions [S3].

2.  **Ray-Box Intersection:**
    *   Calculate the intersection of the view ray with the unit cube bounds $[0,1]^3$.
    *   If the ray does not intersect the cube ($t_{hit.x} > t_{hit.y}$), discard the fragment [S1, unofficial].
    *   Set the starting parameter $t = \max(t_{hit.x}, 0.0)$ [S1, unofficial].

3.  **Step Size Calculation:**
    *   Compute the step size $dt$ to ensure consistent sampling relative to the grid.
        $$ dt = \min \left( \frac{1}{W |v_x|}, \frac{1}{H |v_y|}, \frac{1}{D |v_z|} \right) $$
        where $\vec{v} = (v_x, v_y, v_z)$ is the ray direction and $W,H,D$ are volume dimensions [S1, unofficial].

4.  **Ray Marching Loop:**
    *   Initialize accumulation variables: `alpha = 0.0`, `color = (0,0,0)`.
    *   Initialize previous sample point `pointN0` and value `valueN0` at the start position [S1, unofficial].
    *   Iterate while $t < t_{hit.y}$ and sample count $< MaxSamples$:
        a.  **Stochastic Shifting:** If `Apply Noise` is true, perturb the ray position slightly to reduce banding artifacts [S3].
        b.  **Advance Ray:** Calculate next position `pointN1 = pointN0 + ray_dir * dt` [S1, unofficial].
        c.  **Sample Volume:** Read value `valueN1 = sample_volume(pointN1)` [S1, unofficial].
        d.  **Minclip Check:** If `valueN1 < Minclip`, continue to next sample (discard) [S3].
        e.  **Mode-Specific Processing:**
            *   **First Hit Mode (`render_mode == 0`):**
                *   If `valueN1 > u_renderthreshold`:
                    *   Interpolate to find the exact surface position $t_{surf}$ where the value equals the threshold:
                        $$ t_{surf} = t - dt \cdot \frac{valueN1 - u\_renderthreshold}{valueN1 - valueN0} $$
                        [S1, unofficial]
                    *   Calculate the surface normal using central differences (gradient) at the surface point. Gradient computation uses offsets defined by `GradDelta` [S3].
                    *   Compute lighting (diffuse/specular) using the normal and light direction [S1, unofficial].
                    *   Apply transfer function to map density to color [S1, unofficial].
                    *   Set final pixel color to the lit surface color and break the loop [S1, unofficial].
                *   Else: Update `valueN0 = valueN1`, `pointN0 = pointN1`, $t += dt$.
            *   **MIP Mode (`render_mode == 1`):**
                *   Track the maximum value encountered along the ray.
                *   Update `max_val = max(max_val, valueN1)` [S1, unofficial].
                *   Continue to next sample.
                *   After loop, apply transfer function to `max_val` to determine color [S1, unofficial].
        f.  **Shadow Rays (Optional):**
            *   For each sample, cast a shadow ray towards the light source [S3].
            *   Sample until density exceeds `Threshold (Shadows)` [S3].
            *   Number of samples controlled by `Quality (Shadows)` [S3].
            *   Modulate lighting based on occlusion [S3].

5.  **Transfer Function Application:**
    *   Map the final scalar value (surface value for First Hit, max value for MIP) to color and opacity using the transfer function lookup tables [S3].
    *   The transfer function allows arbitrary modulation of density and gradient magnitude lookups [S3].
    *   Default color mapping may use the Viridis colormap [S1, unofficial].

6.  **Compositing:**
    *   In First Hit mode, the result is the color of the first surface hit.
    *   In MIP mode, the result is the color corresponding to the maximum intensity.
    *   Standard front-to-back alpha compositing is not explicitly detailed for the final pixel in the provided code snippets for First Hit/MIP modes, as they are "single hit" or "max" projections rather than volumetric integration [S1, unofficial]. However, the general definition of DVR implies compositing [S1, unofficial]. The provided code for First Hit breaks on hit, and MIP takes the max, implying no alpha blending accumulation in the loop for these specific modes [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Vertex Shader / Setup
# 1. Transform camera position to unit cube space
transformed_eye = camera_position / volume_size + vec3(0.5, 0.5, 0.5)

# Fragment Shader
# 2. Calculate ray direction in unit cube space
# position is the normalized fragment coordinate (0.0 to 1.0)
ray_dir = (position / volume_size) + vec3(0.5, 0.5, 0.5) - transformed_eye

# 3. Intersect ray with unit cube [0,1]^3
t_hit = intersect_box(transformed_eye, ray_dir)
if t_hit.x > t_hit.y:
    discard

t_start = max(t_hit.x, 0.0)

# 4. Calculate step size
dt_vec = vec3(1.0) / (volume_size * abs(ray_dir))
dt = min(dt_vec.x, min(dt_vec.y, dt_vec.z))

# 5. Initialize variables
alpha = 0.0
color = vec3(0.0)
pointN0 = transformed_eye + t_start * ray_dir
valueN0 = sample_volume(pointN0)

max_val = -infinity

# 6. Ray Marching Loop
for t from t_start to t_hit.y with step dt:
    # Stochastic shifting if enabled
    if apply_noise:
        pointN1 = pointN0 + ray_dir * dt + stochastic_offset()
    else:
        pointN1 = pointN0 + ray_dir * dt

    valueN1 = sample_volume(pointN1)

    # Minclip check
    if valueN1 < minclip:
        valueN0 = valueN1
        pointN0 = pointN1
        continue

    if render_mode == 0: # First Hit
        if valueN1 > u_renderthreshold:
            # Interpolate surface position
            t_surf = t - dt * (valueN1 - u_renderthreshold) / (valueN1 - valueN0)
            surface_pos = pointN0 + ray_dir * (t - t_surf) # Approximate
            
            # Calculate gradient for normal
            grad = compute_gradient(surface_pos, grad_delta)
            normal = normalize(grad)
            
            # Lighting
            light_dir = normalize(light_position - surface_pos)
            diffuse = max(dot(normal, light_dir), 0.0)
            specular = pow(max(dot(reflect(-light_dir, normal), -ray_dir), 0.0), shininess)
            
            # Transfer function
            tf_color = apply_transfer_function(valueN1)
            color = tf_color * (ambient + diffuse * light_color + specular * light_color)
            break # Exit loop on first hit
        else:
            valueN0 = valueN1
            pointN0 = pointN1
    elif render_mode == 1: # MIP
        if valueN1 > max_val:
            max_val = valueN1
        valueN0 = valueN1
        pointN0 = pointN1

# 7. Final Output
if render_mode == 1:
    color = apply_transfer_function(max_val)

# Optional: Apply shadows during the loop (not shown in simplified pseudocode above)
# For each sample, cast shadow ray. If occluded, reduce lighting contribution.

gl_FragColor = vec4(color, 1.0)
```

### Parameters
*   **StepSize:** Controls spatial distance between samples. Higher values reduce quality but increase speed [S3].
*   **MaxSamples:** Limits the total number of samples per ray to prevent infinite loops or excessive computation [S3].
*   **u_renderthreshold:** Iso-value threshold for First Hit rendering. Only values above this trigger surface detection [S1, unofficial].
*   **Minclip:** Threshold below which samples are discarded (treated as empty) [S3].
*   **FBOSize:** Resolution of textures storing ray entry/exit positions [S3].
*   **TransferSize:** Resolution of transfer function lookup textures [S3].
*   **Apply Noise:** Boolean to enable stochastic ray shifting, reducing banding artifacts [S3].
*   **GradDelta:** Offsets for finite difference gradient calculation [S3].
*   **Threshold (Shadows):** Density value for shadow ray occlusion [S3].
*   **Quality (Shadows):** Number of samples for shadow ray traversal (0-100) [S3].

### Edge Cases and Pitfalls
*   **Data Loading:** Volume data is often stored in a `Uint16Array` with the first 3 elements as dimensions. The voxel values must be normalized by dividing by 4095.0 to fit in the $[0,1]$ range expected by the texture [S1, unofficial].
*   **Precision:** High precision (`highp float`) is required in shaders to avoid artifacts in ray marching and interpolation [S1, unofficial].
*   **Bandling Artifacts:** Regular sampling can cause banding. Stochastic ray shifting (`Apply Noise`) is used to mitigate this [S3].
*   **Memory Limits:** Large volumes may exceed GPU texture memory. Hierarchical grid representations or out-of-core techniques are needed for very large datasets [S2].

## Subtopics

### Object-Order Ray-Casting
*   **What it is:** A raycasting method that iterates over volume primitives (voxels) and projects them onto the screen, rather than iterating over pixels [Siblings].
*   **Difference:** Single-pass is image-order (pixel-driven), while Object-Order is primitive-driven. Object-order can be more efficient for sparse data but is harder to parallelize on GPUs compared to the regular access patterns of image-order raycasting [Siblings].
*   **When to choose:** Use when the volume is sparse or when primitive-level manipulation is required. Single-pass is preferred for dense, regular grids on GPUs [Siblings].
*   **Link:** [Object-Order Ray-Casting](6204-object-order-ray-casting.md)

### RaycastBrightnessControl
*   **What it is:** A method or parameterization technique related to controlling brightness and contrast in raycasting [Siblings].
*   **Difference:** Single-pass is a rendering pipeline strategy. RaycastBrightnessControl likely refers to a specific shading or transfer function adjustment technique applied within a raycasting framework [Siblings].
*   **When to choose:** Use when fine-tuning the visual appearance (brightness/contrast) of the raycast output is the primary goal.
*   **Link:** [RaycastBrightnessControl](6526-raycastbrightnesscontrol.md)

## Operating characteristics

*   **Time Complexity:** $O(P \cdot S)$, where $P$ is the number of pixels and $S$ is the number of samples per ray. Shadow rays add a factor of $S_{shadow}$ per sample.
*   **Memory Complexity:** $O(V)$ for the volume texture, where $V$ is the number of voxels. Additional memory for FBOs and transfer functions [S3].
*   **Performance:** Single-pass solutions are more efficient than multi-pass approaches on modern GPUs because they avoid the overhead of rendering to and reading from temporary framebuffers [S2].
*   **Strengths:** High flexibility (custom shaders), real-time potential on capable hardware, accurate lighting and transfer functions [S2, S3].
*   **Limitations:** Performance degrades with high resolution, large volumes, or complex lighting (shadow rays). Memory bound for very large volumes [S2].
*   **When to use:** Real-time visualization of moderate-sized volumes, interactive exploration, spectral DVR [S2, S3].
*   **When not to use:** Extremely large volumes that do not fit in GPU memory without hierarchical streaming, or when offline, high-quality volume rendering is required (where path tracing or other offline methods might be better) [S2].

## Background

*   **Origin:** The concept of single-pass GPU raycasting was developed to address the inefficiencies of multi-pass rendering pipelines on graphics hardware [S2].
*   **Key Contributors:** Stegmaier and colleagues extended the framework to spectral DVR using Kubelka–Munk theory [S2]. Hadwiger et al. used single-pass raycasting for implicit isosurfaces on rectilinear grids [S2]. Roettger et al. built a real-time raycaster with adaptive pre-integrated voxel classification [S2].
*   **History:** Early GPU-based solutions relied on multiple rendering passes [S2]. Single-pass techniques emerged to improve efficiency and flexibility [S2].
*   **Usage:** Used in VStudio as the primary direct volume rendering method [S3]. Extended for spectral DVR [S2].

## Sources

[S1] Masdaofdisasda/Vis1_Direct_Volume_Rendering_2022W, https://github.com/Masdaofdisasda/Vis1_Direct_Volume_Rendering_2022W, fetched 2026-10-26 (unofficial)
[S2] Volume Visualization: A Technical Overview with a Focus on ..., https://pmc.ncbi.nlm.nih.gov/articles/PMC3138940/, fetched 2026-10-27
[S3] VStudio - A Framework for Interactive Volume Visualization, https://www.cg.tuwien.ac.at/courses/Visualisierung/2008-2009/Beispiel1/Hausmair/index.html, fetched 2026-10-27
