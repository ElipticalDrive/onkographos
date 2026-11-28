# Marching Cubes Lookup and Configuration


## Overview
Marching Cubes Lookup and Configuration provides the static data structures and mapping logic that determine how a voxel's scalar field sign configuration is converted into mesh geometry. It solves the combinatorial problem of isosurface extraction by replacing runtime conditional branching with direct array indexing, enabling $O(1)$ determination of triangle topology per voxel. It differs from symmetry-based approaches by handling all 256 cases explicitly (or via symmetry reduction) and from dynamic ambiguity resolution methods by using fixed triangulations for ambiguous cases, which may produce topological artifacts if adjacent cells resolve ambiguities inconsistently.

## Shared principles
The family relies on two precomputed data structures derived from the 256 possible binary configurations of a voxel's eight corner signs relative to an isovalue.

1.  **Edge Intersection Table (`edgeTable`)**: An array of 256 integers (or shorts). Each entry is a 12-bit bitmask. Bit $k$ ($0 \le k \le 11$) is set to 1 if the isosurface intersects the $k$-th edge of the cube, and 0 otherwise. The standard edge numbering convention is:
    *   Bottom face ($z=0$): $e_0=(0,1), e_1=(1,2), e_2=(2,3), e_3=(3,0)$
    *   Top face ($z=1$): $e_4=(4,5), e_5=(5,6), e_6=(6,7), e_7=(7,4)$
    *   Vertical edges: $e_8=(0,4), e_9=(1,5), e_{10}=(2,6), e_{11}=(3,7)$ [S4, S12, unofficial].

2.  **Triangle Connection Table (`triangleTable`)**: A 2D array of size $256 \times 16$ [S8, S12, unofficial].
    *   Rows are indexed by the case index $C \in [0, 255]$.
    *   Columns store integer indices. Indices $0–11$ refer to the 12 cube edges.
    *   Indices are grouped in triplets $(a, b, c)$ to form triangles.
    *   The sequence is terminated by the value `-1` [S8, S12, unofficial].
    *   The maximum number of triangles per voxel is 5, requiring 15 indices plus 1 terminator [S8, S12, unofficial].

**Case Index Calculation**: For each voxel, the 8-bit case index $C$ is determined by the scalar field values $V$ at the 8 vertices $v_i$ ($i=0..7$). If $V(v_i) < I$ (or $> I$, depending on convention), bit $i$ of $C$ is set to 1; otherwise 0 [S7, S8, S12, unofficial]. The bit assignment must strictly match the vertex ordering assumed by the LUT generation [S12, unofficial].

**Ambiguity**: Cases 46, 47, 98, 99, 210, 213, 214, 215 contain saddle points where the topology is ambiguous (the isosurface passes through the cube center) [S7, S8, unofficial]. The standard LUT uses a fixed triangulation for these, which can cause cracks or holes if adjacent voxels resolve the ambiguity differently [S7, S8, unofficial].

## Subtopics

### [Lookup Table and Configuration](4978-lookup-table-and-configuration.md)
Defines the specific bitmasks and table layouts for edge intersection and triangle connectivity.

### [Topology and Ambiguity Handling](6547-topology-and-ambiguity-handling.md)
Addresses non-manifold vertices and edges in ambiguous cases by dynamically resolving connectivity, distinguishing it from static lookup table approaches.

### [Transvoxel](6277-transvoxel.md)
Extends the configuration to handle hierarchical LOD transitions, ensuring watertight meshes across resolution boundaries.

### [Marching Cubes Triangulation](3682-marching-cubes-triangulation.md)
The geometric assembly step that uses the lookup configuration to generate vertex positions and normals from intersection data.

### [Marching Cubes Vertex List Generation](4447-marching-cubes-vertex-list-generation.md)
Calculates precise 3D positions of vertices on intersected edges using linear interpolation, serving as the geometric foundation for triangulation.

### [GPU Acceleration and Parallel Pipelines](4136-gpu-acceleration-and-parallel-pipelines.md)
Offloads the lookup and interpolation to the GPU, managing dynamic output sizes via atomic counters or histogram pyramids.

### [Marching Cubes Job Scheduling](4446-marching-cubes-job-scheduling.md)
Orchestrates parallel execution by decomposing the volume into chunks and scheduling compute jobs for each.

### [CPU Immediate Mode Marching Cubes](2463-cpu-immediate-mode-marching-cubes.md)
A reference CPU implementation that renders triangles immediately without persistent mesh buffers.

### [SDF to PLY Conversion](2343-sdf-to-ply-conversion.md)
Converts a sampled Signed Distance Field into a PLY mesh file using the Marching Cubes extraction pipeline.

## Background
The Marching Cubes algorithm was introduced by William E. Lorensen and Harvey E. Cline in 1987 [S7, S8]. It became the standard method for isosurface extraction due to its efficiency and simplicity. The lookup table approach was formalized to avoid runtime branching, allowing for fast hardware and software implementations. Subsequent research has focused on resolving topological ambiguities [S7, S8] and extending the method to hierarchical grids (Transvoxel) [S1, unofficial] and parallel architectures [S1, 1211-primal-grid-marching-cubes, unofficial].

## Sources
[S1] Transvoxel implementation notes, unofficial
[S2] Marching Cubes Triangulation notes, unofficial
[S4] Edge numbering convention, unofficial
[S7] Lorensen and Cline, "Marching Cubes: A High Resolution 3D Surface Construction Algorithm", 1987
[S8] Standard Marching Cubes Lookup Table definitions, unofficial
[S12] Edge and Vertex indexing conventions, unofficial
