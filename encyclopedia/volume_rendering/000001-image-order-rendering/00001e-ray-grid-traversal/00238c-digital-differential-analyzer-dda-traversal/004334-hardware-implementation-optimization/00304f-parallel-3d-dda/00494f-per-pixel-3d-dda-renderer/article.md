# Per-pixel 3D-DDA Renderer

Also called: Brickmap 3D-DDA, CPU-based 3D-DDA Renderer

## Overview
The Per-pixel 3D-DDA Renderer is a CPU-based volume rendering method that performs a 3D Digital Differential Analyzer (DDA) traversal per pixel over a brickmap [S1]. It serves as a clean-room alternative to column-coherent raycasting, specifically designed for the roxlap engine [S1]. Unlike its parent [Parallel 3D-DDA](6247-parallel-3d-dda.md) which focuses on general parallelization strategies, this method implements a strict per-pixel traversal order that enables correct compositing of transparent voxels without depth sorting or Order-Independent Transparency (OIT) schemes [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A brickmap data structure containing voxel data; a frame buffer; camera parameters (position, orientation, projection).
*   **Output:** A rendered image frame buffer.

**Data Structures:**
*   **Brickmap:** The spatial data structure storing voxel data. The specific internal layout is not detailed in the sources, but it supports runtime editing (carve/fill spans, cubes, rectangles, spheres) via `roxlap_formats::edit::*` [S1].
*   **Material Palette:** A 256-entry palette used to drive per-instance, per-voxel, and world-terrain material properties, supporting mixed opaque-frame and glass models [S1].
*   **Color Callbacks:** Closure-based functions replacing global state, supporting constant, jittered, position-dependent, and texture-mapped color functions [S1].

**Steps:**
1.  **Frame Partitioning:** The renderer splits the frame into row strips for parallel processing using the `rayon` library [S1].
2.  **Ray Initialization:** For each pixel in a strip, initialize a ray based on camera parameters.
3.  **3D-DDA Traversal:** Traverse the brickmap along the ray. The traversal must visit voxels strictly front-to-back [S1].
4.  **Empty Space Skipping:** Implement empty space skipping to optimize performance. A regression test ensures no leaks occur in empty space handling [S1].
5.  **Voxel Sampling:** Sample the voxel at the current DDA step. Retrieve color and material properties from the Material Palette or via color callbacks [S1].
6.  **Compositing:** Composite the voxel color into the pixel buffer. For transparent voxels, use front-to-back compositing. Volumetric opacity is weighted by the ray's path length [S1].
7.  **Termination:** Stop traversal when the ray exits the brickmap or the accumulated opacity reaches a threshold (e.g., 1.0), depending on the compositing model [S1].

**Formulas:**
*   **Volumetric Weighting:** Opacity is weighted by the ray's path length. If $\Delta s$ is the step size along the ray and $\alpha$ is the voxel opacity, the contribution is proportional to $\Delta s \cdot \alpha$ [S1].
*   **Front-to-Back Compositing:** For a sequence of voxels $i$ with transmittance $T_i$ and color $C_i$, the final color $C_{final}$ is computed as:
    $$ C_{final} = \sum_{i} C_i \cdot \alpha_i \cdot \prod_{j < i} (1 - \alpha_j) $$
    where $\alpha_i$ is the effective opacity of voxel $i$ [S1].

**Parameters:**
*   **Material Palette Size:** 256 entries [S1].
*   **Parallelism:** Controlled by `RAYON_NUM_THREADS` environment variable [S1].

**Pitfalls:**
*   **Empty Space Leaks:** Incorrect implementation of empty space skipping can lead to visual artifacts (leaks). A specific regression test is required to ensure correctness [S1].
*   **Global State:** Avoid global state for color functions; use closure-based callbacks to prevent state management issues [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_frame(brickmap, camera, frame_buffer, material_palette):
    # Split frame into row strips for parallel processing
    strips = split_into_row_strips(frame_buffer.height)
    
    # Parallel processing using rayon (conceptual)
    for strip in strips:
        for y in strip.y_range:
            for x in strip.x_range:
                # Initialize ray for pixel (x, y)
                ray = camera.get_ray(x, y)
                
                # Initialize compositing state
                accumulated_color = (0.0, 0.0, 0.0, 0.0)
                transmittance = 1.0
                
                # 3D-DDA Traversal
                current_voxel = brickmap.get_voxel_at(ray.origin)
                while current_voxel is not None and transmittance > 0.0:
                    # Empty space skipping check [G]
                    if current_voxel.is_empty:
                        current_voxel = brickmap.step_ray(ray, current_voxel)
                        continue
                    
                    # Sample voxel properties
                    material = material_palette[current_voxel.material_id]
                    color_func = get_color_callback(material)
                    voxel_color = color_func(current_voxel.position, current_voxel.material_id)
                    voxel_alpha = material.alpha
                    
                    # Weight opacity by ray path length [S1]
                    step_size = ray.step_size
                    effective_alpha = voxel_alpha * step_size
                    
                    # Front-to-back compositing [S1]
                    contribution = voxel_color * effective_alpha * transmittance
                    accumulated_color += contribution
                    transmittance *= (1.0 - effective_alpha)
                    
                    # Advance ray
                    current_voxel = brickmap.step_ray(ray, current_voxel)
                
                # Write to frame buffer
                frame_buffer.set_pixel(x, y, accumulated_color)
```

## Subtopics

### Transparent Voxel Compositing
A compositing strategy for transparent voxels (alpha-blended, additive, Beer–Lambert volumetric) that relies on front-to-back traversal order, eliminating the need for depth sorting or Order-Independent Transparency (OIT) schemes [S1]. It supports constant, jittered, position-dependent, and texture-mapped color functions via closure-based callbacks, replacing the global-state dance of the original engine [S1]. Volumetric opacity is weighted by the ray's path length, causing filled clouds to appear denser at their core [S1]. It uses a 256-entry material palette to drive per-instance and per-voxel material properties, supporting mixed opaque-frame and glass models [S1].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of pixels and the average number of voxels traversed per ray [S1]. Parallelized across CPU cores using `rayon` [S1].
*   **Memory Complexity:** Proportional to the size of the brickmap and the frame buffer [S1].
*   **Performance:** Feature-complete CPU renderer with SIMD support (SSE2 / NEON / wasm simd128) [S1]. Correctness is validated against a dense per-voxel reference walk [S1].
*   **Strengths:** Clean-room implementation; no dependency on original Voxlap C source; supports complex material palettes; deterministic on CPU [S1].
*   **Limitations:** CPU-bound; performance depends on brickmap traversal efficiency; empty space skipping must be carefully implemented to avoid leaks [S1].
*   **When to Use:** When a deterministic, CPU-based volume renderer is needed for brickmap data, especially for transparent voxels requiring correct front-to-back compositing [S1].
*   **When Not to Use:** When GPU acceleration is required or when column-coherent raycasting is preferred for specific hardware architectures [S1].

## Background
The Per-pixel 3D-DDA Renderer is part of the roxlap project, an independent Rust implementation inspired by Ken Silverman's Voxlap engine [S1]. It was developed as a clean-room alternative to Voxlap's column-coherent raycaster [S1]. The renderer is part of the `roxlap-cli` crate, published on crates.io [S1]. The project includes documentation on porting, multicore performance, and scene graph integration [S1].

## Sources
[S1] roxlap-cli 0.32.0, https://docs.rs/crate/roxlap-cli/latest, fetched 2026-10-29
