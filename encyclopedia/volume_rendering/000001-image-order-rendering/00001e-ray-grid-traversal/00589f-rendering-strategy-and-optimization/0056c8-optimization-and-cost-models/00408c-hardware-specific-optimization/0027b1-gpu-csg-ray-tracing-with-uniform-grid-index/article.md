# GPU CSG Ray Tracing with Uniform Grid Index

Also called: GPU CSG Renderer Features, GPU CSG tracer, GpuPlot

## Overview
A GPU-based ray tracer for Constructive Solid Geometry (CSG) that accelerates traversal by replacing unindexed full-scans of the root universe with a uniform grid index over the universe's cells [S1, unofficial]. It solves the performance bottleneck where rays outside the model traverse infinite surface extensions, causing redundant full-scans of all region tokens [S1, unofficial]. Unlike its siblings [UltraVis Cache Optimization](10355-ultravis-cache-optimization.md), [VIZARD II](13297-vizard-ii.md), and [Hardware-Aware Hyperparameter Tuning](17622-hardware-aware-hyperparameter-tuning.md), it operates as a direct f32 transcription of CPU plotting logic into a WGSL shader, maintaining exact visual parity while leveraging hardware-specific optimizations like silent voxel stepping [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A CSG model defined by surfaces, RPN regions, universes, lattices, and translations, flattened into a single `u32` buffer (`outram_blender::csg::gpu::flat`) [S1, unofficial].
*   **Output:** Rendered images (Solid, Wireframe, Slice) via a WGSL shader, or a CPU fallback if GPU constraints are exceeded [S1, unofficial].

**Data Structures:**
1.  **Flat Geometry Buffer (`outram_blender::csg::gpu::flat`):** A single `u32` buffer containing: surfaces definitions; RPN region expressions; universe definitions and graph; rectangular and hexagonal lattices (including maps and outer definitions); translations; static crossing hints (OpenMC-style neighbor lists); and the universe graph used to skip subtrees with no drawn material [S1, unofficial]. Tori are refused [S1, unofficial].
2.  **Uniform Grid Index:** A 3D voxel grid over the root universe's bounding box [S1, unofficial]. Each voxel stores a list of candidate cells that intersect it [S1, unofficial]. Each cell's region expression is constant-folded to the specific surfaces passing through that voxel [S1, unofficial].
3.  **GpuPlot Struct:** Contains variants `Solid`, `Wireframe`, and `Slice`, matching the CPU plotter's structs [S1, unofficial].

**Algorithm Steps:**
1.  **Grid Construction:** Create a voxel grid over the root universe's bounding box [S1, unofficial]. For each voxel, identify intersecting cells [S1, unofficial]. Constant-fold each cell's region expression to reduce it to the specific surfaces intersecting that voxel [S1, unofficial]. Grid resolution is scene-dependent; the HTR-10 root universe uses a $30 \times 30 \times 47$ voxel grid [S1, unofficial].
2.  **Grid Closing (Bounding Box Enforcement):** Ensure no root cell lies outside the grid bounds to allow rays outside the model to jump directly to the grid [S1, unofficial]. For unbounded OpenMC boxes (e.g., absorber-sphere channels, core cell conus), perform vertex enumeration over plane half-spaces [S1, unofficial]. Apply a bound on the cone radius to create tight bounding boxes [S1, unofficial].
3.  **Indexing Budgeting:** Index universes with token counts above 512, within a 16 MB memory budget [S1, unofficial]. If the geometry buffer exceeds the device's 128 MiB binding limit, `set_geometry` returns an error, triggering a CPU fallback [S1, unofficial]. Bed tiles (1,500) were not indexed as they cost 29 MB and changed no frame time [S1, unofficial].
4.  **Ray Traversal (Shader Execution):** When a ray is outside the model, it jumps directly to the uniform grid [S1, unofficial]. Inside the grid, the ray steps through voxels [S1, unofficial]. Voxel faces act as "silent steps" (no surface intersection test required) [S1, unofficial]. Upon entering a voxel, test against the pre-computed surfaces for the candidate cells [S1, unofficial]. Sign complex regions from the surface normal to correct a CPU parity bug where complements were not handled via De Morgan's laws [S1, unofficial]. After a silent voxel step, reset the token to ensure the ray can exit a bore/cell correctly, fixing a bug where the tracer retained the token of the last-crossed surface [S1, unofficial].
5.  **Rendering Modes:** Phong Solid View renders solid geometry with lighting [S1, unofficial]. Slice View renders section planes with cut faces, supporting per-material visibility and x-ray attenuation with outlines [S1, unofficial]. ClipPlane Correction ensures shadow rays are lit once they pass into the cut-away half of a clip plane, fixing a bug where the shadow ray was blocked by the cut-away half [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def trace_ray(origin, direction, grid):
    # 1. Outside model: jump directly to grid
    if not grid.contains(origin):
        t_entry = grid.compute_entry_distance(origin, direction)
        current_pos = origin + direction * t_entry
    else:
        current_pos = origin

    # 2. Traverse grid
    while grid.contains(current_pos):
        voxel = grid.get_voxel(current_pos)

        # Silent step: advance to next voxel boundary without intersection test
        t_next = grid.compute_next_voxel_boundary(current_pos, direction)
        next_pos = current_pos + direction * t_next

        # 3. Intersection test upon entering voxel
        hit_surface = None
        hit_t = float('inf')
        for surface in voxel.candidate_surfaces:
            # Standard ray-surface intersection test
            t_hit = compute_surface_intersection(current_pos, direction, surface)
            if 0 < t_hit < hit_t:
                hit_surface = surface
                hit_t = t_hit

        if hit_surface is not None:
            # 4. Complex region handling: sign from surface normal
            normal = compute_surface_normal(hit_surface)
            region_token = hit_surface.get_region_token()
            if region_token.is_complement():
                # Apply De Morgan's correction by signing from normal
                region_token = sign_region_from_normal(region_token, normal)

            # 5. Post-step correction: reset token to allow exit
            current_token = reset_token(region_token)

            # Advance past hit
            current_pos = hit_surface.hit_point + direction * 1e-6
            continue

        # Silent step completed, move to next voxel
        current_pos = next_pos

    return render_output(current_pos)

# Helper definitions for standard operations
def compute_surface_intersection(origin, direction, surface):
    # Standard analytical ray-surface intersection; returns t or infinity
    pass

def compute_surface_normal(surface):
    # Returns the geometric normal vector of the surface at the hit point
    pass

def sign_region_from_normal(token, normal):
    # Adjusts token sign based on normal direction to fix complement parity
    pass

def reset_token(token):
    # Clears retained surface token to permit ray exit from bores/cells
    pass

def render_output(pos):
    # Applies Phong lighting or slice attenuation based on GpuPlot variant
    pass
```

**Parameters:**
*   **Grid Resolution:** Scene-dependent. HTR-10 root universe: $30 \times 30 \times 47$ [S1, unofficial].
*   **Indexing Threshold:** Universes with token counts > 512 [S1, unofficial].
*   **Memory Budget:** 16 MB for indexing [S1, unofficial].
*   **Buffer Limit:** 128 MiB (device binding limit) [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Complement Handling:** The CPU port did not remove complements by De Morgan's laws or sign complex regions from the surface normal [S1, unofficial]. This caused rays entering a reflector core (written as `box & ~core`) to be re-located incorrectly [S1, unofficial]. The fix signs complex regions from the normal [S1, unofficial].
*   **Silent Step Token Retention:** After stepping through a voxel face, the tracer retained the token of the last-crossed surface [S1, unofficial]. This prevented rays inside a bore from exiting [S1, unofficial]. The fix resets the token [S1, unofficial].
*   **ClipPlane Shadow Bug:** In the CPU SolidRayTracePlot with a ClipPlane, the shadow ray was blocked by the cut-away half [S1, unofficial]. The fix ensures the ray is lit once it passes into that half [S1, unofficial]. This changes the shade of 2.5% of pixels but no pixel changes material [S1, unofficial].
*   **Buffer Overflow:** If the geometry buffer exceeds 128 MiB, the system must fall back to CPU rendering to avoid panics [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Cost per crossing is approximately 90 ns [S1, unofficial].
*   **Memory Complexity:** Indexing budget is 16 MB for universes with >512 tokens [S1, unofficial]. Total geometry buffer must fit within 128 MiB [S1, unofficial].
*   **Measured Performance (Şeker bed, half-section, 900x560):** GPU Indexed: 24-60 ms (best 17 ms) [S1, unofficial]. GPU Unindexed: 0.36-0.39 s [S1, unofficial]. CPU: 6.95 s (from earlier test) [S1, unofficial]. Indexed GPU is ~15-30x faster than unindexed GPU [S1, unofficial].
*   **Scaling:** Performance varies 2-4x from run to run because the GPU is shared with the desktop [S1, unofficial].
*   **Strengths:** Significant speedup over unindexed GPU and CPU methods for large universes; maintains visual parity with CPU renderer [S1, unofficial].
*   **Limitations:** Buffer size limited to 128 MiB; unindexed rays still suffer from full scans; X-ray views with dense lattices (e.g., TRISO) remain slow (~0.64 s) [S1, unofficial].
*   **When to Use:** When rendering large CSG models on GPU with sufficient VRAM (<128 MiB geometry buffer) and when visual parity with CPU is required [S1, unofficial].
*   **When Not to Use:** When geometry buffer exceeds 128 MiB or when tori are present (not supported) [S1, unofficial].

## Background
Originated in the `outram-blender` crate, specifically the `outram_blender::csg::gpu` module, as part of the `dhoby-ghaut` project [S1, unofficial]. The implementation was driven by the need to accelerate 3D and 2D views in a reactor physics visualization tool, replacing a slow CPU-based ray tracer with a GPU-accelerated version using WGSL shaders [S1, unofficial]. The method was developed to address the performance bottleneck of unindexed ray traversal in large CSG universes, such as the HTR-10 and Şeker bed models [S1, unofficial].

## Sources
[S1] dhoby-ghaut: GPU rendering for the 3D and 2D views (sphere impostors for pebbles; GPU or accelerated CSG ray tracing), https://github.com/theodoreOnzGit/outram-park-backend/issues/587, fetched 2026-10-26 (unofficial)
