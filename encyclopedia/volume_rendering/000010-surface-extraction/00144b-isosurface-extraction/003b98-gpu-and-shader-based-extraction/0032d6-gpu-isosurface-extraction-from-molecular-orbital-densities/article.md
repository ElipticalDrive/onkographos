# GPU Isosurface Extraction from Molecular Orbital Densities

Also called: GPU isosurface extraction, gridless isosurface extraction

## Overview
This method extracts isosurfaces from molecular orbital densities (Gaussian Type Orbitals, GTOs) using a fully GPU-accelerated, gridless pipeline that avoids blocking the host CPU [S1, unofficial]. It differs from sibling methods by combining an octree-based spatial culling strategy with a "gridless" evaluation of the atomic orbital (AO) sums directly on the GPU, followed by Surface Nets mesh generation and Newton projection [S1, unofficial]. Unlike [GPU Marching Cubes Terrain Pipeline with Indirect Rendering](3803-gpu-marching-cubes-terrain-pipeline-with-indirect-rendering.md) or [GPU-accelerated Marching Cubes with Distance Extraction](3818-gpu-accelerated-marching-cubes-with-distance-extraction.md), which rely on structured grid traversal and marching cubes topology, this approach evaluates the scalar field at arbitrary points defined by brick corners, optimizing for the specific mathematical properties of molecular densities (maxima at atoms, non-negative weights for densities) [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of molecular orbitals defined by GTOs, an iso-value threshold, and an evaluation grid resolution $h$ (typically 0.2 bohr) [S1, unofficial].
*   **Output:** A triangle mesh representing the isosurface, regrouped per field and sheet [S1, unofficial].

**Data Structures:**
*   **Octree:** A spatial hierarchy tested from level 2 to cull empty regions [S1, unofficial].
*   **Brick Hash:** A device-side hash table mapping spatial coordinates to brick slots [S1, unofficial].
*   **Point List:** A list of 3D coordinates representing the corners of the bricks to be evaluated [S1, unofficial].
*   **Job State:** Managed via a non-blocking API (`md_gto_iso_gpu_job_begin`, `poll`, `progress`, `end`) allowing chunked execution of corner and vertex generation [S1, unofficial].

**Parameters:**
*   `GTO_ISO_EVAL_EPS`: Atomic orbital cutoff for evaluation and culling. Default: $10^{-6}$ [S1, unofficial].
*   `GTO_ISO_DFORM`: Enable D-form evaluation (densities as $\phi^T D \phi$). Default: Off [S1, unofficial].
*   `GTO_ISO_FLOOD`: Enable flood approximation for culling. Default: 1 (On) [S1, unofficial].

**Algorithm Steps:**

1.  **Octree Construction:** Build an octree for the molecular domain, starting tests from level 2. This identifies regions of interest [S1, unofficial].
2.  **Node Emission:** Emit kept octree nodes in Morton order using a prefix sum operation to create a compact, ordered list of active bricks [S1, unofficial].
3.  **Brick Hash Initialization:** Build brick slots and the brick hash table on the GPU device to map spatial locations to data structures [S1, unofficial].
4.  **Corner Computation:** Compute the 3D coordinates of the brick corners from the point list. High-face corners are shared between adjacent bricks to reduce redundant computation [S1, unofficial].
    *   For single-function cases, use `iso_corners_list_f1v4`.
    *   For tiled GEMM operations (16/32/64 functions), use `iso_gcorners_f16/32/64`.
    *   For wide GEMM operations (>64 functions), use `iso_gcorners_w128/w192/y256`.
    *   For D-form densities, use `iso_dcorners` [S1, unofficial].
5.  **Flood Approximation (Optional):** If `GTO_ISO_FLOOD` is enabled and the field has maxima at atoms with non-negative weights:
    *   Identify seed bricks: those whose corners straddle the iso-value plus bricks containing an atom [S1, unofficial].
    *   Propagate the "active" status to neighboring bricks to cull large empty regions [S1, unofficial].
    *   *Note:* This step is approximate and not safe for orbitals due to AO term cancellation causing loose octree bounds [S1, unofficial].
6.  **Scalar Evaluation:** Evaluate the molecular density/orbital value at each corner point using the selected GEMM kernel. This replaces dense grid evaluation [S1, unofficial].
7.  **Mesh Generation (Surface Nets):** Generate Surface Nets cells and quads based on the evaluated corner values. This creates a preliminary mesh topology [S1, unofficial].
8.  **Vertex Relaxation and Projection:**
    *   Relax vertices on the lattice surface to improve initial placement [S1, unofficial].
    *   Apply Newton projection to move vertices precisely onto the isosurface $f(x)=0$.
    *   For a single orbital, this is done in one launch (`iso_project_list_f1`).
    *   The projection terminates for a vertex when the remaining error is predicted to be far below the tolerance [S1, unofficial].
9.  **Output Regrouping:** Regroup the final output triangles per field and sheet [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def gpu_iso_extraction(molecular_orbitals, iso_value, h=0.2):
    # 1. Initialize Job
    job = md_gto_iso_gpu_job_begin(molecular_orbitals, iso_value, h)
    
    # 2. Build Octree (Level 2+)
    octree = build_octree(molecular_orbitals, level_start=2)
    
    # 3. Emit Kept Nodes in Morton Order
    kept_nodes = prefix_sum_octree_nodes(octree)
    
    # 4. Build Brick Hash on Device
    brick_hash = build_brick_hash_on_device(kept_nodes)
    
    # 5. Compute Corners
    # Select kernel based on function count and type
    if is_dform():
        corners = iso_dcorners(kept_nodes)
    elif function_count > 64:
        corners = iso_gcorners_wide(kept_nodes)
    elif function_count > 16:
        corners = iso_gcorners_tiled(kept_nodes)
    else:
        corners = iso_corners_list(kept_nodes)
        
    # 6. Flood Approximation (if enabled and applicable)
    if GTO_ISO_FLOOD and has_atom_maxima(molecular_orbitals):
        seeds = find_straddling_bricks(corners, iso_value)
        seeds = union(seeds, bricks_holding_atoms(molecular_orbitals))
        active_bricks = flood_propagation(seeds, brick_hash)
    else:
        active_bricks = kept_nodes
        
    # 7. Evaluate Densities at Corners
    densities = evaluate_gto_densities(corners, molecular_orbitals)
    
    # 8. Generate Surface Nets Mesh
    mesh_cells = surface_nets_generate(densities, active_bricks)
    mesh_quads = surface_nets_quads(mesh_cells)
    
    # 9. Relax and Project Vertices
    relaxed_vertices = relax_lattice_surface(mesh_quads)
    projected_vertices = newton_project(relaxed_vertices, molecular_orbitals, iso_value)
    
    # 10. Finalize and Regroup
    final_mesh = regroup_per_field_sheet(projected_vertices, mesh_quads)
    
    # 11. End Job
    md_gto_iso_gpu_end(job)
    
    return final_mesh
```

**Edge Cases and Pitfalls:**
*   **Orbital Evaluation:** Orbitals at $h=0.2$ are slower than dense evaluation because octree bounds are loose due to AO cancellation [S1, unofficial].
*   **Flood Safety:** Flooding is not safe for orbitals. For example, in mm HOMO extraction, 704 of 6,592 kept bricks have surface, but flooding loses small lobes [S1, unofficial].
*   **Lobes Loss:** Flooding every field loses small lobes, as measured in the Profiler series and `flood_sweep` test [S1, unofficial].
*   **Device Shutdown:** `md_gto_iso_gpu_shutdown` must be called before device destruction. Re-initializing with a new device without shutdown reuses old kernels [S1, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
The method is optimized for parallel evaluation on the GPU. It avoids the memory overhead of storing a full dense grid by only evaluating points in active octree nodes [S1, unofficial].

**Measured Performance:**
*   **RTX 3080 (Vulkan):**
    *   mm density (156 functions): 8.4–8.6 ms (vs. 14.4–16.1 ms for dense evaluation) [S1, unofficial].
    *   tq density (221 functions): 14.1 ms (vs. 15.9–16.5 ms for dense evaluation) [S1, unofficial].
    *   mm HOMO: 2.25–2.3 ms (vs. 0.48 ms for dense evaluation) [S1, unofficial].
*   **M4 Pro (Metal):**
    *   mm density (156 functions): 19.3–19.7 ms (vs. 33–35 ms for dense evaluation) [S1, unofficial].
    *   mm HOMO: 2.42–2.49 ms (vs. 1.07–1.16 ms for dense evaluation) [S1, unofficial].

**Scaling and Trade-offs:**
*   **Densities vs. Orbitals:** Densities (non-negative, maxima at atoms) benefit significantly from the flood approximation and octree culling, beating dense evaluation times. Orbitals suffer from AO cancellation, making octree bounds loose and reducing culling efficiency, resulting in slower performance than dense evaluation for small systems [S1, unofficial].
*   **Non-blocking API:** The job API allows chunked execution, preventing host blocking during long jobs, with progress markers after each chunk [S1, unofficial].
*   **Strengths:** High performance for molecular densities; avoids host-device synchronization bottlenecks; flexible GEMM kernels for varying function counts.
*   **Limitations:** Not suitable for orbitals where flood approximation is unsafe; requires careful management of GPU device lifecycle to avoid kernel reuse bugs [S1, unofficial].

## Background
This method is part of the `mdlib` project, specifically the `md_gto_iso_gpu` module, which has been optimized through rounds r1–r34 [S1, unofficial]. It represents a shift from CPU-based or grid-based isosurface extraction to a fully GPU-accelerated, gridless approach tailored for molecular orbital densities. The implementation uses Vulkan and Metal backends, with profiling tools integrated into the library [S1, unofficial].

## Sources
[S1] md_gto_iso_gpu: optimized GPU isosurface extraction (non-blocking jobs, flood, one measured path), https://github.com/scanberg/mdlib/pull/15, fetched 2026-10-28 (unofficial)
