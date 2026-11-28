# Specific Primitives and Techniques
## Overview
This family covers specialized voxelization algorithms and data structures that deviate from standard surface or solid filling, targeting specific geometric primitives (lines, porous structures) or optimization constraints (local regions, tubular approximations) [S1][S2, unofficial]. It distinguishes itself from the parent [Voxel Operations and Tests](11227-voxel-operations-and-tests.md) by focusing on non-cubic or non-uniform generation strategies, such as parametric line decomposition, hash-based porosity filtering, localized spatial queries, and piecewise boxing, rather than uniform grid filling of closed meshes [S1][S2, unofficial].

## Shared principles
The methods in this family share the goal of mapping continuous geometric or logical entities into discrete voxel representations, but they diverge in how the domain is sampled or filtered [S1][S2, unofficial].

## Subtopics

### [Porous Voxelization](4361-porous-voxelization.md)
A technique that generates sponge-like meshes by selectively discarding voxels during the voxelization process based on a spatial hash of their coordinates. Unlike standard voxelization which retains all computed voxels, this method creates a porous structure while maintaining spatial distribution properties.
**When to choose**: Use when a lightweight, non-watertight, or porous representation is required for rendering or simulation, and exact surface continuity is less critical than volumetric density distribution.
**Link**: [Porous Voxelization](4361-porous-voxelization.md)

### [Crosshairs Intersection Target Method](4608-crosshairs-intersection-target-method.md)
Not given in the sources.
**When to choose**: Not given in the sources.
**Link**: [Crosshairs Intersection Target Method](4608-crosshairs-intersection-target-method.md)

### [Parametric Line Segment Voxelization](17103-parametric-line-segment-voxelization.md)
An algorithm for generating a 3D line segment primitive by computing points along the segment using a parameter $t$, then mapping these points to voxels. It is distinguished from sibling methods like the seven-neighbor search by its independence of voxel calculations; each voxel's position is computed directly from the segment parameters, making it highly parallelizable on GPGPU architectures.
**When to choose**: Use for rendering or processing thin, linear features (e.g., wires, fibers) where standard grid-based voxelization might produce jagged or disconnected artifacts due to aliasing.
**Link**: [Parametric Line Segment Voxelization](17103-parametric-line-segment-voxelization.md)

### Local Voxelization
Voxelize a mesh in the region of a cube around a specific point, rather than globally [S1]. Useful to reduce memory cost for small values of pitch as opposed to global voxelization [S1].
**When to choose**: Use when only a small portion of a large mesh needs to be voxelized, or when memory is constrained and global voxelization is infeasible [S1].
**Link**: Not given in the sources.
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def local_voxelize(mesh, point, pitch, radius, fill=True, **kwargs):
    """Voxelize a mesh in the region of a cube around a point.
    Returns a VoxelGrid instance or None if the volume is empty [S1]."""
    # Implementation details are provided by the trimesh library [S1].
    pass
```

### Piecewise Boxing for Tubular Structures
A method for hardware volume rendering of tubular colon structures using a series of piecewise boxes with intelligent visibility determination [S2, unofficial]. Portals are the darkened segments indicating the connections or boundaries between the boxes in the piecewise boxing sequence [S2, unofficial].
**When to choose**: Use for hardware volume rendering of tubular structures where subvolume rendering constraints (e.g., 32-voxel boundary) must be met [S2, unofficial].
**Link**: Not given in the sources.

## Background
Local voxelization originates from the `trimesh` library (version 5.1.1/5.2.0), which provides a Pythonic interface for 3D mesh processing [S1]. Piecewise boxing for tubular structures originates from research on real-time volume rendering and virtual colonoscopy [S2, unofficial]. Parametric line segment voxelization and porous voxelization are documented in unofficial notes related to GPGPU optimization and mesh processing libraries [G]. These methods address specific gaps in general-purpose voxelization, such as handling non-closed surfaces, optimizing for sparse data, or accelerating linear primitive rendering [G].

## Sources
[S1] trimesh.voxel.creation - trimesh 5.1.1 documentation, https://trimesh.org/trimesh.voxel.creation.html, fetched 2026-10-26
[S2] Real-Time Volume Rendering, https://cfcs.pku.edu.cn/baoquan/docs/20180622110335231616.pdf, fetched 2026-10-30 (unofficial)
