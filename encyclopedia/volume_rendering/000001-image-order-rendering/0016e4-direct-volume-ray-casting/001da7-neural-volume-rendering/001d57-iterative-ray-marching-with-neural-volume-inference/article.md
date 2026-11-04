# Iterative Ray Marching with Neural Volume Inference


## Overview
Iterative Ray Marching with Neural Volume Inference is a CUDA-based rendering method that splits the volume integration of a neural radiance field into multiple kernel launches to handle rays that do not fully converge (reach alpha $\ge 1$) in a single pass [S1, unofficial]. It distinguishes itself from in-shader neural volume ray marching by decoupling the neural network inference from the shading logic, using a counter-based compaction mechanism to track and process only active rays across iterations [S1, unofficial].

## How it works
The method processes rays in batches of size $N\_ITERS$. It separates the computation into three distinct phases per iteration: intersection (sample coordinate generation), inference (neural network evaluation), and composition (shading and alpha blending). A host-side loop manages the execution, checking for remaining active rays between kernel launches.

### Inputs and Outputs
**Inputs:**
*   `params`: A structure containing rendering configuration, including:
    *   `counter`: A device integer buffer tracking the number of active rays.
    *   `frame_index`: Current frame ID for random number generation.
    *   `rendering_mode`: Enum indicating shading type (e.g., `SHADOW`, `NO_SHADING`, `GRADIENT_SHADING`, `SINGLE_SHADE_HEURISTIC`).
    *   `density_scale`: Scalar multiplier for volume density.
    *   `sampling_rate`: Controls step size.
    *   `volume`: Neural volume context providing the network and transfer function.
    *   `camera`: Camera parameters (origin, direction, projection).
    *   `framebuffer_size`: Width and height of the output image.
    *   `clipbox`: Lower and upper bounds of the volume of interest.
*   `network`: The trained neural network (e.g., TCNN) used to predict density and color.
*   `tfn`: Transfer function mapping network output to color and alpha.

**Outputs:**
*   `framebuffer`: A buffer of `vec4f` (RGBA) pixels.
*   `params.inference_output`: A temporary device buffer of floats storing network outputs for the current batch. Layout is `[numRays * k + i]` where $k$ is the sample index and $i$ is the ray index [S1, unofficial].

### Data Structures
*   **RayPayload (`RayMarchingIter`)**: Stores ray state.
    *   `origin`: `vec3f` Ray origin.
    *   `direction`: `vec3f` Ray direction.
    *   `alpha`: `float` Current accumulated alpha.
    *   `color`: `vec3f` Current accumulated color.
    *   `t_min`, `t_max`: `float` Intersection interval.
*   **Shadow Heuristic State (`ssh`)**: Used in `SINGLE_SHADE_HEURISTIC` mode.
    *   `highest_org`: `vec3f` Origin of the highest alpha sample.
    *   `highest_color`: `vec3f` Color of the highest alpha sample.
    *   `highest_alpha`: `float` Maximum alpha encountered.

### Parameters
*   `N_ITERS`: Number of samples processed per kernel launch iteration. Typical values depend on memory bandwidth and network batch size. [S1, unofficial]
*   `numCoordsPerSample`: Number of coordinate values written per sample to the inference input buffer.
    *   If `GRADIENT_SHADING` is enabled: $4 \times N\_ITERS$ (1 position + 3 gradient components per sample).
    *   Otherwise: $N\_ITERS$ (1 position per sample). [S1, unofficial]
*   `nearly_one`: Threshold for alpha convergence (e.g., $0.999$). [S1, unofficial]
*   `gradientStep`: Step size for finite difference gradient estimation. [S1, unofficial]
*   `shadingScale`: Weight for blending gradient-based shading with sample color. [S1, unofficial]
*   `opacityCorrection`: Correction factor applied to sample alpha. [S1, unofficial]

### Steps

1.  **Initialization**:
    *   Reset `params.counter` to zero using `cudaMemsetAsync` [S1, unofficial].
    *   Launch `iterative_raygen_kernel` (either `camera` or `shadow` mode) to generate initial rays.
        *   **Camera Mode**: Generate jitter using `RandomTEA` seeded by `frame_index` and ray index. Compute ray origin/direction. If the ray intersects the volume, initialize `RayMarchingIter` and save payload. Otherwise, write default color.
        *   **Shadow Mode**: Use pre-computed `highest_org` and `jitter_ssh` from the previous camera pass. Create shadow ray. If it intersects and `highest_alpha > 0`, save payload; otherwise, write existing shading color.
    *   Set `params.counter` to the number of rays saved in the payload.

2.  **Main Iteration Loop**:
    *   **Check Active Rays**: Asynchronously copy `params.counter` to host memory and synchronize. If `count > 0`, continue.
    *   **Reset Counter**: Set `params.counter` to zero for the next iteration's compaction.
    *   **Intersection Kernel**: Launch `iterative_intersect_kernel<MODE>` with $N\_ITERS$ threads.
        *   For each sample $k$ in the batch:
            *   Compute object space position $c$ using linear interpolation: $c = \text{lerp}(jitter, t.x, t.y)$.
            *   If `GRADIENT_SHADING`: Compute gradient sample points $g_x, g_y, g_z$ by adding `gradientStep` to $c$. Store $c$ and gradients in the inference input buffer.
            *   If not gradient shading: Store $c$ in the inference input buffer.
    *   **Neural Inference**:
        *   If a network is provided: Call `iterative_sampling_batch_inference` with input size `numCoordsPerSample * numRays`. Output is written to `params.inference_output`.
        *   If no network (Ground Truth): Launch `iterative_sampling_groundtruth_kernel` to sample the volume directly.
    *   **Composition Kernel**: Launch `iterative_compose_kernel<MODE>` with $N\_ITERS$ threads.
        *   For each ray $i$ and sample $k$:
            *   Retrieve sample value $v$ from `params.inference_output`.
            *   Apply transfer function to get `sampleColor` and `sampleAlpha`.
            *   Apply `opacityCorrection` to `sampleAlpha`.
            *   **Shading**:
                *   If `GRADIENT_SHADING`: Compute normal $N_o$ from gradients and sample value. Transform to world space $N_w$. Calculate `shadingColor` (e.g., Phong). Lerp with `sampleColor` using `shadingScale`.
                *   If `SINGLE_SHADE_HEURISTIC`: Track the highest alpha sample. If `ssh.highest_alpha < (1.0 - ray.alpha) * sampleAlpha`, update `ssh.highest_org`, `ssh.highest_color`, and `ssh.highest_alpha`.
            *   **Volume Blending**:
                *   Calculate transmittance $tr = 1.0 - ray.alpha$.
                *   Update ray alpha: $ray.alpha += tr \times sampleAlpha$.
                *   If `MODE != SHADOW`: Update ray color: $ray.color += tr \times shadingColor \times sampleAlpha$.
            *   **Iteration Control**:
                *   If $k < N\_ITERS$ and $ray.alpha < nearly_one$: Mark ray as active (increment `params.counter` at the corresponding index).
                *   If ray is finished ($ray.alpha \ge nearly_one$ or not resumable):
                    *   Write final result to framebuffer.
                    *   For `SHADOW` mode: Lerp `shadingColor` with `final_highest_color * shadingColor.w * transmittance`.
                    *   For `SINGLE_SHADE_HEURISTIC`: Save final highest org/color/alpha and `shading_color`.
                    *   Otherwise: Write `ray.color` and `ray.alpha`.

3.  **Post-Loop**:
    *   If any rays remain active (counter > 0), repeat from Step 2.
    *   Finalize any pending writes to the framebuffer.

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def iterative_ray_marching(params, network, tfn):
    """
    Main iterative ray marching loop.
    params: Object containing counter, frame_index, rendering_mode, etc.
    network: Neural network model for inference.
    tfn: Transfer function.
    """
    # 1. Initialize
    cuda_memset_async(params.counter, 0)
    
    # Launch Ray Generation
    if params.rendering_mode == SHADOW:
        launch_kernel(iterative_raygen_kernel_shadow, params)
    else:
        launch_kernel(iterative_raygen_kernel_camera, params)
    
    # Get initial active ray count
    # Note: In practice, this involves async copy and sync
    active_count = get_counter_value(params.counter)
    
    while active_count > 0:
        # 2. Reset counter for next iteration compaction
        cuda_memset_async(params.counter, 0)
        
        # 3. Intersection Kernel
        # Computes sample coordinates and gradient points if needed
        launch_kernel(iterative_intersect_kernel, params, grid_size=active_count)
        
        # 4. Neural Inference
        if network is not None:
            # Input buffer size depends on mode
            input_size = params.numCoordsPerSample * active_count
            inference_output = network.forward(params.inference_input, batch_size=input_size)
            params.inference_output = inference_output
        else:
            launch_kernel(iterative_sampling_groundtruth_kernel, params)
            
        # 5. Composition Kernel
        # Shading, blending, and determining next iteration status
        launch_kernel(iterative_compose_kernel, params, grid_size=active_count)
        
        # 6. Check for remaining active rays
        active_count = get_counter_value(params.counter)
        
    # 7. Finalize
    # Ensure all writes are complete
    cuda_stream_synchronize()
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
