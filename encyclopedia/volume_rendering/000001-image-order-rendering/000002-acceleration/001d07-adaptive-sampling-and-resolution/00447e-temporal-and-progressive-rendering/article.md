# Temporal and Progressive Rendering

Also called: Progressive Volume Rendering, Preview-then-Refine Rendering

## Overview
Temporal and Progressive Rendering is a method that decouples immediate visual feedback from final image quality by first generating a low-resolution preview and then iteratively refining it with full-resolution samples [S1, unofficial]. It solves the problem of long initial wait times in volume rendering by providing a usable image immediately, while progressively reducing noise and increasing resolution [S1, unofficial]. It sets itself apart from [Checkerboard Progressive Rendering](4836-checkerboard-progressive-rendering.md) by using a resolution-scaling strategy rather than spatial subsampling, and from [Temporal Upsampling and Dithering](2872-temporal-upsampling-and-dithering.md) by relying on iterative sample accumulation rather than temporal blending of full-resolution frames [S1, unofficial].

## How it works
**Inputs:** Scene description (volumes, lights, camera), target resolution $R_{target}$, preview ratio $pratio$, total sample count $N_{samples}$, and exposure $E$.
**Outputs:** Final rendered image buffer.

**Data Structures:**
*   `app->render`: 2D array of size $R_{target} \times R_{target}$ storing linear pixel values.
*   `app->display`: 2D array of size $R_{target} \times R_{target}$ storing tonemapped pixel values for display.
*   `pprms`: Preview state object containing `resolution` (set to $R_{target} / pratio$) and `samples` (set to 1).
*   `app->render_state`: Full-resolution rendering state.

**Parameters:**
*   $pratio$: Integer preview scaling factor. Range: $[1, 64]$ [S1, unofficial].
*   $N_{samples}$: Total sample count for progressive refinement. Range: $[16, 4096]$ [S1, unofficial].
*   $E$: Exposure value for tonemapping. Range: $[-5, 5]$ [S1, unofficial].

**Steps:**
1.  **Initialize Preview State:** Set preview resolution $R_{preview} = \lfloor R_{target} / pratio \rfloor$ and preview sample count $N_{preview} = 1$ [S1, unofficial].
2.  **Trace Preview:** Execute volume tracing with $R_{preview}$ and $N_{preview}$ to generate a low-resolution image buffer `pstate->render` [S1, unofficial].
3.  **Upscale and Display:** For each pixel $(i, j)$ in the full-resolution display grid:
    *   Map to preview coordinates: $p_i = \text{clamp}(\lfloor i / pratio \rfloor, 0, R_{preview}.x - 1)$, $p_j = \text{clamp}(\lfloor j / pratio \rfloor, 0, R_{preview}.y - 1)$ [S1, unofficial].
    *   Copy value: `app->render[{i, j}] = pstate->render[{p_i, p_j}]` [S1, unofficial].
    *   Apply tonemapping: `app->display[{i, j}] = tonemap(app->render[{i, j}], E)` [S1, unofficial].
4.  **Progressive Refinement Loop:** For $s = 0$ to $N_{samples} - 1$:
    *   Update progress fraction: $P = s / N_{samples}$ [S1, unofficial].
    *   Check stop flag; if set, terminate [S1, unofficial].
    *   Trace full-resolution samples into `app->render_state` [S1, unofficial].
    *   Copy result: `app->render[{i, j}] = app->render_state->render[{i, j}]` [S1, unofficial].
    *   Apply tonemapping: `app->display[{i, j}] = tonemap(app->render[{i, j}], E)` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def temporal_progressive_render(scene, camera, R_target, pratio, N_samples, exposure):
    # 1. Initialize Preview State
    R_preview_x = max(1, R_target.x // pratio)
    R_preview_y = max(1, R_target.y // pratio)
    
    # 2. Trace Preview
    pstate_render = trace_samples(
        resolution=(R_preview_x, R_preview_y),
        samples=1,
        scene=scene,
        camera=camera
    )
    
    # 3. Upscale and Display
    render_buffer = zeros((R_target.x, R_target.y))
    display_buffer = zeros((R_target.x, R_target.y))
    
    for i in range(R_target.x):
        for j in range(R_target.y):
            pi = clamp(i // pratio, 0, R_preview_x - 1)
            pj = clamp(j // pratio, 0, R_preview_y - 1)
            render_buffer[i, j] = pstate_render[pi, pj]
            display_buffer[i, j] = tonemap(render_buffer[i, j], exposure)
            
    # 4. Progressive Refinement Loop
    for s in range(N_samples):
        # Update progress
        progress = s / N_samples
        
        # Check stop flag (implementation detail)
        if check_stop_flag():
            break
            
        # Trace full resolution
        full_render = trace_samples(
            resolution=R_target,
            samples=1, # Or accumulated depending on implementation
            scene=scene,
            camera=camera
        )
        
        # Copy and Tonemap
        for i in range(R_target.x):
            for j in range(R_target.y):
                render_buffer[i, j] = full_render[i, j]
                display_buffer[i, j] = tonemap(render_buffer[i, j], exposure)
                
    return display_buffer
```

**Edge Cases and Pitfalls:**
*   **Division by Zero:** If $pratio$ is 0, resolution calculation fails. Ensure $pratio \ge 1$ [S1, unofficial].
*   **Boundary Clamping:** The mapping $i / pratio$ must be clamped to valid preview indices to avoid out-of-bounds access [S1, unofficial].
*   **Precision:** Linear buffer values should be stored in floating-point to prevent clipping before tonemapping [S1, unofficial].

## Subtopics

### [Temporal Upsampling and Dithering](2872-temporal-upsampling-and-dithering.md)
Combines spatial dithering with temporal accumulation to reduce noise in real-time volume rendering. Chosen when computational budget limits per-frame sampling density and temporal stability is required without full resolution.

### [Checkerboard Progressive Rendering](4836-checkerboard-progressive-rendering.md)
Traces only half the pixels in a checkerboard pattern each frame, blending with previous frames. Chosen when spatial subsampling is preferred over resolution scaling for per-frame cost reduction.

### [Multi-Resolution Rendering for Clouds](7260-multi-resolution-rendering-for-clouds.md)
Renders at low resolution first to identify regions for high-resolution sampling. Chosen when hierarchical data structures allow skipping expensive evaluations in empty areas.

## Operating characteristics
*   **Time Complexity:** Initial preview is $O((R_{target}/pratio)^2)$. Each progressive step is $O(R_{target}^2)$. Total time is $O((R_{target}/pratio)^2 + N_{samples} \cdot R_{target}^2)$ [S1, unofficial].
*   **Memory Complexity:** $O(R_{target}^2)$ for the render and display buffers [S1, unofficial].
*   **Performance:** Provides immediate visual feedback, reducing perceived latency. The preview step is significantly faster due to reduced resolution [S1, unofficial].
*   **Strengths:** Fast initial image, progressive improvement, simple implementation [S1, unofficial].
*   **Limitations:** Preview may have aliasing artifacts; progressive steps may show flickering if not accumulated properly [S1, unofficial].
*   **When to Use:** Interactive applications where immediate feedback is critical [S1, unofficial].
*   **When Not to Use:** Offline rendering where final quality is prioritized over initial feedback time [S1, unofficial].

## Background
Originated in the context of progressive path tracing for volumetric rendering, specifically implemented in the Yocto-GL VPT project [S1, unofficial]. The method leverages the principle that a low-resolution preview can be generated quickly, providing a baseline image that is then refined [S1, unofficial].

## Sources
[S1] RickyMexx/yocto-gl-vpt, https://github.com/RickyMexx/yocto-gl-vpt, fetched 2026-10-26 (unofficial)
