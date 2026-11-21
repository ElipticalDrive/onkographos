# Coherent and Connectivity Traversal


## Overview
This family accelerates ray-grid traversal by exploiting spatial coherence among ray bundles or by enforcing strict topological connectivity rules to resolve boundary ambiguities [G]. It solves the problem of packet divergence in coherent ray tracing and normal inaccuracies in uniform grid traversal [G].

## Shared principles
All methods in this family build on [Ray-grid traversal](17-ray-grid-traversal.md), which advances analytically from one cell boundary to the next [Parent]. Optimization and cost models do not alter this fundamental geometric logic but instead optimize the cost of executing it or the parameters under which it runs [Parent].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ray_grid_traversal(ray_origin, ray_direction, voxel_size, grid_dims):
    # Initialize current voxel index
    current_voxel = [int(ray_origin[i] // voxel_size) for i in range(3)]
    
    # Step direction per axis
    step = [1 if ray_direction[i] > 0 else (-1 if ray_direction[i] < 0 else 0) for i in range(3)]
    
    # Distance to next boundary
    # If d > 0, next boundary is at voxel+1. If d < 0, next boundary is at voxel.
    next_boundary = [(current_voxel[i] + (1 if ray_direction[i] > 0 else 0)) * voxel_size for i in range(3)]
    t_max = [(next_boundary[i] - ray_origin[i]) / ray_direction[i] if ray_direction[i] != 0 else float('inf') for i in range(3)]
    
    # Distance delta per axis
    t_delta = [voxel_size / abs(ray_direction[i]) if ray_direction[i] != 0 else float('inf') for i in range(3)]
    
    t_entry = 0.0
    
    while 0 <= current_voxel[0] < grid_dims[0] and \
          0 <= current_voxel[1] < grid_dims[1] and \
          0 <= current_voxel[2] < grid_dims[2]:
        
        min_idx = t_max.index(min(t_max))
        t_exit = t_max[min_idx]
        
        yield current_voxel, t_entry, t_exit
        
        current_voxel[min_idx] += step[min_idx]
        t_max[min_idx] += t_delta[min_idx]
        t_entry = t_exit
```

## Subtopics

### [Coherent Grid Traversal](2987-coherent-grid-traversal.md)
Accelerates ray tracing by processing bundles of coherent rays as frustums, advancing slice-by-slice along a dominant axis rather than cell-by-cell. It solves the packet divergence problem inherent in cell-by-cell traversal (like 3DDDA), where rays in a group disagree on the next cell, by maintaining packet coherence through incremental frustum-slice overlap computation. Unlike hierarchical structures like kd-trees, CGT uses a flat uniform grid, enabling extremely fast per-frame rebuilds suitable for fully dynamic scenes.

### [Hero-like 4-Connectivity (H4T)](3436-hero-like-4-connectivity-h4t.md)
Guarantees strict 4-connectivity by explicitly resolving boundary crossing ambiguities via stable sorting of intersection distances. It solves the problem of diagonal jumps (8-connectivity errors) and associated normal inaccuracies that occur when a ray crosses an edge or corner in standard Digital Differential Analyzer (DDA) approaches. Unlike its sibling [Hierarchical Planes 4-Connectivity (HP4T)](3444-hierarchical-planes-4-connectivity-hp4t.md), which uses plane intersections and clamping, H4T relies on a 2x2 cell test and stable sorting to determine the first cell entered, achieving 100% correctness in randomized testing.

### Ray Connectivity
#### definition
- [S1, unofficial] A property defining the set of cells a ray can traverse to from its current cell, inspired by pixel connectivity in image processing.
#### formula
- [S1, unofficial] Taxicab metric distance between $P_1=(x_1, y_1)$ and $P_2=(x_2, y_2)$ is $|x_1 - x_2| + |y_1 - y_2|$.
- [S1, unofficial] Chebyshev metric distance between $P_1=(x_1, y_1)$ and $P_2=(x_2, y_2)$ is $\max(|x_1 - x_2|, |y_1 - y_2|)$.
#### number
- [S1, unofficial] In 2D, there are 4-connectivity (orthogonal neighbors) and 8-connectivity (including diagonal neighbors).

## Background
The concept of connectivity in ray traversal was formalized in a thesis by Hur [S1, unofficial], which analyzed the behavior of rays crossing voxel corners. The thesis noted that standard DDA methods might traverse through corners, leading to 8-connectivity, whereas specific algorithms aim for 4-connectivity to ensure correct normal calculation and topological consistency.

## Sources
[S1] versal algorithm for ray tracing Hur man gör en hierarkisk ..., https://kth.diva-portal.org/smash/get/diva2:2069409/FULLTEXT01.pdf, fetched 2026-10-25 (unofficial)
