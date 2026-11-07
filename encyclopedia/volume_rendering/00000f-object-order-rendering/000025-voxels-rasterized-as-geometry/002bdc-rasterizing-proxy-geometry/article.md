# Rasterizing Proxy Geometry
Also called: Liu et al. proxy rasterization

## Overview
Rasterizing proxy geometry solves the problem of efficient visibility determination and empty-space skipping in sparse volume rendering by converting hierarchical data structures into geometric primitives for the graphics pipeline [S1, unofficial]. It differs from direct ray-casting or tree traversal methods by leveraging hardware rasterization to identify visible regions, thereby reducing the number of look-ups into empty-space skipping data structures [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A hierarchical sparse volume representation (e.g., an octree) where nodes are classified as empty, non-empty, or unknown.
*   **Output:** A rasterized image or visibility buffer indicating which voxels or regions are visible, enabling optimized ray casting or shading.

**Data Structures:**
The method relies on generating "proxy geometry" for octree nodes. Each node in the hierarchy is represented by a geometric primitive (typically a cube or bounding box) that approximates its spatial extent. The classification of the node (empty, non-empty, unknown) determines whether the proxy is generated or culled [S1, unofficial].

**Steps:**
1.  **Hierarchy Traversal and Classification:** Traverse the sparse volume hierarchy (e.g., octree). Classify each node based on its content:
    *   *Empty:* No data; no proxy generated.
    *   *Non-empty:* Contains data; proxy geometry is generated.
    *   *Unknown:* Status not yet determined; proxy may be generated conservatively or deferred [S1, unofficial].
2.  **Proxy Generation:** For each "non-empty" or "unknown" node, generate a proxy mesh (e.g., a cube) representing the node's bounding box in world space. This geometry serves as a view-dependent cut of the octree [S1, unofficial].
3.  **Rasterization:** Submit the proxy geometry to the graphics pipeline. The primitives are transformed into clip space and rasterized. The depth buffer (z-buffer) is used to determine visibility.
4.  **Visibility Determination:** The rasterized output identifies which parts of the volume are visible from the current camera view. This visibility information is used to guide subsequent ray casting, limiting look-ups into the empty-space skipping data structure only to visible regions [S1, unofficial].

**Formulas:**
Not given in the sources.

**Parameters:**
Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Unknown Nodes:** If a node's status is "unknown," generating a proxy may lead to over-rendering (processing empty space) or under-rendering (missing data) if not handled correctly. The method must define a strategy for these nodes [S1, unofficial].
*   **Precision:** Rasterization artifacts may occur if proxy geometry is not tightly bound to the actual voxel data, leading to incorrect visibility decisions.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def rasterize_proxy_geometry(octree, camera):
    # Step 1: Traverse and classify nodes
    proxy_meshes = []
    for node in octree.traverse():
        if node.classification == 'non-empty' or node.classification == 'unknown':
            # Step 2: Generate proxy geometry
            # Create a cube mesh representing the node's bounding box
            mesh = create_cube_mesh(node.bounding_box)
            proxy_meshes.append(mesh)
    
    # Step 3: Rasterize
    # Submit proxy_meshes to the graphics pipeline
    # Use camera's MVP matrix for transformation
    visibility_buffer = rasterize(proxy_meshes, camera)
    
    # Step 4: Use visibility for ray casting
    # Limit ray casting to visible regions
    return visibility_buffer
```

## Subtopics

### [Proxy Mesh Rendering for Page Visibility](8386-proxy-mesh-rendering-for-page-visibility.md)
Proxy Mesh Rendering for Page Visibility determines which virtual memory pages are visible in a scene by rendering a simplified proxy mesh to a visibility buffer and reducing that buffer to a list of active page IDs. It differs from standard visibility buffers by encoding depth information to support Level of Detail (LOD) selection and by utilizing a compute shader reduction step to minimize memory bandwidth, addressing the performance bottlenecks of atomic operations in concurrent thread updates.

## Operating characteristics
*   **Time Complexity:** Depends on the number of non-empty and unknown nodes in the hierarchy. Rasterization is generally faster than per-pixel tree traversal for sparse data.
*   **Memory Complexity:** Requires storage for the proxy geometry, which scales with the number of visible nodes.
*   **Strengths:** Reduces empty-space look-ups; leverages hardware rasterization for efficient visibility culling.
*   **Limitations:** Requires generating and managing proxy geometry; may over-render if "unknown" nodes are not handled efficiently.
*   **When to Use:** Sparse volume rendering where empty-space skipping is critical and the hierarchy is available on the GPU.
*   **When Not to Use:** Dense volumes where the overhead of proxy generation outweighs the benefits of culling.

## Background
The method was proposed by Liu et al. [LCDP13] as a way to optimize volume rendering by rasterizing proxy geometry representing a view-dependent cut of octree nodes [S1, unofficial]. It is part of a broader trend of using rasterization hardware to avoid hierarchy traversal on the GPU, as seen in methods like SparseLeap [S1, unofficial].

## Sources
[S1] Skalierbares Webbasiertes Volume Rendering großer biomedizinischer Datensätze, https://doi.org/10.34726/hss.2023.106203, fetched 2026-10-26 (unofficial)
