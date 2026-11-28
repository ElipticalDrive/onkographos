# Progressive Ray Tracing
Also called: Progressive Path Tracing with Preview Scaling, Progressive Refinement Ray Casting

## Overview
Progressive ray tracing is a technique for producing lower quality images which are then improved with time [S2, unofficial]. It addresses the problem of interactive volume rendering by computing only a subset of rays during user interaction and progressively filling in the remaining pixels when interaction stops [S3, book]. Unlike its parent [Checkerboard Progressive Rendering](4836-checkerboard-progressive-rendering.md), which reduces per-frame compute by tracing only half the pixels each frame while blending results over time, progressive ray tracing typically uses spatial subsampling (computing every $n$-th ray) combined with interpolation during interaction, followed by full-resolution refinement [S3, book].

## How it works
**Inputs and Outputs:**
- **Inputs:** Scene data (density, emission, materials), camera parameters, image resolution, preview scaling factor (`pratio`), total sample count (`samples`), and exposure value.
- **Outputs:** A rendered image buffer that starts as a low-resolution, single-sample preview and converges to a high-resolution, multi-sample result.

**Data Structures:**
- `app->render`: 2D array/vector indexed by `{i, j}` (x, y) storing pixel values before tonemapping [S1, unofficial].
- `app->display`: 2D array/vector indexed by `{i, j}` storing tonemapped pixel values for display [S1, unofficial].
- `pprms`: Preview state structure containing `resolution` and `samples` [S1, unofficial].
- `pstate`: Temporary render state for preview tracing [S1, unofficial].
- `app->render_state`: Full-resolution render state for progressive refinement [S1, unofficial].

**Parameters:**
- `pratio`: Integer preview scaling factor. Preview resolution is divided by this value. Range: 1 to 64 [S1, unofficial].
- `exposure`: Float tonemapping exposure value. Range: -5 to 5 [S1, unofficial].
- `samples`: Integer total sample count for progressive rendering. Range: 16 to 4096 [S1, unofficial].

**Steps:**
1. **Initialize Preview State:** Set the preview resolution by dividing the application resolution by `pratio` and set preview samples to 1 [S1, unofficial].
   ```python
   pprms.resolution = app.resolution / pratio
   pprms.samples = 1
   ```
2. **Trace Preview Samples:** During user interaction, trace rays into the temporary preview state using the reduced resolution [S1, unofficial].
   ```python
   trace_samples(pstate, app.scene, app.camera, pprms)
   ```
3. **Interpolate (Optional during interaction):** If using the refinement variant, compute only every $n$-th ray along each image dimension and use interpolation to fill in the remaining pixels [S3, book].
4. **Downsample Preview to Full Resolution:** Iterate over full image coordinates `(i, j)` and map to preview coordinates `(pi, pj)` using clamped division [S1, unofficial].
   ```python
   for i in range(app.resolution.x):
       for j in range(app.resolution.y):
           pi = clamp(i / pratio, 0, pstate.render.size.x - 1)
           pj = clamp(j / pratio, 0, pstate.render.size.y - 1)
           app.render[{i, j}] = pstate.render[{pi, pj}]
           app.display[{i, j}] = tonemap(app.render[{i, j}], app.exposure)
   ```
5. **Launch Asynchronous Worker Loop:** Start a background loop to progressively render full-resolution samples [S1, unofficial].
   ```python
   for sample in range(app.params.samples):
       # Update progress fraction
       app.progress = sample / app.params.samples
       
       # Trace full-resolution samples
       trace_samples(app.render_state, app.scene, app.camera, app.params, &app.render_stop)
       
       # Check stop flag
       if app.render_stop:
           return
       
       # Copy and tonemap full resolution
       for i in range(app.resolution.x):
           for j in range(app.resolution.y):
               app.render[{i, j}] = app.render_state.render[{i, j}]
               app.display[{i, j}] = tonemap(app.render[{i, j}], app.exposure)
   ```

**Edge Cases and Pitfalls:**
- **Precision:** Clamping indices during downsampling prevents out-of-bounds access but may introduce aliasing if `pratio` is large [S1, unofficial].
- **Boundaries:** Ensure `pratio` is within [1, 64] to avoid excessive blur or performance issues [S1, unofficial].
- **Degenerate Input:** If `samples` is 1, no progressive refinement occurs, and the preview is displayed directly [S1, unofficial].

## Subtopics

### [FBSlabs](16350-fbslabs.md)
FBSlabs is a progressive ray casting technique that refines a low-resolution volume rendering into high resolution by splitting each ray into segments of fixed length (slabs) and rendering them in front-to-back order over subsequent frames. It addresses the problem of interactive volume rendering on mobile and low-performance devices by distributing the computational workload of high-resolution ray casting across multiple frames, ensuring a constant frame rate during user interaction. Unlike sibling methods such as [STiles](9724-stiles.md), which sort screen-space tiles by cost, FBSlabs distributes work uniformly across all pixels per frame, making it suitable for hardware lacking compute shaders.

## Operating characteristics
- **Time Complexity:** During interaction, time complexity is reduced by a factor of $n^2$ where $n$ is the subsampling factor (e.g., every 2nd ray results in 4x speed-up) [S3, book]. Progressive refinement time is proportional to the number of samples and resolution.
- **Memory Complexity:** Requires storage for both preview and full-resolution render states, plus temporary interpolation buffers if used [S1, unofficial].
- **Measured Performance:** Computing only every $n$-th ray during interaction results in a four-times speed-up without noticeable decrease in image quality for $n=2$ [S3, book]. Further speed-ups can be achieved with larger reductions, but at the cost of blurry, less detailed images [S3, book].
- **Strengths:** Provides immediate visual feedback during interaction; improves image quality over time without blocking user input [S2, unofficial].
- **Limitations:** High-frequency details may be lost in the preview phase; interpolation artifacts may be visible if not handled carefully [S2, unofficial][S3, book].
- **When to Use:** Interactive volume rendering on hardware with limited compute resources; scenarios where immediate feedback is more important than final image quality [S3, book].
- **When Not to Use:** Offline rendering where final image quality is paramount and time is not a constraint; scenes with very high-frequency details that require full resolution from the start [S2, unofficial].

## Background
Progressive ray tracing was introduced as a technique for producing lower quality images which are then improved with time [S2, unofficial]. It is part of the broader field of adaptive sampling and resolution in volume rendering, aiming to balance interactivity and image quality [S3, book]. The method is particularly relevant for distributed volume visualization and interactive applications where real-time feedback is critical [S2, unofficial].

## Sources
[S1] RickyMexx/yocto-gl-vpt, https://github.com/RickyMexx/yocto-gl-vpt, fetched 2026-10-26 (unofficial)
[S2] Massive Data Management for Distributed Volume Visualization, http://hdl.handle.net/1951/47671, fetched 2026-10-26 (unofficial)
[S3] Chapter 7 - Advanced Computer Graphics - VTK, https://examples.vtk.org/site/VTKBook/07Chapter7/, fetched 2026-10-29 (book)
