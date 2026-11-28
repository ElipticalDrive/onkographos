# GPU Isosurface Extraction from Molecular Orbital Densities

Also called: GPU isosurface extraction, gridless isosurface extraction

## Overview
This method extracts isosurfaces from molecular orbital densities represented by Gaussian Type Orbitals (GTOs) directly on the GPU, avoiding the generation of a uniform volumetric grid [S1, unofficial]. It solves the problem of efficiently visualizing complex quantum chemical fields by using an octree for spatial culling and a gridless brick-based Surface Nets approach [S1, unofficial].

## How it works
The pipeline processes GTO fields to produce triangle meshes [S1, unofficial]. It operates in two modes: a blocking extraction (`md_gto_iso_gpu_extract`) and a non-blocking job API (`md_gto_iso_gpu_job_begin`, `poll`, `progress`, `end`) which chunks corner and vertex launches for long jobs [S1, unofficial].

### Inputs and Outputs
*   **Input:** A set of GTO functions, an iso-value, and a bounding box [S1, unofficial].
*   **Output:** A list of triangles (mesh) regrouped per field and sheet [S1, unofficial].

### Data Structures
*   **Octree:** A spatial hierarchy tested starting from level 2 [S1, unofficial].
*   **Brick Slots & Hash:** Device-side storage mapping brick coordinates to allocated memory slots [S1, unofficial].
*   **Point List:** A list of 3D coordinates representing the corners of the bricks [S1, unofficial].
*   **Corner Data:** Computed density values and gradients at brick corners [S1, unofficial].
*   **Vertex List:** The final mesh vertices after relaxation and projection [S1, unofficial].

### Parameters
*   `GTO_ISO_EVAL_EPS`: AO cutoff for evaluation and culling. Default: $10^{-6}$ [S1, unofficial].
*   `GTO_ISO_DFORM`: Enable D form (densities as $\phi^T D \phi$). Default: off [S1, unofficial].
*   `GTO_ISO_FLOOD`: Enable flood approximation. Default: 1 [S1, unofficial].

### Steps

1.  **Octree Construction:** Build an octree for the GTO field, testing nodes from level 2. Nodes are evaluated to determine if they contain the isosurface [S1, unofficial].
2.  **Node Emission:** Emit kept nodes in Morton order. A prefix sum is used to compact the list of kept nodes [S1, unofficial].
3.  **Brick Setup:** Build brick slots and the brick hash on the device [S1, unofficial].
4.  **Corner Computation:** Compute corner values from the point list. High-face corners are shared between adjacent bricks to ensure continuity. The kernel depends on the function set size: `iso_corners_list_f1v4` (one function), `iso_gcorners_f16/32/64` (tiled GEMM for up to 64 functions), `iso_gcorners_w128/w192/y256` (wide GEMM for canonical sets > 64 functions), and `iso_dcorners` (for D form) [S1, unofficial].
5.  **Flood Approximation (Optional):** If `GTO_ISO_FLOOD` is enabled and the field has maxima at atoms with non-negative weights, seeds are identified as bricks whose corners straddle the iso value plus bricks holding an atom [S1, unofficial].
6.  **Surface Nets Generation:** Generate Surface Nets cells and quads from the corner data [S1, unofficial].
7.  **Relaxation and Projection:** Relax vertices on the lattice surface, then apply Newton projection to snap vertices to the exact isosurface defined by the GTOs. One orbital runs the whole projection in one launch (`iso_project_list_f1`). A step ends the vertex if the remaining error is predicted far below the tolerance [S1, unofficial].
8.  **Regrouping:** Regroup the output per field and sheet [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import itertools

def gpu_iso_extraction(gto_functions, iso_value, bounding_box, device):
    # 1. Octree Construction: Build octree tested from level 2.
    # Evaluate AO at node centers/bounds to cull nodes below GTO_ISO_EVAL_EPS.
    octree = build_octree_from_gto(gto_functions, bounding_box, level_start=2)
    kept_nodes = [n for n in octree.nodes if n.culled is False]
    
    # 2. Node Emission: Emit kept nodes in Morton order via prefix sum.
    morton_codes = [compute_morton_code(n.coords) for n in kept_nodes]
    prefix_sum = list(itertools.accumulate(morton_codes))
    compacted_nodes = [kept_nodes[i] for i in prefix_sum]
    
    # 3. Brick Setup: Build brick slots and brick hash on device.
    brick_slots = [None] * len(compacted_nodes)
    brick_hash = {n.coords: i for i, n in enumerate(compacted_nodes)}
    
    # 4. Corner Computation: Compute corners from point list, sharing high-face corners.
    point_list = generate_corner_points(compacted_nodes)
    if len(gto_functions) == 1:
        corners = iso_corners_list_f1v4(point_list, gto_functions)
    elif len(gto_functions) <= 64:
        corners = iso_gcorners_tiled_gemm(point_list, gto_functions)
    else:
        corners = iso_gcorners_wide_gemm(point_list, gto_functions)
    if GTO_ISO_DFORM:
        corners = iso_dcorners(corners, gto_functions)
        
    # 5. Flood Approximation: Seeds are bricks straddling iso + bricks holding an atom.
    seeds = []
    if GTO_ISO_FLOOD and is_density_field(gto_functions):
        seeds = [b for b in compacted_nodes if straddles_iso(b, corners, iso_value)]
        seeds += [b for b in compacted_nodes if b.holds_atom()]
        
    # 6. Surface Nets Generation: Generate Surface Nets cells and quads.
    surface_nets_cells = generate_surface_nets(corners, iso_value, seeds)
    
    # 7. Relaxation and Projection: Relax on lattice, then Newton projection.
    vertices = extract_vertices(surface_nets_cells)
    vertices = relax_vertices(vertices, surface_nets_cells)
    for orbital in gto_functions:
        vertices = newton_project(vertices, orbital, iso_value)
        
    # 8. Regrouping: Regroup output per field and sheet.
    meshes = regroup_per_field_and_sheet(vertices, gto_functions)
    return meshes
```

### Edge Cases and Pitfalls
*   **Orbital Evaluation Speed:** Orbitals at $h=0.2$ are slower than dense evaluation because octree bounds are loose due to AO cancellation. For example, in mm HOMO extraction, 704 of 6,592 kept bricks have surface, but the loose bounds cause unnecessary computation [S1, unofficial].
*   **Flooding Safety:** Flooding is not safe for orbitals because of the loose bounds. For mm HOMO, flooding loses small lobes [S1, unofficial].
*   **Flooding Every Field:** Flooding every field loses small lobes, as measured in the Profiler series and flood_sweep test [S1, unofficial].
*   **Device Shutdown:** `md_gto_iso_gpu_shutdown` must be called before device destruction. Re-initializing with a new device without shutdown reuses old kernels [S1, unofficial].

## Subtopics

### [Non-blocking GPU Isosurface Extraction with Progress](11069-non-blocking-gpu-isosurface-extraction-with-progress.md)
Extracts isosurfaces using a non-blocking job API, capping per-frame processing time. It builds on this method by adding asynchronous job management and selective flooding.

## Operating characteristics
*   **Performance:**
    *   **RTX 3080:**
        *   mm density (156 functions): 8.4–8.6 ms extraction vs. 14.4–16.1 ms dense evaluation.
        *   tq density (221 functions): 14.1 ms extraction vs. 15.9–16.5 ms dense evaluation.
        *   mm HOMO: 2.25–2.3 ms extraction vs. 0.48 ms dense evaluation.
    *   **M4 Pro:**
        *   mm density (156 functions): 19.3–19.7 ms extraction vs. 33–35 ms dense evaluation.
        *   mm HOMO: 2.42–2.49 ms extraction vs. 1.07–1.16 ms dense evaluation.
    *   Densities beat dense evaluation. Orbitals at $h=0.2$ stay slower than dense evaluation due to loose octree bounds [S1, unofficial].
*   **Complexity:** The method uses an octree for culling, which reduces the number of bricks processed compared to a full grid. The complexity depends on the sparsity of the isosurface and the number of GTO functions.
*   **Strengths:** Efficient for dense fields (densities). Non-blocking API allows for interactive rendering.
*   **Limitations:** Less efficient for orbitals due to AO cancellation leading to loose bounds. Flooding is not safe for orbitals.
*   **When to use:** When visualizing molecular densities where performance is critical and the field is dense.
*   **When not to use:** When visualizing orbitals with high precision requirements for small lobes, or when the AO cancellation makes the octree bounds too loose.

## Background
This method is part of the `mdlib` project, specifically the `md_gto_iso_gpu` component. It has been optimized through rounds r1–r34, with the branch carrying a single path after r32. It was measured using `viamd's Profiler` on RTX 3080 (Vulkan) and M4 Pro (Metal) hardware [S1, unofficial].

## Sources
[S1] md_gto_iso_gpu: optimized GPU isosurface extraction (non-blocking jobs, flood, one measured path), https://github.com/scanberg/mdlib/pull/15, fetched 2026-10-28 (unofficial)
