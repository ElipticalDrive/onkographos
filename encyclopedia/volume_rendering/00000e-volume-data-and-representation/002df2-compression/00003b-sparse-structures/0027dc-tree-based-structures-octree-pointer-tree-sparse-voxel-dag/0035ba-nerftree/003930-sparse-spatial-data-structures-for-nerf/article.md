# Sparse spatial data structures for NeRF

Also called: Octree-based NeRF, Sparse Parametric Encodings (Octree/Sparse Grid), Sparse voxel grids, VDB-based NeRF

## Overview
This family of methods uses sparse data structures (sparse grids, octrees, VDBs) to localize the surface interface and concentrate samples only in relevant regions, effectively skipping empty space to reduce MLP evaluations [S1]. By refining the sampling strategy to focus solely on the localized surface, these methods significantly reduce the number of required MLP evaluations, thereby improving computational efficiency [S1]. Sparse voxel grids allow skipping empty regions during ray tracing, leading to rendering speedups of up to 10 times compared to standard NeRF [S1].

## Shared principles
The core challenge in applying sparse structures to NeRF is that surfaces emerge during training, so space-partitioning data structures cannot be precomputed in advance and must be periodically updated [S1, S2]. To address this, methods typically adopt a multistage, coarse-to-fine strategy where regions are progressively refined and pruned as necessary [S1, S2]. The general workflow involves: (1) learning implicit functions over an initial voxel grid to produce a coarse approximation of scene geometry [S1]; (2) progressively pruning empty voxels during training based on this initial geometry signal [S1]; and (3) using hierarchical structures or sparse grids to cull unused features from dense grids, improving efficiency when the surface of interest is known or can be progressively refined [S2].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sample_ray_vdb(ray_origin, ray_direction, vdb_grid, step_mode, ray_length):
    """Samples a NeRF ray using VDB grid traversal."""
    samples = []
    t = 0.0
    # Not given in the sources.
    step_size = 0.05 if step_mode == 'constant' else 0.01
    max_steps = 1000  # [G] Safety limit

    for _ in range(max_steps):
        if t >= ray_length:
            break
        pos = ray_origin + ray_direction * t
        status, tile_bounds = vdb_grid.query(pos)

        if status == 'EMPTY':
            t += step_size
        elif status == 'OCCUPIED':
            # [G] Evaluate MLP at pos to get color and alpha
            samples.append((pos, 0.0, 0.0))
            t += step_size
        elif status == 'TILE_OCCUPIED':
            while t < tile_bounds.max_t and t < ray_length:
                pos = ray_origin + ray_direction * t
                # [G] Evaluate MLP at pos to get color and alpha
                samples.append((pos, 0.0, 0.0))
                t += step_size
        elif status == 'TILE_EMPTY':
            t = tile_bounds.max_t
        else:
            break

        if step_mode == 'linear':
            step_size *= 1.1  # Not given in the sources.
    return samples
```

## Subtopics

### [VDB-based Occupancy Grid Conversion](11675-vdb-based-occupancy-grid-conversion.md)
This method replaces the dense binary Occupancy Grid (OG) of a trained NeRF model with a VDB grid to reduce spatial redundancy caused by empty voxels, accelerating ray-sampling without re-training the model. It sets itself apart from sibling methods like Octree-based pruning by fixing the tree depth to save traversal costs and leveraging hierarchical structures for ray-tracing, whereas siblings like PlenVDB use VDB only for memory efficiency without hierarchical ray traversal. It is chosen when post-training acceleration is required without modifying the MLP, though it is not suitable for frequent updates during training because VDB grids on GPU are read-only and CPU pruning requires costly re-transfers.

## Background
The observation that only around 10 to 20% of evaluated samples have non-zero density and contribute to the final image motivated acceleration methods that localize the surface interface [S1]. Early approaches like NSVF [LGZL∗ 20] and Efficient-NeRF [HLC∗ 22] introduced multistage coarse-to-fine strategies using sparse grids or two-level octrees (NerfTree) to dynamically adapt voxelization to the underlying scene structure [S1, S2]. Later works incorporated VDB structures [YLMM23] to further optimize traversal and memory usage [S1].

## Sources
[S1] Recent Advances and Trends in Learning‐Based 3D Representations, https://doi.org/10.1111/cgf.70530, fetched 2026-10-26
[S2] Instant Neural Graphics Primitives with a Multiresolution ..., https://www.cs.jhu.edu/~misha/ReadingSeminar/Papers/Mueller22.pdf, fetched 2026-10-28
