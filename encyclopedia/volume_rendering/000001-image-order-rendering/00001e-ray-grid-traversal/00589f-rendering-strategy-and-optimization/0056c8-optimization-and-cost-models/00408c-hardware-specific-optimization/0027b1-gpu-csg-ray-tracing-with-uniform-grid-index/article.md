# GPU CSG Ray Tracing with Uniform Grid Index

Also called: GPU CSG Renderer Features, GPU CSG tracer, GpuPlot

## Overview
A GPU-based ray tracer for Constructive Solid Geometry (CSG) that accelerates ray traversal by replacing unindexed full-scans of the root universe with a uniform grid index over the universe's cells [S1, unofficial]. It solves the performance bottleneck where rays outside the model traverse infinite surface extensions, causing redundant full-scans of all region tokens [S1, unofficial]. Unlike its siblings, it operates as a direct f32 transcription of CPU plotting logic into a WGSL shader, maintaining exact visual parity while leveraging hardware-specific optimizations like silent voxel stepping [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A CSG model defined by surfaces, RPN regions, universes, lattices, and translations, flattened into a single `u32` buffer (`outram_blender::csg::gpu::flat`) [S1, unofficial].
*   **Output:** Rendered images (Solid, Wireframe, Slice) via a WGSL shader, or a CPU fallback if GPU constraints are exceeded [S1, unofficial].

**Data Structures:**
1.  **Flat Geometry Buffer (`outram_blender::csg::gpu::flat`):** A single `u32` buffer containing:
    *   Surfaces definitions.
    *   RPN region expressions.
    *   Universe definitions and graph.
    *   Rectangular and Hexagonal lattices (including maps and outer definitions).
    *   Translations.
    *   Static crossing hints (OpenMC-style neighbor lists).
    *   *Note:* Tori are refused [S1, unofficial].
2.  **Uniform Grid Index:** A 3D voxel grid over the root universe's bounding box.
    *   Each voxel stores a list of candidate cells that intersect it.
    *   Each cell's region expression is constant-folded to the specific surfaces passing through that voxel [S1, unofficial].
3.  **GpuPlot Struct:** Contains variants `Solid`, `Wireframe`, and `Slice`, matching the CPU plotter's structs [S1, unofficial].

**Algorithm Steps:**

1.  **Grid Construction:**
    *   Create a voxel grid over the root universe's bounding box.
    *   For each voxel, identify intersecting cells.
    *   Constant-fold each cell's region expression to reduce it to the specific surfaces intersecting that voxel [S1, unofficial].
    *   *Grid Resolution:* Determined by scene complexity. Example: HTR-10 root universe uses a $30 \times 30 \times 47$ voxel grid [S1, unofficial].

2.  **Grid Closing (Bounding Box Enforcement):**
    *   Ensure no root cell lies outside the grid bounds to allow rays outside the model to jump directly to the grid [S1, unofficial].
    *   For unbounded OpenMC boxes (e.g., absorber-sphere channels, core cell conus):
        *   Perform vertex enumeration over plane half-spaces.
        *   Apply a bound on the cone radius to create tight bounding boxes [S1, unofficial].

3.  **Indexing Budgeting:**
    *   Index universes with token counts above 512, within a 16 MB memory budget [S1, unofficial].
    *   If the geometry buffer exceeds the device's 128 MiB binding limit, `set_geometry` returns an error, triggering a CPU fallback [S1, unofficial].
    *   *Example:* Bed tiles (1,500) were not indexed as they cost 29 MB and changed no frame time [S1, unofficial].

4.  **Ray Traversal (Shader Execution):**
    *   **Outside Model:** The ray jumps directly to the uniform grid.
    *   **Inside Grid:** The ray steps through voxels. Voxel faces act as "silent steps" (no surface intersection test required) [S1, unofficial].
    *   **Intersection Test:** Upon entering a voxel, test against the pre-computed surfaces for the candidate cells.
    *   **Complex Region Handling:** Sign complex regions from the surface normal. This corrects a CPU parity bug where complements were not handled via De Morgan's laws [S1, unofficial].
    *   **Post-Step Correction:** After a silent voxel step, reset the token to ensure the ray can exit a bore/cell correctly. Previously, the tracer retained the token of the last-crossed surface, trapping the ray [S1, unofficial].

5.  **Rendering Modes:**
    *   **Phong Solid View:** Render solid geometry with lighting.
    *   **Slice View:** Render section planes with cut faces. Supports per-material visibility and x-ray attenuation with outlines [S1, unofficial].
    *   **ClipPlane Correction:** Ensure shadow rays are lit once they pass into the cut-away half of a clip plane, fixing a bug where the shadow ray was blocked by the cut-away half [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Simplified WGSL-like logic for the traversal step
# Assumes flat buffer is loaded into device memory

fn trace_ray(ray_origin: vec3<f32>, ray_direction: vec3<f32>, grid: UniformGrid) -> Color {
    # 1. Check if ray is outside the root universe bounding box
    if not grid.contains(ray_origin):
        # Jump directly to the grid entry point
        t_entry = grid.compute_entry_distance(ray_origin, ray_direction)
        current_pos = ray_origin + ray_direction * t_entry
    else:
        current_pos = ray_origin

    # 2. Traverse the uniform grid
    while current_pos is inside grid bounds:
        voxel = grid.get_voxel(current_pos)
        
        # Silent step: moving through voxel faces requires no intersection test
        # Advance to next voxel boundary
        t_next = grid.compute_next_voxel_boundary(current_pos, ray_direction)
        next_pos = current_pos + ray_direction * t_next
        
        # 3. Intersection Test at Voxel Entry
        # Test against pre-computed surfaces for candidate cells in the voxel
        for surface in voxel.candidate_surfaces:
            t_hit = compute_intersection(current_pos, ray_direction, surface)
            if t_hit > 0 and t_hit < t_next:
                hit_surface = surface
                hit_t = t_hit
                break
        
        if hit_surface is not null:
            # 4. Complex Region Handling
            # Sign complex regions from the surface normal
            normal = surface.compute_normal(hit_pos)
            region_token = hit_surface.get_region_token()
            
            # Correct De Morgan's law handling for complements
            if region_token.is_complement():
                region_token = sign_from_normal(region_token, normal)
            
            # 5. Post-Step Correction
            # Reset token to allow exit from bores/cells
            current_token = reset_token(region_token)
            
            # Continue traversal from hit point
            current_pos = hit_pos + epsilon * ray_direction
            continue
            
        current_pos = next_pos

    # 6. Render Output (Solid/Slice)
    return render_phong(current_pos, normal)
```

**Parameters:**
*   **Grid Resolution:** Scene-dependent. HTR-10 root universe: $30 \times 30 \times 47$ [S1, unofficial].
*   **Indexing Threshold:** Universes with token counts > 512 [S1, unofficial].
*   **Memory Budget:** 16 MB for indexing [S1, unofficial].
*   **Buffer Limit:** 128 MiB (device binding limit) [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Complement Handling:** The CPU port did not remove complements by De Morgan's laws or sign complex regions from the surface normal. This caused rays entering a reflector core (written as `box & ~core`) to be re-located incorrectly. The fix signs complex regions from the normal [S1, unofficial].
*   **Silent Step Token Retention:** After stepping through a voxel face, the tracer retained the token of the last-crossed surface. This prevented rays inside a bore from exiting. The fix resets the token [S1, unofficial].
*   **ClipPlane Shadow Bug:** In the CPU SolidRayTracePlot with a ClipPlane, the shadow ray was blocked by the cut-away half. The fix ensures the ray is lit once it passes into that half. This changes the shade of 2.5% of pixels but no pixel changes material [S1, unofficial].
*   **Buffer Overflow:** If the geometry buffer exceeds 128 MiB, the system must fall back to CPU rendering to avoid panics [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Cost per crossing is approximately 90 ns [S1, unofficial].
*   **Memory Complexity:** Indexing budget is 16 MB for universes with >512 tokens. Total geometry buffer must fit within 128 MiB [S1, unofficial].
*   **Measured Performance (Şeker bed, half-section, 900x560):**
    *   GPU Indexed: 24-60 ms (best 17 ms) [S1, unofficial].
    *   GPU Unindexed: 0.36-0.39 s [S1, unofficial].
    *   CPU: 6.95 s (from earlier test) [S1, unofficial].
    *   Indexed GPU is ~15-30x faster than unindexed GPU [S1, unofficial].
*   **Scaling:** Performance varies 2-4x from run to run because the GPU is shared with the desktop [S1, unofficial].
*   **Strengths:** Significant speedup over unindexed GPU and CPU methods for large universes; maintains visual parity with CPU renderer.
*   **Limitations:** Buffer size limited to 128 MiB; unindexed rays still suffer from full scans; X-ray views with dense lattices (e.g., TRISO) remain slow (~0.64 s) [S1, unofficial].
*   **When to Use:** When rendering large CSG models on GPU with sufficient VRAM (<128 MiB geometry buffer) and when visual parity with CPU is required.
*   **When Not to Use:** When geometry buffer exceeds 128 MiB or when tori are present (not supported) [S1, unofficial].

## Background
Originated in the `outram-blender` crate, specifically the `outram_blender::csg::gpu` module, as part of the `dhoby-ghaut` project [S1, unofficial]. The implementation was driven by the need to accelerate 3D and 2D views in a reactor physics visualization tool, replacing a slow CPU-based ray tracer with a GPU-accelerated version using WGSL shaders [S1, unofficial]. The method was developed to address the performance bottleneck of unindexed ray traversal in large CSG universes, such as the HTR-10 and Şeker bed models [S1, unofficial].

## Sources
[S1] dhoby-ghaut: GPU rendering for the 3D and 2D views (sphere impostors for pebbles; GPU or accelerated CSG ray tracing), https://github.com/theodoreOnzGit/outram-park-backend/issues/587, fetched 2026-10-26 (unofficial)
