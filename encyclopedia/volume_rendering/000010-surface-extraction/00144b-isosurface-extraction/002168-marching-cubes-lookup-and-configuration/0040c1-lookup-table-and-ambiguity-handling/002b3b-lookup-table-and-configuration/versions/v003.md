# Lookup Table and Configuration

Also called: Marching Cubes Lookup Tables, MC LUTs, Case Index Tables

## Overview
This family defines the static data structures and configuration mechanisms used to map the 256 possible binary sign configurations of a voxel's eight corners to a specific polygonal topology. It solves the problem of efficiently determining which edges of a cube are intersected by an isosurface and how to connect those intersections into triangles without runtime geometric reasoning. What sets this family apart from siblings like [Snap Marching Cubes](2181-snap-marching-cubes.md) or [GPU Acceleration](4136-gpu-acceleration-and-parallel-pipelines.md) is its reliance on precomputed, static decision logic (lookup tables) rather than dynamic geometric queries or hardware-specific optimizations.

## Shared principles
The family relies on two precomputed data structures derived from the 256 possible binary configurations of a voxel's eight corner signs relative to an isovalue.

1.  **Edge Intersection Table (`edgeTable`)**: An array of 256 integers (or shorts). Each entry is a 12-bit bitmask. Bit $k$ ($0 \le k \le 11$) is set to 1 if the isosurface intersects the $k$-th edge of the cube, and 0 otherwise. The standard edge numbering convention is:
    *   Bottom face ($z=0$): $e_0=(0,1), e_1=(1,2), e_2=(2,3), e_3=(3,0)$
    *   Top face ($z=1$): $e_4=(4,5), e_5=(5,6), e_6=(6,7), e_7=(7,4)$
    *   Vertical edges: $e_8=(0,4), e_9=(1,5), e_{10}=(2,6), e_{11}=(3,7)$

2.  **Triangle Connection Table (`triangleTable`)**: A 2D array of size $256 \times 16$.
    *   Rows are indexed by the case index $C \in [0, 255]$.
    *   Columns store integer indices. Indices $0–11$ refer to the 12 cube edges.
    *   Indices are grouped in triplets $(a, b, c)$ to form triangles.
    *   The sequence is terminated by the value `-1`.
    *   The maximum number of triangles per voxel is 5, requiring 15 indices plus 1 terminator.

**Case Index Calculation**: For each voxel, the 8-bit case index $C$ is determined by the scalar field values $V$ at the 8 vertices $v_i$ ($i=0..7$). If $V(v_i) < I$ (or $> I$, depending on convention), bit $i$ of $C$ is set to 1; otherwise 0. The bit assignment must strictly match the vertex ordering assumed by the LUT generation.

**Ambiguity**: Cases 46, 47, 98, 99, 210, 213, 214, 215 contain saddle points where the topology is ambiguous (the isosurface passes through the cube center). The standard LUT uses a fixed triangulation for these, which can cause cracks or holes if adjacent voxels resolve the ambiguity differently.

## Subtopics

### [Symmetry Mapping](869-symmetry-mapping.md)
Reduces the 256 cases to 23 canonical base cases using rotation mappings to simplify logic.

### [Marching Cubes Lookup Table](4800-marching-cubes-lookup-table.md)
The standard static 256-entry table mapping case indices directly to triangle edge indices.

### [Watt & Watt Marching Cubes](7787-watt-watt-marching-cubes.md)
Optimizes the standard LUT with vertex deduplication and modified tables for topological consistency on shared faces.

### [Improved Marching Cubes Algorithm](9090-improved-marching-cubes-algorithm.md)
Expands the configuration set to 24 cases to resolve saddle point ambiguities and uses tunable interpolation.

### Patch Boundary Detection
*   **Definition**: A process to detect patch boundaries as groups of rings of linear contours on cell faces for automatic construction of look-up tables [S1].
*   **Steps**:
    1.  Construct a single ring using face-tracing: starting from a cell edge with a sign change, facing the positive end, look for the next edge with a sign change by turning counter-clockwise on the face boundary [S1].
    2.  Repeat the search process until it returns to the starting edge, forming a closed ring [S1].
    3.  Orient the linear contour on the cell face so that the negative region on that face lies to its left when looking from outside [S1].
    4.  Group rings constructed around the same edge-connected component of positive corners to form the boundary of a single patch [S1].

### Bloomenthal Polygonization
*   **Definition**: A fast polygonizer for algebraic surfaces based on a generalization of marching cubes [S2, unofficial].

## Background
The concept of using lookup tables for isosurface extraction originated with Lorensen and Cline's 1987 paper "Marching Cubes: A High Resolution 3D Surface Construction Algorithm" [S1, S6, S8, S9, unofficial]. They introduced the 256-case classification and the associated `edgeTable` and `triangleTable` to replace complex geometric branching with efficient array indexing. Subsequent research, such as Watt & Watt (1992), addressed topological inconsistencies in the original tables [S1]. Later variants, such as the Improved Marching Cubes algorithm, expanded the configuration set to handle ambiguous saddle points more robustly [S1][S2, unofficial]. Automatic construction methods for these tables, particularly for non-cubic cells or complex topologies, have also been developed [S1].

## Sources
[S1] Convex contouring of volumetric data, https://doi.org/10.1007/s00371-003-0216-0, fetched 2026-10-28
[S2] RAY TRACING IMPLICIT SURFACES FOR INTERACTIVE ..., https://www.sci.utah.edu/~knolla/thesis.pdf, fetched 2026-10-29 (unofficial)
