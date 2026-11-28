# Vertex Normal Averaging for Shading

Also called: Face Normal Computation (shading-based), Position Normals, Shading Normal Computation, Surface Normal Averaging for Shading

## Overview
Vertex Normal Averaging for Shading computes smooth surface normals for a mesh by averaging the geometric normals of all adjacent triangles sharing a vertex, then normalizing the result [S2, unofficial]. It reduces the faceting artifact inherent in flat shading by providing interpolated normals for rendering [S1]. Unlike its sibling [Face Normal Computation](2042-face-normal-computation.md), which computes a single normal per face, this method produces per-vertex normals that vary across the surface, enabling smooth lighting transitions [S2, unofficial].

## How it works
**Inputs:** A list of triangles, where each triangle is defined by three vertex indices [S2, unofficial]. Each triangle must have a precomputed geometric face normal [S2, unofficial].
**Outputs:** A list of vertex normals, one for each unique vertex in the mesh [S2, unofficial].

**Data Structures:**
1. **Vertex Accumulator:** A list or array of structures, one per vertex [S2, unofficial]. Each structure contains a 3D vector for the accumulated normal and an integer for the face count [S2, unofficial].
2. **Triangle List:** An array of triangles [S2, unofficial]. Each triangle stores indices to its three vertices and its precomputed face normal vector [S2, unofficial].

**Steps:**
1. **Initialization:** Create the vertex accumulator list with size equal to the number of unique vertices [S2, unofficial]. Set all normal vectors to zero and all face count integers to zero before accumulation begins [S2, unofficial].
2. **Accumulation:** Iterate through every triangle in the mesh [S2, unofficial]. For each triangle with face normal $\mathbf{n}_f$ and vertex indices $i_0, i_1, i_2$: Add $\mathbf{n}_f$ to `accumulator[i_0].normal` [S2, unofficial]. Add $\mathbf{n}_f$ to `accumulator[i_1].normal` [S2, unofficial]. Add $\mathbf{n}_f$ to `accumulator[i_2].normal` [S2, unofficial]. Increment `accumulator[i_0].faceCount` by 1 [S2, unofficial]. Increment `accumulator[i_1].faceCount` by 1 [S2, unofficial]. Increment `accumulator[i_2].faceCount` by 1 [S2, unofficial].
3. **Normalization:** Iterate through every vertex in the mesh [S2, unofficial]. Let $\mathbf{S}$ be `accumulator[v].normal` and $C$ be `accumulator[v].faceCount` [S2, unofficial]. If $C > 0$, compute the averaged normal $\mathbf{n}_v = \frac{\mathbf{S}}{C}$ [S2, unofficial]. Normalize $\mathbf{n}_v$ to unit length: $\mathbf{n}_v \leftarrow \frac{\mathbf{n}_v}{\|\mathbf{n}_v\|}$ [S2, unofficial]. Store $\mathbf{n}_v$ as the final vertex normal [S1][S2, unofficial].
4. **Usage:** Pass the computed vertex normals to the underlying rendering function for use in shading the triangles [S1].

**Formulas:**
The averaged vertex normal is computed as:
$$\mathbf{n}_v = \frac{\sum_{t \in \text{adj}(v)} \mathbf{n}_t}{\left\| \sum_{t \in \text{adj}(v)} \mathbf{n}_t \right\|}$$
where $\mathbf{n}_t$ is the face normal of triangle $t$, and $\text{adj}(v)$ is the set of triangles sharing vertex $v$ [S2, unofficial]. If face normals are not precomputed, they can be derived from edge vectors $\mathbf{e}_{t1}, \mathbf{e}_{t2}$ via cross product before accumulation: $\mathbf{n}_t = \mathbf{e}_{t1} \times \mathbf{e}_{t2}$ [S4, unofficial].

**Parameters:**
* **Weighting:** The basic method uses equal weighting for all adjacent faces [S2, unofficial]. Sources do not specify area-weighted or angle-weighted variants [S2, unofficial].

**Edge Cases and Pitfalls:**
* **Unindexed Meshes:** If the mesh is stored as an unindexed, unordered list of vertices, determining adjacency requires iterating through all triangles for each vertex, which is computationally expensive [S2, unofficial]. An index buffer is required for efficiency [S2, unofficial].
* **Performance:** Computing and storing vertex normals increases computing time and memory usage compared to flat shading [S1].
* **Initialization:** Failure to reset accumulators to zero before a new computation leads to incorrect results [S2, unofficial].
* **Degenerate Vertices:** Isolated vertices with zero adjacent faces must be handled to avoid division by zero [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_vertex_normals(triangles, num_vertices):
    # Step 1: Initialization
    accumulators = [{'normal': [0.0, 0.0, 0.0], 'faceCount': 0} for _ in range(num_vertices)]

    # Step 2: Accumulation
    for tri in triangles:
        n = tri['normal']
        i0, i1, i2 = tri['indices']
        accumulators[i0]['normal'][0] += n[0]
        accumulators[i0]['normal'][1] += n[1]
        accumulators[i0]['normal'][2] += n[2]
        accumulators[i0]['faceCount'] += 1
        accumulators[i1]['normal'][0] += n[0]
        accumulators[i1]['normal'][1] += n[1]
        accumulators[i1]['normal'][2] += n[2]
        accumulators[i1]['faceCount'] += 1
        accumulators[i2]['normal'][0] += n[0]
        accumulators[i2]['normal'][1] += n[1]
        accumulators[i2]['normal'][2] += n[2]
        accumulators[i2]['faceCount'] += 1

    # Step 3: Normalization
    vertex_normals = []
    for i in range(num_vertices):
        acc = accumulators[i]
        if acc['faceCount'] > 0:
            avg = [acc['normal'][0] / acc['faceCount'], acc['normal'][1] / acc['faceCount'], acc['normal'][2] / acc['faceCount']]
            length = (avg[0]**2 + avg[1]**2 + avg[2]**2)**0.5
            if length > 0:
                vertex_normals.append([avg[0]/length, avg[1]/length, avg[2]/length])
            else:
                vertex_normals.append([0.0, 1.0, 0.0]) # [G] Default up vector for degenerate cases
        else:
            vertex_normals.append([0.0, 1.0, 0.0]) # [G] Default up vector for isolated vertices
    return vertex_normals
```

## Subtopics

### Camera-Aligned Vertex Normal Fixing
For each camera position, compute direction vectors from vertices to camera and normalize [S6, unofficial]. Compute dot product between vertex normals and directions to camera [S6, unofficial]. Normalize the accumulated new_normals and create a new Meshes object with the updated normals [S6, unofficial].

### Vertex Normal Assembly for Rendering
A two-step process to compute spatially varying, continuous surface normals for smooth shading by assembling local nodal normals into global vectors [S7, unofficial]. Compute local nodal normals for all active surfaces [S7, unofficial]. Assemble global normals: Scatter local normals to global nodes using the polygon vertex list $P_{ij}$, accumulating the number of contributing polygons [S7, unofficial]. Compute average normals: Divide the accumulated nodal normal vector by the accumulated count of contributing polygons [S7, unofficial]. Handle sharp corners: Use a heuristic based on the dot product of the vertex normal and the globally averaged normal to decide whether to use the averaged normal or the local normal [S7, unofficial].

## Operating characteristics
* **Time Complexity:** $O(F + V)$, where $F$ is the number of faces and $V$ is the number of vertices [S2, unofficial].
* **Memory Complexity:** $O(V)$ for the accumulator structures [S2, unofficial].
* **Strengths:** Produces smooth shading on curved surfaces and yields a more aesthetically pleasing surface at the expense of directly perceiving grid resolution [S5].
* **Limitations:** Increases computing time and memory usage [S1]. Does not account for surface curvature or area [S2, unofficial].
* **When to use:** When smooth shading is desired for a mesh with known topology [S5].
* **When not to use:** When flat shading is sufficient for the visual style or memory is constrained [S1].

## Background
The technique is described in the context of R's `rgl` package for rendering isosurfaces, where it is used to reduce faceting artifacts by passing averaged normals to the underlying rendering function [S1]. It is implemented in Data Explorer's Normals module as the default "positions" method, which creates position normals for smoother appearance [S5]. The method is a standard practice in computer graphics for Gouraud shading.

## Sources
[S1] Computing and Displaying Isosurfaces in R, https://doi.org/10.18637/jss.v028.i01, fetched 2026-10-27
[S2] Marching Cubes Vertex Normals, https://gamedev.stackexchange.com/questions/97707/marching-cubes-vertex-normals, fetched 2026-10-26 (unofficial)
[S3] prateekchandan/Advanced-CG-High-Resolution-Sparse-Voxel-DAGs, https://github.com/prateekchandan/Advanced-CG-High-Resolution-Sparse-Voxel-DAGs, fetched 2026-10-26 (unofficial)
[S4] vittoboa/procedural-terrain-generation, https://github.com/vittoboa/procedural-terrain-generation, fetched 2026-10-27 (unofficial)
[S5] User's Guide - Visualization Techniques, https://theochem.mercer.edu/dxdocs/pages/usrgu063.htm, fetched 2026-10-28
[S6] GAP-LAB-CUHK-SZ/gaustudio, https://github.com/GAP-LAB-CUHK-SZ/gaustudio, fetched 2026-10-26 (unofficial)
[S7] Visualization methods for high-resolution, transient, 3-D, finite element situations, https://digital.library.unt.edu/ark:/67531/metadc686501/?q=tensor field visualization volume, fetched 2026-10-29 (unofficial)
