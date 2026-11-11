# In-Shader Neural Volume Ray Marching

Also called: in-shader rendering

## Overview
In-Shader Neural Volume Ray Marching is a direct volume rendering technique where the neural network (MLP) is evaluated directly inside the CUDA kernel for each sample point during ray traversal, rather than pre-decoding the volume into a 3D texture or using sample-streaming [S1, unofficial]. It replaces traditional texture lookups with neural network inference at every step of the ray march [S1, unofficial]. This method supports multiple shading modes, including gradient shading and single-shade heuristics, allowing for complex lighting effects directly within the rendering loop [S1, unofficial]. It is distinct from [Radiance-Predicting Neural Networks (RPNN)](7049-radiance-predicting-neural-networks-rpnn.md), which queries the network for accumulated multiple scattering contributions based on global shading configurations rather than per-sample evaluation [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Neural network weights (MLP), camera parameters (origin, direction, field of view), volume bounding box, transfer function parameters, and shading mode configuration.
*   **Outputs:** Final composited pixel color and opacity.

**Data Structures:**
The implementation relies on specific CUDA device classes to wrap the neural network.
*   `TcnnDeviceVolume<N, N_FEATURES_PER_LEVEL>`: A CUDA device class wrapping the neural network. Template parameters define network width ($N \in \{16, 32, 64\}$) and features per level. It provides `init()`, `launch2D()`, and `sample(p)` methods [S1, unofficial].
*   `FvsrnDeviceVolume<N>`: A CUDA device class for fV-SRN networks, used when `N_FEATURES_PER_LEVEL == -2`. Supports widths from $16 \times 1$ to $16 \times 8$. It provides `init()`, `launch()`, and `sample(p)` methods [S1, unofficial].

**Parameters:**
*   `params.mode`: Shading mode. Supported values include `GRADIENT_SHADING` and `SINGLE_SHADE_HEURISTIC` [S1, unofficial].
*   `params.scivis_shading_scale`: Scalar float controlling the interpolation weight between raw sample color and shaded color in gradient shading mode [S1, unofficial].
*   `params.grad_step`: Step size used for numerical gradient estimation via finite differences [S1, unofficial].
*   `N`: Network width. Must be 16, 32, or 64 for standard networks, or a multiple of 16 (16*1 to 16*8) for fV-SRN [S1, unofficial].
*   `N_FEATURES_PER_LEVEL`: Encoding method identifier. Must not be -1 [S1, unofficial].

**Algorithm Steps:**

1.  **Ray Generation:**
    Initialize the ray origin $\mathbf{o}$ and direction $\mathbf{d}$ in world space. Transform the camera origin from camera space to world space using the world-to-view transformation matrix $\mathbf{W}_{to}$:
    $$ \mathbf{o} = \mathbf{W}_{to} \cdot \text{camera.position} $$
    Construct the ray direction from the camera's forward, horizontal, and vertical vectors, shifted by the normalized screen position $(s_x - 0.5)$ and $(s_y - 0.5)$, then normalized [S1, unofficial]:
    $$ \mathbf{d} = \text{normalize}(\mathbf{f} \cdot \text{fov} + \mathbf{h} \cdot (s_x - 0.5) + \mathbf{v} \cdot (s_y - 0.5)) $$

2.  **Bounding Box Intersection:**
    Intersect the ray with the volume bounding box to determine the entry $t_{start}$ and exit $t_{end}$ points. If no intersection is found, return the background color and alpha [S1, unofficial].

3.  **Ray Marching Loop:**
    Iterate through the ray segment $[t_{start}, t_{end}]$ using an iterator. For each step interval $[t_i, t_{i+1}]$:
    *   **Sample Position:** Compute the sample position $\mathbf{p}$ using jittered linear interpolation:
        $$ \mathbf{p} = \mathbf{o} + \text{lerp}(\text{jitter}, t_i, t_{i+1}) \cdot \mathbf{d} $$
    *   **Neural Query:** Sample the neural volume at $\mathbf{p}$ using the network's `sample(p)` method to retrieve the raw value $v$ (typically density $\sigma$) [S1, unofficial].
    *   **Gradient Estimation (Optional):** If `params.mode == GRADIENT_SHADING`, estimate the gradient $\nabla v$ numerically using finite differences with step size `grad_step`:
        $$ \nabla v \approx \frac{v(\mathbf{p} + \delta) - v(\mathbf{p} - \delta)}{2\delta} $$
        Negate the gradient for shading purposes [S1, unofficial]. Otherwise, set gradient to zero.
    *   **Transfer Function:** Apply the transfer function to $v$ to obtain sample color $\mathbf{c}_i$ and sample opacity $\alpha_i$. Apply opacity correction for the step size $\Delta t = t_{i+1} - t_i$:
        $$ \alpha_i = 1 - \exp(-v \cdot \Delta t) $$
        (Note: The specific transfer function implementation may vary, but typically maps raw density to opacity and color) [S1, unofficial].
    *   **Shading (Optional):**
        *   If `GRADIENT_SHADING`: Transform the gradient to world space $\mathbf{N}_w$ and the ray direction to world space. Compute shaded color using a lighting model (e.g., Phong with ambient and directional lights). Interpolate between raw sample color and shaded color:
            $$ \mathbf{c}_{final\_sample} = \text{lerp}(\text{shadingScale}, \mathbf{c}_i, \mathbf{c}_{shaded}) $$
        *   If `SINGLE_SHADE_HEURISTIC`: Track the sample with the highest density $\alpha_{max}$ and its position $\mathbf{p}_{max}$ [S1, unofficial].
    *   **Compositing:** Composite the sample into the accumulated color $C_{acc}$ and opacity $\alpha_{acc}$ using front-to-back blending:
        $$ C_{acc}^{(new)} = C_{acc}^{(old)} + (1 - \alpha_{acc}^{(old)}) \cdot \alpha_i \cdot \mathbf{c}_{final\_sample} $$
        $$ \alpha_{acc}^{(new)} = \alpha_{acc}^{(old)} + (1 - \alpha_{acc}^{(old)}) \cdot \alpha_i $$
        where transmittance $tr = 1 - \alpha_{acc}^{(old)}$ [S1, unofficial].
    *   **Early Termination:** Continue marching while the ray is alive and $\alpha_{acc} < \text{nearly\_one}$ (e.g., 0.999). If $\alpha_{acc}$ exceeds this threshold, break the loop [S1, unofficial].

4.  **Single Shade Heuristic (Optional):**
    If `params.mode == SINGLE_SHADE_HEURISTIC` and `highestAlpha > 0`:
    *   Calculate the transmittance from the highest density point $\mathbf{p}_{max}$ to the light source using a secondary ray march towards the light direction [S1, unofficial].
    *   Interpolate the final color:
        $$ C_{final} = \text{lerp}(\text{shadingScale}, C_{acc}, \mathbf{c}_{max} \cdot \alpha_{max} \cdot T_{light}) $$
        where $T_{light}$ is the transmittance to the light [S1, unofficial].

5.  **Final Output:**
    Composite the accumulated color over the background color $C_{bg}$:
    $$ C_{pixel} = C_{acc} + (1 - \alpha_{acc}) \cdot C_{bg} $$
    Return $C_{pixel}$ and $\alpha_{acc}$ [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def in_shader_neural_volume_ray_marching(camera, volume, transfer_function, params):
    """
    Renders a frame using in-shader neural volume ray marching.
    
    Args:
        camera: Camera object with position, forward, horizontal, vertical vectors, and fov.
        volume: NeuralVolume object with sample(p) method.
        transfer_function: Function mapping raw value to (color, alpha).
        params: Configuration dict with 'mode', 'scivis_shading_scale', 'grad_step'.
        
    Returns:
        framebuffer: Array of pixel colors and alphas.
    """
    width, height = camera.resolution
    framebuffer = zeros((height, width, 4)) # RGBA
    
    # Pre-calculate camera vectors
    cam_pos_world = transform_point(camera.world_to_view_matrix, camera.position)
    cam_forward = normalize(camera.forward)
    cam_right = normalize(camera.horizontal)
    cam_up = normalize(camera.vertical)
    
    for y in range(height):
        for x in range(width):
            # 1. Ray Generation
            # Normalized screen coordinates centered at 0
            sx = (x + 0.5) / width - 0.5
            sy = (y + 0.5) / height - 0.5
            
            # Ray direction
            ray_dir = normalize(cam_forward * camera.fov + cam_right * sx + cam_up * sy)
            ray_org = cam_pos_world
            
            # 2. Bounding Box Intersection
            t_start, t_end = intersect_volume_bbox(ray_org, ray_dir, volume.bbox)
            if t_start > t_end:
                framebuffer[y, x] = camera.background_color
                continue
                
            # 3. Ray Marching
            acc_color = vec3(0.0)
            acc_alpha = 0.0
            highest_density = 0.0
            highest_pos = vec3(0.0)
            highest_color = vec3(0.0)
            
            # Discretize ray into steps
            num_samples = int((t_end - t_start) / volume.step_size)
            for i in range(num_samples):
                t = t_start + i * volume.step_size
                t_next = t + volume.step_size
                
                # Jittered sample position
                jitter = random_float() # [0, 1)
                p = lerp(ray_org + ray_dir * t, ray_org + ray_dir * t_next, jitter)
                
                # Neural Query
                raw_value = volume.sample(p)
                
                # Gradient Estimation (if needed)
                gradient = vec3(0.0)
                if params.mode == 'GRADIENT_SHADING':
                    # Numerical gradient using finite differences
                    eps = params.grad_step
                    grad_x = (volume.sample(p + vec3(eps, 0, 0)) - volume.sample(p - vec3(eps, 0, 0))) / (2 * eps)
                    grad_y = (volume.sample(p + vec3(0, eps, 0)) - volume.sample(p - vec3(0, eps, 0))) / (2 * eps)
                    grad_z = (volume.sample(p + vec3(0, 0, eps)) - volume.sample(p - vec3(0, 0, eps))) / (2 * eps)
                    gradient = vec3(grad_x, grad_y, grad_z)
                    gradient = -gradient # Negate for shading normal
                
                # Transfer Function
                sample_color, sample_alpha = transfer_function(raw_value)
                
                # Opacity correction for step size
                # Note: raw_value is often density sigma. Alpha = 1 - exp(-sigma * dt)
                dt = volume.step_size
                sample_alpha = 1.0 - exp(-raw_value * dt)
                
                # Shading
                final_sample_color = sample_color
                if params.mode == 'GRADIENT_SHADING':
                    # Transform gradient to world space
                    N_world = transform_normal(volume.object_to_world_matrix, gradient)
                    # Compute shaded color (e.g., Phong)
                    shaded_color = compute_phong_shading(N_world, ray_dir, volume.light_positions, volume.light_colors)
                    # Interpolate
                    final_sample_color = lerp(params.scivis_shading_scale, sample_color, shaded_color)
                elif params.mode == 'SINGLE_SHADE_HEURISTIC':
                    if raw_value > highest_density:
                        highest_density = raw_value
                        highest_pos = p
                        highest_color = sample_color
                
                # Compositing
                transmittance = 1.0 - acc_alpha
                acc_color += transmittance * sample_alpha * final_sample_color
                acc_alpha += transmittance * sample_alpha
                
                # Early termination
                if acc_alpha > 0.999:
                    break
            
            # Single Shade Heuristic Post-processing
            if params.mode == 'SINGLE_SHADE_HEURISTIC' and highest_density > 0.0:
                # Secondary ray march to light from highest_pos
                light_transmittance = compute_transmittance_to_light(highest_pos, volume.light_directions, volume)
                # Interpolate final color
                shaded_contribution = highest_color * highest_density * light_transmittance
                acc_color = lerp(params.scivis_shading_scale, acc_color, shaded_contribution)
            
            # Final Background Composite
            background_color = camera.background_color
            final_color = acc_color + (1.0 - acc_alpha) * background_color
            
            framebuffer[y, x] = vec4(final_color, acc_alpha)
            
    return framebuffer
```

## Subtopics

### [Radiance-Predicting Neural Networks (RPNN)](7049-radiance-predicting-neural-networks-rpnn.md)
Radiance-Predicting Neural Networks (RPNN) is a neural volume rendering technique that uses a multilayer perceptron (MLP) to predict radiance for multiple scattering events in high-order scattering media, such as atmospheric clouds. Unlike its sibling [In-Shader Neural Volume Ray Marching](1495-in-shader-neural-volume-ray-marching.md), which evaluates the network per sample point during ray marching, RPNN queries the network for the accumulated multiple scattering contribution based on a global shading configuration, enabling results nearly identical to path tracing but approximately a thousand times faster.

## Operating characteristics
*   **Time Complexity:** $O(N \cdot M)$, where $N$ is the number of pixels and $M$ is the number of samples per ray. The cost of the neural network inference per sample is significant and dominates the rendering time.
*   **Memory Complexity:** $O(W)$, where $W$ is the size of the neural network weights. The volume data itself is not stored in 3D textures, reducing memory footprint compared to traditional voxel-based methods.
*   **Performance:** Performance is heavily dependent on the neural network architecture and the hardware's ability to perform matrix multiplications (e.g., Tensor Cores in NVIDIA GPUs). The use of CUDA kernels allows for parallel execution across pixels and samples.
*   **Strengths:** High visual fidelity due to the expressive power of neural networks; no need for large 3D textures; supports complex lighting and shading directly in the shader.
*   **Limitations:** High computational cost per sample; requires GPU acceleration; sensitive to network architecture choices (width, features per level); potential for artifacts if the network is not well-trained or if sampling is insufficient.
*   **When to use:** Suitable for high-quality offline rendering or real-time applications with powerful GPUs where visual quality is paramount and traditional texture-based methods are insufficient.
*   **When not to use:** Not suitable for CPU-only rendering or low-end GPUs due to the high computational cost of neural network inference.

## Background
In-Shader Neural Volume Ray Marching is a technique developed as part of the instantvnr project by VIDILabs [S1, unofficial]. It leverages the capabilities of modern GPUs and neural network libraries (such as tiny-cuda-nn) to perform volume rendering directly within the rendering pipeline. The method was introduced to address the limitations of traditional volume rendering techniques, such as high memory usage and limited expressiveness, by replacing explicit volume representations with implicit neural representations [S1, unofficial].

## Sources
[S1] instantvnr, https://github.com/VIDILabs/instantvnr, fetched 2026-10-26 (unofficial)
