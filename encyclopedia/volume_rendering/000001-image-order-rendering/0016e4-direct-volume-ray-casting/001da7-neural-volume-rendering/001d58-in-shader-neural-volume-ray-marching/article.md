# In-Shader Neural Volume Ray Marching

Also called: in-shader rendering

## Overview
In-Shader Neural Volume Ray Marching is a direct volume rendering technique where the neural network (MLP) is evaluated directly inside the CUDA kernel for each sample point during ray marching, rather than pre-decoding the volume into a 3D texture or using sample-streaming [S1, unofficial]. It supports multiple shading modes, including gradient shading and single-shade heuristics, allowing for real-time interactive visualization of neural radiance fields [S1, unofficial]. Unlike [Radiance-Predicting Neural Networks (RPNN)](7049-radiance-predicting-neural-networks-rpnn.md), which queries the network for accumulated multiple scattering contributions based on a global shading configuration, this method evaluates the network per sample point during the ray march [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A trained neural network (wrapped in `TcnnDeviceVolume` or `FvsrnDeviceVolume`), camera parameters (position, orientation, field of view), volume bounding box, transfer function, and rendering parameters (mode, shading scale, gradient step).
*   **Outputs:** A 2D framebuffer containing the composited color and alpha for each pixel.

**Data Structures:**
The neural network is encapsulated in a CUDA device class. Two primary implementations are supported [S1, unofficial]:
1.  `TcnnDeviceVolume<N, N_FEATURES_PER_LEVEL>`: Wraps a standard MLP. Template parameters define network width ($N \in \{16, 32, 64\}$) and features per level.
2.  `FvsrnDeviceVolume<N>`: Wraps an fV-SRN network. Used when `N_FEATURES_PER_LEVEL == -2`. Supports widths from $16 \times 1$ to $16 \times 8$.

Both classes provide `init()`, `launch()`/`launch2D()`, and `sample(p)` methods. The `sample(p)` method performs the neural network inference at position $\mathbf{p}$, returning raw density and color values [S1, unofficial].

**Parameters:**
*   `params.mode`: Shading mode. Supported values: `GRADIENT_SHADING`, `SINGLE_SHADE_HEURISTIC`.
*   `params.scivis_shading_scale`: Scalar float controlling the interpolation weight between raw sample color and shaded color in gradient mode.
*   `params.grad_step`: Step size for numerical gradient estimation via finite differences.

**Algorithm Steps:**

1.  **Ray Initialization:**
    For each pixel $(screen.x, screen.y)$, compute the ray origin $\mathbf{o}$ and direction $\mathbf{d}$ in world space.
    *   Transform camera position to world space: $\mathbf{o} = \text{xfmPoint}(W2C, \mathbf{c}_{pos})$.
    *   Construct direction from camera forward, horizontal, and vertical vectors, shifted by normalized screen coordinates:
        $$ \mathbf{d} = \text{normalize}(\mathbf{f} + (screen.x - 0.5)\mathbf{h} + (screen.y - 0.5)\mathbf{v}) $$
    where $\mathbf{f}, \mathbf{h}, \mathbf{v}$ are the camera's basis vectors [S1, unofficial].

2.  **Bounding Box Intersection:**
    Intersect the ray with the volume bounding box to determine the entry $t_0$ and exit $t_1$ distances.
    *   If no intersection is found (`!block_any(...)`), return background color and alpha.
    *   Set initial ray state: $\mathbf{r}_{org} = \mathbf{o}$, $\mathbf{r}_{dir} = \mathbf{d}$, $t = t_0$ [S1, unofficial].

3.  **Ray Marching Loop:**
    Iterate through the ray interval $[t_0, t_1]$ using an iterator. For each step interval $[t.x, t.y]$:
    *   **Sample Position:** Compute the sample point $\mathbf{p}$ using jittered linear interpolation:
        $$ \mathbf{p} = \mathbf{r}_{org} + \text{lerp}(\text{jitter}, t.x, t.y) \cdot \mathbf{r}_{dir} $$
    *   **Neural Query:** Evaluate the neural network at $\mathbf{p}$:
        $$ \text{value} = \text{neuralvolume.sample}(\mathbf{p}) $$
        This returns the raw density $\sigma$ and color $\mathbf{c}_{raw}$ [S1, unofficial].
    *   **Gradient Estimation (Optional):** If `params.mode == GRADIENT_SHADING`, estimate the gradient $\nabla \sigma$ numerically:
        $$ \nabla \sigma \approx -\text{sampleGradient}(\mathbf{p}, \text{value}, \text{grad\_step}, \text{sampler}) $$
        Otherwise, $\nabla \sigma = \mathbf{0}$ [S1, unofficial].
    *   **Transfer Function:** Apply the transfer function to map raw values to opacity $\alpha_s$ and color $\mathbf{c}_s$:
        $$ (\mathbf{c}_s, \alpha_s) = \text{sampleTransferFunction}(\text{tfn}, \text{value}) $$
        Apply opacity correction for the step size $\Delta t = t.y - t.x$.
    *   **Shading (Optional):**
        *   If `GRADIENT_SHADING`: Transform gradient to world space $\mathbf{N}_w = \text{xfmNormal}(C2W, \nabla \sigma)$ and ray direction to world space. Compute shaded color $\mathbf{c}_{shaded}$ using ambient and directional lighting (`shade_scivis_light`). Interpolate:
            $$ \mathbf{c}_s = \text{lerp}(\text{shadingScale}, \mathbf{c}_{raw}, \mathbf{c}_{shaded}) $$
        *   If `SINGLE_SHADE_HEURISTIC`: Track the point with the highest density $\alpha_{max}$ and its position $\mathbf{p}_{max}$.
    *   **Compositing:** Update accumulated color $\mathbf{C}$ and alpha $\alpha$ using front-to-back blending:
        $$ \tau = 1 - \alpha $$
        $$ \mathbf{C} \leftarrow \mathbf{C} + \tau \cdot \mathbf{c}_s \cdot \alpha_s $$
        $$ \alpha \leftarrow \alpha + \tau \cdot \alpha_s $$
    *   **Termination Check:** Continue if `alive` is true and $\alpha < \alpha_{near\_one}$. The iterator callback returns `alive && (alpha < nearly_one)` [S1, unofficial].

4.  **Single Shade Heuristic (Post-March):**
    If `params.mode == SINGLE_SHADE_HEURISTIC` and `highestAlpha > 0`:
    *   Calculate transmittance from the highest density point $\mathbf{p}_{max}$ to the light source using a secondary ray march towards the light direction.
    *   Interpolate the final color:
        $$ \mathbf{C}_{final} = \text{lerp}(\text{shadingScale}, \mathbf{C}, \mathbf{c}_{highest} \cdot \alpha_{max} \cdot \text{transmittance}) $$
    [S1, unofficial]

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def in_shader_neural_volume_ray_marching(camera, volume, tfn, params):
    """
    Renders a frame using in-shader neural volume ray marching.
    
    Args:
        camera: Camera object with position, forward/horizontal/vertical vectors.
        volume: NeuralVolume object with sample(p) method.
        tfn: TransferFunction object.
        params: Dict with 'mode', 'scivis_shading_scale', 'grad_step'.
        
    Returns:
        framebuffer: 2D array of (color, alpha) tuples.
    """
    width, height = camera.resolution
    framebuffer = zeros((height, width, 4)) # RGBA
    
    # Pre-compute camera basis vectors
    cam_pos = camera.position
    f = camera.forward
    h = camera.horizontal
    v = camera.vertical
    
    # World-to-View and View-to-World matrices for transformations
    # Note: xfmPoint and xfmNormal are assumed to be standard affine transforms
    W2C = camera.world_to_camera_matrix
    C2W = camera.camera_to_world_matrix
    
    for y in range(height):
        for x in range(width):
            # 1. Ray Initialization
            # Normalized screen position [-0.5, 0.5]
            nx = x - 0.5
            ny = y - 0.5
            
            # Ray origin in world space
            ray_org = xfm_point(W2C, cam_pos)
            
            # Ray direction in world space
            ray_dir = normalize(f + nx * h + ny * v)
            
            # 2. Bounding Box Intersection
            t0, t1 = intersect_volume_bbox(ray_org, ray_dir, volume.bbox)
            
            if t1 <= t0:
                # No intersection, return background
                framebuffer[y, x] = (0.0, 0.0, 0.0, 0.0)
                continue
            
            # 3. Ray Marching
            accumulated_color = vec3(0.0)
            accumulated_alpha = 0.0
            highest_alpha = 0.0
            highest_color = vec3(0.0)
            highest_pos = vec3(0.0)
            
            t = t0
            alive = True
            
            # Step size can be fixed or adaptive; here we assume a fixed step or iterator
            # The source uses an iterator that yields intervals [t.x, t.y]
            for t_start, t_end in ray_marching_iterator(t0, t1, volume):
                if not alive:
                    break
                
                # Sample position with jitter
                jitter = random_float() # [0, 1)
                p = lerp(ray_org + t_start * ray_dir, ray_org + t_end * ray_dir, jitter)
                
                # Neural Query
                # sample(p) returns (density, raw_color)
                density, raw_color = volume.sample(p)
                
                # Gradient Estimation
                gradient = vec3(0.0)
                if params.mode == GRADIENT_SHADING:
                    # Numerical gradient via finite differences
                    # sampleGradient returns the negative gradient estimate
                    gradient = -sample_gradient(p, density, params.grad_step, volume.sample)
                
                # Transfer Function
                sample_color, sample_alpha = tfn.apply(density)
                
                # Opacity correction for step size
                # Note: The source mentions applying opacity correction for step size
                # Assuming standard volume rendering alpha calculation: alpha = 1 - exp(-sigma * delta_t)
                # Here, sample_alpha is likely already processed, but we apply step correction
                step_size = t_end - t_start
                sample_alpha = 1.0 - exp(-density * step_size) # Approximation if raw density is sigma
                # If sample_alpha is already final alpha, skip this. 
                # Based on source: "Apply opacity correction for the step size"
                # We assume sample_alpha from TF is normalized [0,1] and needs scaling by step
                # However, standard VR uses sigma. Let's assume sample_alpha is the alpha contribution
                # for this step.
                
                # Shading
                shaded_color = raw_color
                if params.mode == GRADIENT_SHADING:
                    # Transform gradient to world space
                    N_w = xfm_normal(C2W, gradient)
                    # Transform ray direction to world space (already is, but for consistency)
                    d_w = ray_dir
                    
                    # Compute shaded color
                    shaded_color = shade_scivis_light(N_w, d_w, raw_color)
                    
                    # Interpolate
                    sample_color = lerp(params.scivis_shading_scale, raw_color, shaded_color)
                
                elif params.mode == SINGLE_SHADE_HEURISTIC:
                    if sample_alpha > highest_alpha:
                        highest_alpha = sample_alpha
                        highest_color = sample_color
                        highest_pos = p
                
                # Compositing
                transmittance = 1.0 - accumulated_alpha
                accumulated_color += transmittance * sample_color * sample_alpha
                accumulated_alpha += transmittance * sample_alpha
                
                # Termination check
                if accumulated_alpha >= 0.999: # nearly_one
                    alive = False
            
            # 4. Single Shade Heuristic Post-Processing
            if params.mode == SINGLE_SHADE_HEURISTIC and highest_alpha > 0.0:
                # Secondary ray march from highest_pos to light
                # Assuming light direction L is known or fixed
                L = vec3(0.0, 1.0, 0.0) # Example light direction
                transmittance_light = compute_transmittance(highest_pos, L, volume)
                
                # Interpolate final color
                # Source: color = lerp(shadingScale, color, highestColor * alpha * transmittance)
                # Note: highestColor is sample_color at highest point
                final_shaded_contribution = highest_color * highest_alpha * transmittance_light
                accumulated_color = lerp(params.scivis_shading_scale, accumulated_color, final_shaded_contribution)
            
            framebuffer[y, x] = (accumulated_color.x, accumulated_color.y, accumulated_color.z, accumulated_alpha)
    
    return framebuffer
```

## Subtopics

### [Radiance-Predicting Neural Networks (RPNN)](7049-radiance-predicting-neural-networks-rpnn.md)
A technique that predicts radiance for multiple scattering events in high-order scattering media. It differs by querying the network for accumulated contributions based on global shading configuration, enabling path-tracing-like results much faster than per-sample evaluation.

## Operating characteristics
*   **Time Complexity:** $O(N \cdot M)$, where $N$ is the number of pixels and $M$ is the number of samples per ray. The cost per sample is dominated by the neural network inference (MLP evaluation).
*   **Memory Complexity:** $O(N_{params})$, where $N_{params}$ is the number of parameters in the neural network. The volume data itself is not stored in a 3D texture, reducing VRAM usage compared to voxel-based methods.
*   **Performance:** Real-time performance is achievable on modern GPUs due to the parallel nature of CUDA kernels and the efficiency of small MLPs. Performance scales with network width and depth.
*   **Strengths:** No need for pre-computation or decoding of the volume into textures. Supports complex neural representations directly. Flexible shading modes.
*   **Limitations:** Performance is limited by the speed of neural network inference on the GPU. Requires specific network widths (16, 32, 64 for standard; multiples of 16 for fV-SRN). Numerical gradient estimation adds computational overhead.
*   **When to use:** Interactive visualization of trained neural radiance fields where pre-computation is not feasible or desired.
*   **When not to use:** When extremely high-quality path-traced multi-scattering is required (use RPNN or offline rendering). When network inference is too slow for the target frame rate.

## Background
In-Shader Neural Volume Ray Marching is implemented in the **instantvnr** library by VIDILabs [S1, unofficial]. It was developed to provide a flexible and efficient way to render neural volume representations directly on the GPU without the overhead of decoding large volumes into textures. The method leverages the `tiny-cuda-nn` library for efficient MLP evaluation on CUDA [S1, unofficial]. It supports both standard MLPs and fV-SRN (feature-vector Super-Resolution Network) architectures [S1, unofficial].

## Sources
[S1] VIDILabs/instantvnr, https://github.com/VIDILabs/instantvnr, fetched 2026-10-26 (unofficial)
