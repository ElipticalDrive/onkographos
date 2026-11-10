# Marching Cubes Surface Point Extraction

Also called: QEF Surface Point, Blocky Surface Point, Centroid Surface Point

## Overview
This method extracts a single 3D position representing the isosurface intersection for each voxel in a grid, serving as the geometric foundation for point-based rendering or further mesh generation. Unlike its parent [Marching Cubes Vertex List Generation](4447-marching-cubes-vertex-list-generation.md), which generates a list of vertices for polygonal connectivity, this method aggregates intersections into a single point per voxel using one of three strategies: Quadratic Error Function (QEF), Centroid, or Blocky (cube center). It solves the problem of determining a representative surface location when multiple edges intersect a voxel, prioritizing computational efficiency on the GPU over topological correctness.

## How it works
The method operates as a compute shader kernel (`SurfacePoints`) dispatched with one thread per voxel. It identifies edges of the voxel cube that cross the isosurface (defined by value 0) and computes a surface point based on the selected strategy.

### Inputs and Outputs
*   **Input**: `_IsoVals`: A structured buffer containing scalar field values for each voxel, indexed by `threeDToOneD(id)`.
*   **Input**: `_Resolution`: The dimension of the voxel grid (assumed cubic, $N \times N \times N$).
*   **Input**: `_Step`: The physical size of a voxel edge.
*   **Input**: `_GridToWorld`: A $4 \times 4$ transformation matrix mapping grid coordinates to world space.
*   **Output**: `_SurfacePoints`: A `RWStructuredBuffer<float4>` storing the extracted surface points. The w-component is 1.0 if a valid point is found, 0.0 otherwise.
*   **Output**: `_Normals`: A `RWStructuredBuffer<float4>` storing normals (implementation detail noted in sources, but calculation logic for normals is not fully detailed in the provided text, only usage in QEF).

### Data Structures
*   **Edge**: A struct containing `start` and `end` fields, representing the grid coordinates of the edge endpoints.
*   **Cube Edges**: An array `cube_edges[12]` storing the 12 edges of the current voxel.

### Coordinate Systems and Helpers
The method uses integer grid coordinates `id` (uint3) and converts them to 1D indices for buffer access.

$$ \text{threeDToOneD}(id) = id.x + \text{Resolution} \times (id.y + \text{Resolution} \times id.z) $$

$$ \text{oneDToThreeD}(i) = \text{uint3}(i \% \text{Resolution}, (i / \text{Resolution}) \% \text{Resolution}, i / (\text{Resolution} \times \text{Resolution})) $$

The physical position of a grid point `p` is calculated as:
$$ \text{transformPoint}(p) = (\text{p} - 0.5) \times \text{Step} + \text{Step} \times 0.5 $$
*[G] This formula shifts the grid origin to the center of the voxel grid and scales by step size, assuming the grid is centered at the origin in world space.*

Linear interpolation between two points `p1`, `p2` with values `d1`, `d2` to find the zero-crossing:
$$ \text{linInterpolate}(p1, p2, d1, d2) = (1 - \frac{d1}{d1 - d2}) p1 + \frac{d1}{d1 - d2} p2 $$

### Algorithm Steps

1.  **Initialization**:
    *   Get thread ID `id` representing the voxel position.
    *   If `id.x >= _Resolution` or `id.y >= _Resolution` or `id.z >= _Resolution`, return (out of bounds).
    *   Initialize `edges_on_surface` list.
    *   Define the 12 edges of the unit cube rooted at `pos = id`. For each axis $i \in \{0,1,2\}$, there are 4 edges parallel to that axis. The start of an edge is determined by `pos` and the strides of the other two axes. The end is `start + stride_i`.

2.  **Edge Intersection Test**:
    *   Iterate through all 12 edges.
    *   For each edge, get the scalar values at `start` and `end` from `_IsoVals`.
    *   If the product of the values is less than 0 (opposite signs), the edge intersects the isosurface. Add the edge to `edges_on_surface`.

3.  **Surface Point Computation**:
    *   If `edges_on_surface` is empty, the voxel is fully inside or outside. Set the surface point to the transformed center of the voxel:
        $$ \text{surface\_point} = \text{transformToCenter}(pos) $$
        where $\text{transformToCenter}(pos) = \text{transformPoint}(pos + 0.5)$.
    *   If intersections exist, select one of three strategies based on preprocessor defines:

        #### Strategy A: QEF (Quadratic Error Function)
        *   Initialize accumulators: `ATA` (3x3 matrix, zeros), `Atb` (float4, zeros), `pointaccum` (float4, zeros), `btb` (float, 0).
        *   For each intersected edge $i$:
            1.  Calculate the linear interpolation crossing point in grid space:
                $$ \text{edge\_crossing} = \text{linInterpolate}(\text{end}, \text{start}, \text{IsoVal}(\text{end}), \text{IsoVal}(\text{start})) $$
            2.  Transform to world space: $P = \text{transformPoint}(\text{edge\_crossing})$.
            3.  Estimate the normal at $P$: $N = \text{EstimateNormal}(P)$. *[G] EstimateNormal typically computes the gradient of the scalar field or uses a finite difference approximation.*
            4.  Add contribution to QEF system:
                $$ \text{qef\_add}(N, P, \text{ATA}, \text{Atb}, \text{pointaccum}, \text{btb}) $$
                This updates the system $A^T A x = A^T b$ where each plane is defined by normal $N$ and point $P$.
        *   Solve the QEF system to get `surface_point_`.
        *   Calculate the center of mass:
            $$ \text{com} = \frac{\text{pointaccum}.xyz}{\text{pointaccum}.w} $$
        *   **Clamp Check**: If `surface_point_` is outside the cube bounds defined by $\text{Min} = \text{pos}$ and $\text{Max} = \text{pos} + (\text{strides}[0] + \text{strides}[1] + \text{strides}[2])$, fallback to `com`.

        #### Strategy B: Centroid
        *   Initialize `surface_point = float3(0,0,0)` and `count = 0`.
        *   For each intersected edge $i$:
            1.  Calculate the linear interpolation crossing point in grid space.
            2.  Transform to world space.
            3.  Add to `surface_point`.
            4.  Increment `count`.
        *   Divide `surface_point` by `count` (broadcasted as float3) to get the average position.

        #### Strategy C: Blocky
        *   Return the cube center:
            $$ \text{surface\_point} = \text{pos} + \frac{\text{strides}[0] + \text{strides}[1] + \text{strides}[2]}{2} $$
            Transformed to world space.

4.  **Output**:
    *   Store the resulting `val` in `_SurfacePoints` at index `threeDToOneD(id)` as `float4(val, 1.0)`. If no valid point was found (e.g., empty edges list handled in step 3), store `float4(transformToCenter(id), 0.0)`.

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def SurfacePoints(id, Resolution, Step, GridToWorld, IsoVals):
    # Boundary check
    if id.x >= Resolution or id.y >= Resolution or id.z >= Resolution:
        return None

    # Helper: Convert grid pos to world pos
    def transformPoint(p):
        # p is grid coordinate (float3)
        # Shift to origin, scale by step, shift back
        return (p - 0.5) * Step + Step * 0.5

    # Helper: Linear interpolation for zero crossing
    def linInterpolate(p1, p2, d1, d2):
        if d1 == d2:
            return p1 # Avoid division by zero
        t = d1 / (d1 - d2)
        return (1 - t) * p1 + t * p2

    # Define 12 edges of the cube at id
    # Edges are defined by start and end grid coordinates
    edges = []
    strides = [Step, Step, Step] # Assuming uniform grid for simplicity, or use actual strides
    
    # Generate edges based on axes
    # Axis 0 (x): 4 edges
    for y in [0, 1]:
        for z in [0, 1]:
            start = id + float3(0, y, z)
            end = start + float3(1, 0, 0)
            edges.append((start, end))
    # Axis 1 (y): 4 edges
    for x in [0, 1]:
        for z in [0, 1]:
            start = id + float3(x, 0, z)
            end = start + float3(0, 1, 0)
            edges.append((start, end))
    # Axis 2 (z): 4 edges
    for x in [0, 1]:
        for y in [0, 1]:
            start = id + float3(x, y, 0)
            end = start + float3(0, 0, 1)
            edges.append((start, end))

    edges_on_surface = []
    for start, end in edges:
        d1 = IsoVals[threeDToOneD(start)]
        d2 = IsoVals[threeDToOneD(end)]
        if d1 * d2 < 0:
            edges_on_surface.append((start, end))

    if len(edges_on_surface) == 0:
        # Voxel fully inside or outside, use center
        center_grid = id + float3(0.5, 0.5, 0.5)
        surface_point = transformPoint(center_grid)
        return (surface_point, 0.0) # w=0 indicates invalid/empty

    # Select strategy
    if SURFACE_POINT_QEF:
        surface_point = GetSurfacePointQEF(edges_on_surface, id, Step, transformPoint)
    elif SURFACE_POINT_CENTROID:
        surface_point = GetSurfacePointCentroid(edges_on_surface, transformPoint)
    elif SURFACE_POINT_BLOCKY:
        surface_point = GetSurfacePointBlocky(id, Step)
    
    return (surface_point, 1.0)

def GetSurfacePointQEF(edges, pos, Step, transformPoint):
    ATA = zeros((3, 3))
    Atb = zeros(4)
    pointaccum = zeros(4)
    btb = 0.0
    
    for start, end in edges:
        d1 = IsoVals[threeDToOneD(start)]
        d2 = IsoVals[threeDToOneD(end)]
        
        # Linear interpolation in grid space
        edge_crossing_grid = linInterpolate(start, end, d1, d2)
        
        # Transform to world space
        P = transformPoint(edge_crossing_grid)
        
        # Estimate normal
        N = EstimateNormal(P) # [G] Implementation of EstimateNormal not provided in source
        
        # QEF Add
        # Plane equation: N . (x - P) = 0 => N.x - N.P = 0
        # A = N, b = N.P
        # ATA += A^T A
        # Atb += A^T b
        # pointaccum += [A, b]
        
        ATA += outer_product(N, N)
        Atb += N * dot(N, P)
        pointaccum += float4(N[0], N[1], N[2], 1.0) # [G] Assuming pointaccum accumulates homogeneous coords for COM
        btb += dot(N, P) ** 2

    # Solve QEF: ATA x = Atb
    # [G] Standard linear solver (e.g., Cholesky or Gaussian elimination)
    surface_point_ = solve_linear_system(ATA, Atb)
    
    # Center of mass fallback
    com = pointaccum[0:3] / pointaccum[3]
    
    # Clamp check
    min_bound = pos
    max_bound = pos + float3(1, 1, 1) # Assuming unit grid steps for bounds check
    if not is_inside_cube(surface_point_, min_bound, max_bound):
        return com
        
    return surface_point_

def GetSurfacePointCentroid(edges, transformPoint):
    sum_pos = zeros(3)
    count = 0
    for start, end in edges:
        d1 = IsoVals[threeDToOneD(start)]
        d2 = IsoVals[threeDToOneD(end)]
        edge_crossing_grid = linInterpolate(start, end, d1, d2)
        P = transformPoint(edge_crossing_grid)
        sum_pos += P
        count += 1
    if count == 0:
        return zeros(3)
    return sum_pos / count

def GetSurfacePointBlocky(pos, Step):
    center_grid = pos + float3(0.5, 0.5, 0.5)
    return transformPoint(center_grid)
```

## Subtopics

### [Point Engine for Scalar Fields](7529-point-engine-for-scalar-fields.md)
A method to estimate foot points on an isosurface of a scalar field using a simplified version of Marching Cubes that finds only vertices without computing triangles, avoiding the expense of iterative refinement. It sets itself apart from its parent [Marching Cubes Surface Point Extraction](6417-marching-cubes-surface-point-extraction.md) by strictly limiting the search to three specific edges per voxel and storing all resulting points in a single flat list, thereby eliminating the need for triangle connectivity, table lookups, or shared vertex bookkeeping.

## Operating characteristics
*   **Time Complexity**: $O(1)$ per voxel, as it processes a fixed number of edges (12) and performs a constant number of arithmetic operations. The QEF strategy involves solving a $3 \times 3$ linear system, which is also constant time but more expensive than Centroid or Blocky.
*   **Memory Complexity**: $O(N^3)$ for the output buffer `_SurfacePoints`, where $N$ is the resolution.
*   **Performance**: The method is highly parallelizable and suitable for GPU compute shaders. The Blocky strategy is the fastest, followed by Centroid, then QEF.
*   **Strengths**: Simple to implement, low memory overhead per voxel, no need for complex data structures like linked lists or hash maps for vertex sharing.
*   **Limitations**: The extracted points do not form a manifold mesh directly. The QEF strategy may produce points outside the voxel if not clamped. The method does not preserve topology.
*   **When to use**: Point-based rendering, initial guess for iterative refinement, or when topological accuracy is not required.
*   **When not to use**: When a watertight mesh is needed directly from the extraction step.

## Background
The method is derived from the Marching Cubes algorithm, which was originally introduced by Lorensen and Cline in 1987. The specific implementation described here, including the QEF, Centroid, and Blocky strategies, is found in the unofficial repository `vinceniko/Isosurface` [S1, unofficial]. It adapts the classic surface extraction concept to a point-based representation, likely for use in real-time graphics applications where point sprites are rendered.

## Sources
[S1] vinceniko/Isosurface, https://github.com/vinceniko/Isosurface, fetched 2026-10-27 (unofficial)
