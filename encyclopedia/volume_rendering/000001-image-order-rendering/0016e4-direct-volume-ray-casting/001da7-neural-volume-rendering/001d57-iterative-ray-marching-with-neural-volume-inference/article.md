# Iterative Ray Marching with Neural Volume Inference


## Overview
Iterative Ray Marching with Neural Volume Inference is a CUDA-based rendering method that splits the volume integration of a neural radiance field into multiple kernel launches to handle rays that do not fully converge (reach alpha $\ge 1$) in a single pass [S1, unofficial]. It distinguishes itself from in-shader neural volume ray marching by decoupling neural network inference from shading logic, using a counter-based compaction mechanism to track and process only active rays across iterations [S1, unofficial].

## How it works
The method processes rays in batches of size $N\_ITERS$. It separates the computation into three distinct phases per iteration: intersection (sample coordinate generation), inference (neural network evaluation), and composition (shading and alpha blending) [S1, unofficial]. A host-side loop manages the execution, checking for remaining active rays between kernel launches [S1, unofficial].

### Inputs and Outputs
**Inputs:**
*   `params`: A structure containing rendering configuration, including:
    *   `counter`: A device integer buffer tracking the number of active rays [S1, unofficial].
    *   `frame_index`: Current frame ID for random number generation [S1, unofficial].
    *   `rendering_mode`: Enum indicating shading type (e.g., `SHADOW`, `NO_SHADING`, `GRADIENT_SHADING`, `SINGLE_SHADE_HEURISTIC`) [S1, unofficial].
    *   `density_scale`: Scalar multiplier for volume density [S1, unofficial].
    *   `sampling_rate`: Controls step size [S1, unofficial].
    *   `volume`: Neural volume context providing the network and transfer function [S1, unofficial].
    *   `camera`: Camera parameters (origin, direction, projection) [S1, unofficial].
    *   `framebuffer_size`: Width and height of the output image [S1, unofficial].
    *   `clipbox`: Lower and upper bounds of the volume of interest [S1, unofficial].
*   `network`: The trained neural network (e.g., TCNN) used to predict density and color [S1, unofficial].
*   `tfn`: Transfer function mapping network output to color and alpha [S1, unofficial].

**Outputs:**
*   `framebuffer`: A buffer of `vec4f` (RGBA) pixels [S1, unofficial].
*   `params.inference_output`: A temporary device buffer of floats storing network outputs for the current batch. Layout is `[numRays * k + i]` where $k$ is the sample index and $i$ is the ray index [S1, unofficial].

### Data Structures
*   **RayPayload (`RayMarchingIter`)**: Stores ray state [S1, unofficial].
    *   `origin`: `vec3f` Ray origin [S1, unofficial].
    *   `direction`: `vec3f` Ray direction [S1, unofficial].
    *   `alpha`: `float` Current accumulated alpha [S1, unofficial].
    *   `color`: `vec3f` Current accumulated color [S1, unofficial].
    *   `t_min`, `t_max`: `float` Intersection interval [S1, unofficial].
*   **Shadow Heuristic State (`ssh`)**: Used in `SINGLE_SHADE_HEURISTIC` mode [S1, unofficial].
    *   `highest_org`: `vec3f` Origin of the highest alpha sample [S1, unofficial].
    *   `highest_color`: `vec3f` Color of the highest alpha sample [S1, unofficial].
    *   `highest_alpha`: `float` Maximum alpha encountered [S1, unofficial].

### Parameters
*   `N_ITERS`: Number of samples processed per kernel launch iteration [S1, unofficial].
*   `numCoordsPerSample`: Number of coordinate values written per sample to the inference input buffer.
    *   If `GRADIENT_SHADING` is enabled: $4 \times N\_ITERS$ (1 position + 3 gradient components per sample) [S1, unofficial].
    *   Otherwise: $N\_ITERS$ (1 position per sample) [S1, unofficial].
*   `nearly_one`: Threshold for alpha convergence (e.g., $0.999$) [S1, unofficial].
*   `gradientStep`: Step size for finite difference gradient estimation [S1, unofficial].
*   `shadingScale`: Weight for blending gradient-based shading with sample color [S1, unofficial].
*   `opacityCorrection`: Correction factor applied to sample alpha [S1, unofficial].

### Steps

1.  **Initialization**:
    *   Reset `params.counter` to zero using `cudaMemsetAsync` [S1, unofficial].
    *   Launch `iterative_raygen_kernel` (either `camera` or `shadow` mode) to generate initial rays [S1, unofficial].
        *   **Camera Mode**: Generate jitter using `RandomTEA` seeded by `frame_index` and ray index. Compute ray origin/direction. If the ray intersects the volume, initialize `RayMarchingIter` and save payload. Otherwise, write default color [S1, unofficial].
        *   **Shadow Mode**: Use pre-computed `highest_org` and `jitter_ssh` from the previous camera pass. Create shadow ray. If it intersects and `highest_alpha > 0`, save payload; otherwise, write existing shading color [S1, unofficial].
    *   Set `params.counter` to the number of rays saved in the payload [S1, unofficial].

2.  **Main Iteration Loop**:
    *   **Check Active Rays**: Asynchronously copy `params.counter` to host memory and synchronize. If `count > 0`, continue [S1, unofficial].
    *   **Reset Counter**: Set `params.counter` to zero for the next iteration's compaction [S1, unofficial].
    *   **Intersection Kernel**: Launch `iterative_intersect_kernel<MODE>` with $N\_ITERS$ threads [S1, unofficial].
        *   For each sample $k$ in the batch:
            *   Compute object space position $c$ using linear interpolation: $c = \text{lerp}(jitter, t.x, t.y)$ [S1, unofficial].
            *   If `GRADIENT_SHADING`: Compute gradient sample points $g_x, g_y, g_z$ by adding `gradientStep` to $c$. Store $c$ and gradients in the inference input buffer [S1, unofficial].
            *   If not gradient shading: Store $c$ in the inference input buffer [S1, unofficial].
    *   **Neural Inference**:
        *   If a network is provided: Call `iterative_sampling_batch_inference` with input size `numCoordsPerSample * numRays`. Output is written to `params.inference_output` [S1, unofficial].
        *   If no network (Ground Truth): Launch `iterative_sampling_groundtruth_kernel` to sample the volume directly [S1, unofficial].
    *   **Composition Kernel**: Launch `iterative_compose_kernel<MODE>` with $N\_ITERS$ threads [S1, unofficial].
        *   For each ray $i$ and sample $k$:
            *   Retrieve sample value $v$ from `params.inference_output` [S1, unofficial].
            *   Apply transfer function to get `sampleColor` and `sampleAlpha` [S1, unofficial].
            *   Apply `opacityCorrection` to `sampleAlpha` [S1, unofficial].
            *   **Shading**:
                *   If `GRADIENT_SHADING`: Compute normal $N_o$ from gradients and sample value. Transform to world space $N_w$. Calculate `shadingColor` (e.g., Phong). Lerp with `sampleColor` using `shadingScale` [S1, unofficial].
                *   If `SINGLE_SHADE_HEURISTIC`: Track the highest alpha sample. If `ssh.highest_alpha < (1.0 - ray.alpha) * sampleAlpha`, update `ssh.highest_org`, `ssh.highest_color`, and `ssh.highest_alpha` [S1, unofficial].
            *   **Volume Blending**:
                *   Calculate transmittance $tr = 1.0 - ray.alpha$ [S1, unofficial].
                *   Update ray alpha: $ray.alpha += tr \times sampleAlpha$ [S1, unofficial].
                *   If `MODE != SHADOW`: Update ray color: $ray.color += tr \times shadingColor \times sampleAlpha$ [S1, unofficial].
            *   **Iteration Control**:
                *   If $k < N\_ITERS$ and $ray.alpha < nearly_one$: Mark ray as active (increment `params.counter` at the corresponding index) [S1, unofficial].
                *   If ray is finished ($ray.alpha \ge nearly_one$ or not resumable):
                    *   Write final result to framebuffer [S1, unofficial].
                    *   For `SHADOW` mode: Lerp `shadingColor` with `final_highest_color * shadingColor.w * transmittance` [S1, unofficial].
                    *   For `SINGLE_SHADE_HEURISTIC`: Save final highest org/color/alpha and `shading_color` [S1, unofficial].
                    *   Otherwise: Write `ray.color` and `ray.alpha` [S1, unofficial].

3.  **Post-Loop**:
    *   If any rays remain active (counter > 0), repeat from Step 2 [S1, unofficial].
    *   Finalize any pending writes to the framebuffer [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import random

def compute_ray_exit(origin, direction, bounds):
    """Compute ray exit distance from volume bounds [G]."""
    t0 = max((bounds[0][i] - origin[i]) / direction[i] if direction[i] != 0 else float('inf') for i in range(3))
    t1 = min((bounds[1][i] - origin[i]) / direction[i] if direction[i] != 0 else float('inf') for i in range(3))
    return max(t0, t1)

def trilinear_interpolate(volume, pos, dims):
    """Standard trilinear interpolation [G]."""
    vx = pos[0] * (dims[0] - 1)
    vy = pos[1] * (dims[1] - 1)
    vz = pos[2] * (dims[2] - 1)
    x0, y0, z0 = int(math.floor(vx)), int(math.floor(vy)), int(math.floor(vz))
    x1, y1, z1 = x0 + 1, y0 + 1, z0 + 1
    x0 = max(0, min(x0, dims[0]-1))
    x1 = max(0, min(x1, dims[0]-1))
    y0 = max(0, min(y0, dims[1]-1))
    y1 = max(0, min(y1, dims[1]-1))
    z0 = max(0, min(z0, dims[2]-1))
    z1 = max(0, min(z1, dims[2]-1))
    wx, wy, wz = vx - x0, vy - y0, vz - z0
    c00 = volume[x0][y0][z0] * (1-wx) + volume[x1][y0][z0] * wx
    c01 = volume[x0][y0][z1] * (1-wx) + volume[x1][y0][z1] * wx
    c10 = volume[x0][y1][z0] * (1-wx) + volume[x1][y1][z0] * wx
    c11 = volume[x0][y1][z1] * (1-wx) + volume[x1][y1][z1] * wx
    r0 = c00 * (1-wy) + c10 * wy
    r1 = c01 * (1-wy) + c11 * wy
    return r0 * (1-wz) + r1 * wz

def iterative_ray_marching(params, network, tfn, volume_data, volume_dims, volume_bounds):
    """
    Main iterative ray marching loop.
    params: Object containing counter, frame_index, rendering_mode, etc.
    network: Neural network model for inference.
    tfn: Transfer function.
    volume_data: 3D array of scalar values.
    volume_dims: [dim_x, dim_y, dim_z].
    volume_bounds: [[min_x, min_y, min_z], [max_x, max_y, max_z]].
    """
    # 1. Initialize ray counter to zero
    params.counter = 0
    
    # 2. Launch Ray Generation
    active_rays = []
    h, w = params.framebuffer_size
    for y in range(h):
        for x in range(w):
            ray_origin, ray_direction = params.camera.cast_ray(x, y)
            t_end = compute_ray_exit(ray_origin, ray_direction, volume_bounds)
            if t_end > 0:
                # Camera mode: generate jitter
                jitter = random.uniform(0.0, params.sampling_rate)
                ray_state = {
                    'origin': ray_origin, 'direction': ray_direction,
                    'alpha': 0.0, 'color': [0.0, 0.0, 0.0],
                    't_min': 0.0, 't_max': t_end, 'jitter': jitter,
                    'ssh': {'highest_org': [0.0, 0.0, 0.0], 'highest_color': [0.0, 0.0, 0.0], 'highest_alpha': 0.0}
                }
                active_rays.append(ray_state)
                params.counter += 1
            else:
                # Write default color if not SINGLE_SHADE_HEURISTIC
                pass

    # 3. Enter main loop: check if any rays remain active
    while params.counter > 0:
        # 4. Reset counter for next iteration's compaction check
        params.counter = 0
        next_active_rays = []
        
        # 5. Intersection Kernel: compute sample coordinates
        for ray in active_rays:
            t = ray['jitter']
            for k in range(params.N_ITERS):
                if ray['alpha'] >= params.nearly_one:
                    break
                    
                # Compute object space position c using lerp(jitter, t.x, t.y)
                c = ray['origin'] + t * ray['direction']
                
                # If GRADIENT_SHADING, compute gradient sample points
                if params.rendering_mode == 'GRADIENT_SHADING':
                    gx = c + [params.gradientStep, 0.0, 0.0]
                    gy = c + [0.0, params.gradientStep, 0.0]
                    gz = c + [0.0, 0.0, params.gradientStep]
                    inference_input = [c, gx, gy, gz]
                else:
                    inference_input = [c]
                    
                # Neural Inference
                if network is not None:
                    sample_value = network(inference_input)
                else:
                    # Ground truth sampling
                    sample_value = trilinear_interpolate(volume_data, c, volume_dims)
                    
                # Composition Kernel
                sample_color, sample_alpha = tfn(sample_value)
                sample_alpha *= params.opacityCorrection
                
                # Shading
                if params.rendering_mode == 'GRADIENT_SHADING':
                    # Compute normal No from gradients and sample value
                    # Transform to world space Nw
                    # Calculate shadingColor
                    shading_color = sample_color # Simplified
                    sample_color = [sample_color[i] * (1 - params.shadingScale) + shading_color[i] * params.shadingScale for i in range(3)]
                    
                # SINGLE_SHADE_HEURISTIC
                if params.rendering_mode == 'SINGLE_SHADE_HEURISTIC':
                    if ray['ssh']['highest_alpha'] < (1.0 - ray['alpha']) * sample_alpha:
                        ray['ssh']['highest_org'] = c
                        ray['ssh']['highest_color'] = sample_color
                        ray['ssh']['highest_alpha'] = (1.0 - ray['alpha']) * sample_alpha
                        
                # Volume blending
                tr = 1.0 - ray['alpha']
                ray['alpha'] += tr * sample_alpha
                if params.rendering_mode != 'SHADOW':
                    ray['color'] = [ray['color'][i] + tr * sample_color[i] * sample_alpha for i in range(3)]
                    
                t += params.sampling_rate
                
                # Continue iteration if k < N_ITERS and ray.alpha < nearly_one
                if k < params.N_ITERS - 1 and ray['alpha'] < params.nearly_one:
                    params.counter += 1
                    next_active_rays.append(ray)
                else:
                    # Write final result
                    if params.rendering_mode == 'SHADOW':
                        ray['color'] = [ray['color'][i] + (1.0 - ray['alpha']) * ray['ssh']['highest_color'][i] * 0.5 for i in range(3)]
                    elif params.rendering_mode == 'SINGLE_SHADE_HEURISTIC':
                        pass # Save final highest org/color/alpha
                    else:
                        pass # Write ray.color and ray.alpha
                    # In actual implementation, write to framebuffer here
                    pass
                    
        active_rays = next_active_rays

    # Finalize
    # Ensure all writes are complete
    pass
```

### Edge Cases and Pitfalls
*   **Wired Error**: The intersection step is separated from ray generation and composition due to an unidentified "wired error" that occurred when they were merged. Merging them did not provide obvious performance benefits [S1, unofficial].
*   **Precision**: Alpha accumulation can suffer from precision issues if `nearly_one` is too close to 1.0 or if transmittance becomes very small.
*   **Boundary Conditions**: Rays that do not intersect the volume must be handled in the ray generation kernel to avoid undefined behavior in subsequent kernels.
*   **Gradient Shading**: Requires careful computation of finite differences. The `gradientStep` must be small enough to approximate the derivative but large enough to avoid numerical noise.

## Operating characteristics
*   **Time Complexity**: $O(R \cdot S / N\_ITERS)$ kernel launches, where $R$ is the number of rays and $S$ is the total number of samples per ray. Each launch involves $N\_ITERS$ samples. The total work is proportional to $R \cdot S$.
*   **Memory Complexity**: $O(N\_ITERS \cdot \text{coords\_per\_sample})$ for inference buffers, plus $O(R)$ for ray payloads and counters. The iterative nature allows processing large scenes with limited GPU memory by batching rays.
*   **Performance**: Performance depends heavily on the neural network inference speed and the efficiency of the counter-based compaction. The separation of kernels allows for better occupancy and memory coalescing compared to in-shader methods, but incurs kernel launch overhead.
*   **Strengths**: Handles rays that require many samples to converge without infinite loops in a single kernel. Decouples network inference from shading, allowing for optimized inference batching.
*   **Limitations**: Kernel launch overhead for each iteration. Complexity in managing ray state across iterations. The "wired error" suggests potential stability issues with merged kernels.

## Background
The method is part of the `instantvnr` library by VIDILabs [S1, unofficial]. It is designed for real-time neural volume rendering, leveraging CUDA and tiny-cuda-nn (TCNN) for efficient inference. The iterative approach addresses the challenge of variable convergence rates among rays in neural volume rendering.

## Sources
[S1] VIDILabs/instantvnr, https://github.com/VIDILabs/instantvnr, fetched 2026-10-26 (unofficial)
