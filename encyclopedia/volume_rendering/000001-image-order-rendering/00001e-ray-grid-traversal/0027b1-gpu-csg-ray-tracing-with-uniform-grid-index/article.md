# GPU CSG Ray Tracing with Uniform Grid Index

Also called: GPU CSG Renderer Features, GPU CSG tracer, GpuPlot

## Overview
This method is a GPU-accelerated ray tracer for Constructive Solid Geometry (CSG) that replaces unindexed full-scan traversal with a uniform grid index over the root universe's cells [S1, unofficial]. It solves the performance bottleneck where rays outside the model traverse infinite CSG extensions (planes, cones), causing redundant full-scans of all surface tokens [S1, unofficial]. By pre-computing voxel-to-cell mappings and constant-folding region expressions, it reduces frame times from hundreds of milliseconds to tens of milliseconds [S1, unofficial]. It differs from standard DDA [3061-digital-differential-analyzer-dda-traversal.md] by integrating CSG region evaluation directly into the grid traversal loop, and from hierarchical methods [1122-hierarchical-digital-differential-analyzer-hdda.md] by using a fixed uniform grid rather than a tree structure.

## How it works
**Inputs and Outputs:**
*   **Input:** CSG geometry defined by surfaces, RPN region expressions, universes, lattices, and translations [S1, unofficial].
*   **Output:** Rendered image (Solid, Wireframe, or Slice views) with lighting, attenuation, and outlines [S1, unofficial].

**Data Structures:**
1.  **Flat Geometry Buffer (`outram_blender::csg::gpu::flat`):** A single `u32` buffer containing:
    *   Surfaces
    *   RPN regions
    *   Universes
    *   Rect/Hex lattices (with maps and outer definitions)
    *   Translations
    *   Static crossing hints (OpenMC-style neighbor lists)
    *   Universe graph (used to skip subtrees with no drawn material) [S1, unofficial]
2.  **Uniform Grid Index:** A 3D voxel grid over the root universe's bounding box. Each voxel stores a list of intersecting cells. Each cell's region is constant-folded to the specific surfaces passing through that voxel [S1, unofficial].
3.  **GpuPlot Struct:** Contains variants `Solid`, `Wireframe`, `Slice`, matching the CPU plotter's structs [S1, unofficial].

**Algorithm Steps:**

1.  **Grid Construction:**
    *   Create a voxel grid over the root universe's bounding box.
    *   For each voxel, identify intersecting cells.
    *   Constant-fold each cell's region expression to determine the specific surfaces passing through the voxel [S1, unofficial].
    *   **Closing the Grid:** Ensure no root cell lies outside the grid bounds. For unbounded OpenMC boxes (e.g., absorber-sphere channels, core cell conus), perform vertex enumeration over plane half-spaces and apply a bound on the cone radius to create tight bounding boxes [S1, unofficial].

2.  **Indexing Budget:**
    *   Index universes with token counts above 512, within a 16 MB memory budget [S1, unofficial].
    *   Example: Bed tiles (1,500) were not indexed as they cost 29 MB and changed no frame time [S1, unofficial].
    *   Buffer Limit: The geometry buffer must not exceed the device's 128 MiB binding limit. If exceeded, `set_geometry` returns an error (CPU fallback) instead of panicking [S1, unofficial].

3.  **Ray Traversal Loop (WGSL Shader):**
    *   **Outside Model:** If the ray is outside the model, it jumps directly to the grid [S1, unofficial].
    *   **Inside Grid:** The ray steps through voxels. Voxel faces act as 'silent steps' (no surface intersection test required) [S1, unofficial].
    *   **Intersection Test:** Upon entering a voxel, test against the pre-computed surfaces for the candidate cells [S1, unofficial].
    *   **Region Evaluation:** Sign complex regions from the surface normal. This corrects a CPU parity bug where complements were not handled via De Morgan's laws [S1, unofficial].
    *   **Post-Step Correction:** After a silent voxel step, reset the token to ensure the ray can exit a bore/cell correctly. Previously, the tracer kept the token of the last-crossed surface, trapping the ray [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode for GPU CSG Ray Tracing with Uniform Grid Index
# Based on [S1, unofficial]

struct Ray {
    origin: Vec3
    direction: Vec3
}

struct Voxel {
    cells: List<CellIndex>
    surfaces: List<SurfaceIndex>
}

struct CSGRenderer {
    grid: VoxelGrid
    flat_buffer: FlatGeometryBuffer
}

fn trace_ray(renderer: CSGRenderer, ray: Ray) -> Color {
    # 1. Check if ray is outside the model
    if not is_inside_model(renderer.grid, ray.origin):
        # Jump directly to the grid
        current_voxel = get_voxel_at(ray.origin, renderer.grid)
    else:
        current_voxel = get_voxel_at(ray.origin, renderer.grid)

    # 2. Traverse the grid
    while ray_is_within_bounds(ray, renderer.grid):
        # Silent step: voxel faces do not require intersection tests
        # Test against pre-computed surfaces for candidate cells
        for surface_idx in current_voxel.surfaces:
            hit = intersect_surface(ray, surface_idx, renderer.flat_buffer)
            if hit.is_some():
                # 3. Region Evaluation
                # Sign complex regions from the surface normal
                region_token = evaluate_region(hit.surface, renderer.flat_buffer)
                if is_inside_region(region_token, ray):
                    # 4. Post-Step Correction
                    # Reset token to allow exit from bore/cell
                    reset_token()
                    return render_surface(hit, renderer.flat_buffer)
        
        # Advance to next voxel
        current_voxel = next_voxel(current_voxel, ray.direction, renderer.grid)

    return background_color
}

fn evaluate_region(surface: Surface, flat_buffer: FlatGeometryBuffer) -> RegionToken {
    # Constant-folded region expression for the voxel
    # Sign complex regions from the surface normal
    return flat_buffer.get_region_token(surface)
}
```

**Parameters:**
*   **Grid Resolution:** HTR-10 root universe uses a 30x30x47 voxel grid [S1, unofficial].
*   **Indexing Threshold:** Universes with token counts above 512 are indexed [S1, unofficial].
*   **Memory Budget:** 16 MB for indexing [S1, unofficial].
*   **Buffer Limit:** 128 MiB [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Complement Handling:** The CPU port did not remove complements by De Morgan's laws or sign complex regions from the surface normal. This caused rays entering a reflector core (written as `box & ~core`) to be re-located incorrectly. The fix signs complex regions from the normal [S1, unofficial].
*   **Silent Step Token Retention:** After stepping through a voxel face (silent step), the tracer retained the token of the last-crossed surface. This prevented rays inside a bore from exiting. The fix resets the token [S1, unofficial].
*   **ClipPlane Shadow Bug:** In the CPU SolidRayTracePlot with a ClipPlane, the shadow ray was blocked by the cut-away half. The fix ensures the ray is lit once it passes into that half. This changes the shade of 2.5% of pixels but no pixel changes material [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Cost per crossing is approximately 90 ns [S1, unofficial]. The reflector cells hold up to 124 region tokens, contributing to this cost [S1, unofficial].
*   **Memory Complexity:** Indexing budget is 16 MB [S1, unofficial]. Geometry buffer limit is 128 MiB [S1, unofficial].
*   **Measured Performance:**
    *   **Şeker bed, half-section, 900x560:** GPU indexed time 24-60 ms (best 17 ms). GPU unindexed time 0.36-0.39 s. CPU time 6.95 s [S1, unofficial].
    *   **Şeker reflector, half-section:** GPU indexed time 19-21 ms. GPU unindexed time 0.36 s [S1, unofficial].
    *   **x-y slice, mid-bed:** GPU indexed time 2 ms. GPU unindexed time 17 ms [S1, unofficial].
    *   **DEM bed, half-section:** GPU indexed time 33 ms. GPU unindexed time 0.39 s [S1, unofficial].
    *   **DEM reflector, half-section:** GPU indexed time 20 ms. GPU unindexed time 0.35 s [S1, unofficial] [S1, unofficial].
*   **Scaling:** Indexing reduces frame time from ~0.36s to 24-60ms (min 17ms) for the Şeker bed half-section [S1, unofficial]. Unindexed rays performed ~10 full scans of the 128-cell root universe because rays outside the model stepped through infinite extensions (planes, cones) crossing all 1,675 tokens [S1, unofficial].
*   **Strengths:** Significant performance improvement over unindexed GPU and CPU methods. Supports Solid, Wireframe, and Slice views with lighting and attenuation [S1, unofficial].
*   **Limitations:** Buffer limit of 128 MiB may require CPU fallback. Slice view does not do overlap checks, level or mesh lines [S1, unofficial]. GPU times vary 2-4x from run to run because the GPU is shared with the desktop [S1, unofficial].

## Background
This method was developed by theodoreOnzGit and committed on Oct 5, 2026, as part of the `outram-park-backend` project [S1, unofficial]. It is part of the `outram_blender` crate, which owns the CSG types [S1, unofficial]. The WGSL shader is an f32 transcription of the CPU plotter's raytrace, slice, geometry, cell, and lattice logic [S1, unofficial]. It was verified on NVIDIA RTX A5000 with Vulkan [S1, unofficial].

## Sources
[S1] dhoby-ghaut: GPU rendering for the 3D and 2D views (sphere impostors for pebbles; GPU or accelerated CSG ray tracing), https://github.com/theodoreOnzGit/outram-park-backend/issues/587, fetched 2026-10-26 (unofficial)
