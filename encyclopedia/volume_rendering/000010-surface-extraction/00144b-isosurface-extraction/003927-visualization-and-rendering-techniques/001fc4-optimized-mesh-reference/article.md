# Optimized Mesh Reference

Also called: Best-case indirect isosurface visualization, Forward mesh renderer reference

## Overview
The Optimized Mesh Reference is a theoretical performance baseline for indirect isosurface visualization, representing the best-case scenario where geometry extraction and occlusion culling are instantaneous [S1][S2, unofficial]. It measures the rendering time of a highly-optimized explicit mesh representation of the isosurface, excluding the costs of extraction and visibility determination [S1][S2, unofficial]. This method serves as an upper bound for forward mesh rendering efficiency, allowing researchers to assess how close a practical pipeline comes to the theoretical limit of rendering an optimized isosurface [S1][S2, unofficial].

## How it works
Inputs: scalar volume data and isovalue. Output: final rendered image frame time.

**Data Structures:**
- **Explicit Mesh:** A collection of vertices and indices [G].
- **Vertex:** Contains 3D position (float3) and normal vector (float3) [G].
- **Meshlet:** A subset of the mesh defined by a vertex buffer and an index buffer [G].
- **GPU Memory:** The optimized mesh data is written directly to GPU memory buffers [S1][S2, unofficial].

**Parameters:**
- **Maximum vertices per meshlet:** 64 [S1][S2, unofficial].
- **Optimization Library:** Kapoulkine's `meshoptimizer` C++ library [S1][S2, unofficial].

**Steps:**
1. Extract vertex positions and normals for the mesh representation with the same extraction approach used in the primary pipeline [S1][S2, unofficial].
2. Write the explicit geometry representation directly to GPU memory [S1][S2, unofficial].
3. Remove duplicate vertices from the extracted mesh [S1][S2, unofficial].
4. Divide the cleaned mesh into meshlets using the optimization library [S1][S2, unofficial].
5. Limit each meshlet to a maximum of 64 vertices [S1][S2, unofficial].
6. Render the optimized mesh using a forward mesh renderer without accounting for geometry extraction time [S1][S2, unofficial].
7. Render the optimized mesh using a forward mesh renderer without accounting for occlusion culling time [S1][S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def optimized_mesh_reference(volume_data, isovalue):
    # Step 1: Extract vertex positions and normals using the pipeline's standard extraction method
    raw_vertices = extract_isosurface_vertices(volume_data, isovalue)
    raw_normals = extract_isosurface_normals(volume_data, isovalue)
    
    # Step 2: Write the explicit geometry representation directly to GPU memory
    gpu_vertex_buffer = allocate_and_upload(raw_vertices)
    gpu_index_buffer = allocate_and_upload(raw_normals)
    
    # Step 3: Remove duplicate vertices from the extracted mesh
    unique_vertices, vertex_remap = remove_duplicate_vertices(raw_vertices)
    remapped_indices = apply_remap(gpu_index_buffer, vertex_remap)
    
    # Step 4 & 5: Divide the cleaned mesh into meshlets, limiting each to max 64 vertices
    meshlets = cluster_meshlets(unique_vertices, remapped_indices, max_vertices=64)
    
    # Step 6 & 7: Render the optimized mesh using a forward mesh renderer
    # Do not account for geometry extraction time.
    # Do not account for occlusion culling time.
    frame_time = 0.0
    for meshlet in meshlets:
        frame_time += forward_render(meshlet)
        
    return frame_time

def extract_isosurface_vertices(volume, iso):
    # Standard isosurface extraction algorithm (e.g., marching cubes)
    return [v for v in compute_surface(volume, iso)]

def extract_isosurface_normals(volume, iso):
    # Compute normals for extracted surface
    return [compute_normal(v) for v in compute_surface(volume, iso)]

def remove_duplicate_vertices(vertices):
    # Hash-based deduplication to merge identical vertices
    unique = {}
    remap = []
    for i, v in enumerate(vertices):
        key = hash(v)
        if key not in unique:
            unique[key] = len(unique)
            remap.append(len(unique) - 1)
        else:
            remap.append(unique[key])
    return list(unique.values()), remap

def cluster_meshlets(vertices, indices, max_vertices):
    # Cluster vertices into meshlets respecting the maximum vertex count
    meshlets = []
    current = []
    for idx in indices:
        if len(current) >= max_vertices:
            meshlets.append(current)
            current = []
        current.append(idx)
    if current:
        meshlets.append(current)
    return meshlets

def forward_render(meshlet):
    # Rasterize meshlet using forward rendering pipeline
    return 0.001
```

## Operating characteristics
- **Performance:** The Optimized Mesh Reference consistently outperforms practical indirect rendering approaches. In evaluations, it was faster by factors of 1.61 (CT Head), 2.03 (Vertebra), and 1.85 (Chameleon) [S1][S2, unofficial]. For time-varying data (Supernova), it was 2.1 times faster [S1][S2, unofficial].
- **Complexity:** The method assumes $O(1)$ extraction and culling time, which is physically impossible but serves as a theoretical lower bound for rendering latency [S1][S2, unofficial].
- **Strengths:** Provides a clear target for optimization; isolates the cost of rasterization and shading from the cost of geometry generation [S1][S2, unofficial].
- **Limitations:** Does not reflect real-world performance because it ignores the significant computational cost of extracting the isosurface and determining visibility (occlusion culling) [S1][S2, unofficial].

## Background
The Optimized Mesh Reference was introduced in the context of evaluating efficient direct isosurface rasterization techniques [S1][S2, unofficial]. It was proposed to provide a fair comparison between direct rasterization, ray-marching, and indirect extraction-based approaches [S1][S2, unofficial]. The method relies on the `meshoptimizer` library by Kapoulkine [S1][S2, unofficial] and follows recommendations by Kubisch regarding meshlet sizes [S1][S2, unofficial]. It is used as a benchmark to demonstrate that practical pipelines can approach the efficiency of offline-optimized rendering when extraction overheads are excluded [S1][S2, unofficial].

## Subtopics

### Triangular Mesh Scene Rendering
* **What it is:** Rendering scenes composed of pre-existing triangular meshes.
* **Difference:** Does not involve isosurface extraction from scalar data.
* **When to choose:** When the geometry is already available as a mesh.
* **Link:** [Triangular Mesh Scene Rendering](4644-triangular-mesh-scene-rendering.md)

### Isosurface Rendering
* **What it is:** General techniques for visualizing isosurfaces, including ray-marching and extraction-based methods.
* **Difference:** Includes the cost of extraction and culling; Optimized Mesh Reference is a specific baseline within this family.
* **When to choose:** When visualizing scalar field data.
* **Link:** [Isosurface Rendering](7669-isosurface-rendering.md)

### Multi-Isosurface Rendering
* **What it is:** Rendering multiple isosurfaces simultaneously from a single volume.
* **Difference:** Handles multiple thresholds and potential blending/occlusion between surfaces.
* **When to choose:** When multiple material boundaries need to be visualized.
* **Link:** [Multi-Isosurface Rendering](8745-multi-isosurface-rendering.md)

## Sources
[S1] Efficient Direct Isosurface Rasterization of Scalar Volumes, https://doi.org/10.1111/cgf.14670, fetched 2026-10-27
[S2] Efficient Direct Isosurface Rasterization of Scalar Volumes - EG, https://diglib.eg.org/items/8a65e0f0-c048-4fce-9d0b-5acf07452485, fetched 2026-10-30 (unofficial)
