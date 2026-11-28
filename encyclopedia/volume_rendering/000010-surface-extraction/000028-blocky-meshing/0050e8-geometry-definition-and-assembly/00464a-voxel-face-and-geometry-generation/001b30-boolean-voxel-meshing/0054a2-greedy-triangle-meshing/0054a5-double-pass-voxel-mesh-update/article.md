# Double-Pass Voxel Mesh Update

Also called: CEWBS VoxelMesh update, transparent mesh pass

## Overview
This method updates a BabylonJS mesh by extracting geometry from a greedy voxel mesher and distributing it into two separate meshes: an opaque main mesh and a transparent secondary mesh. It solves the problem of rendering volumetric data with mixed opacity levels by separating faces based on their alpha values, ensuring correct depth sorting and blending for transparent surfaces while maintaining performance for opaque ones. It differs from single-pass meshing by explicitly handling transparency segregation during the geometry assembly phase.

## How it works
### Inputs and Outputs
**Inputs:**
*   `voxelData`: An object containing:
    *   `dimensions`: An array `[x, y, z]` representing the voxel grid size.
    *   `voxels`: A flat array of length $x \times y \times z$. Each element is either `null` or an array `[id, meta]`.
*   `evaluateFunction`: A function used by the greedy mesher to determine voxel occupancy or properties.
*   `coloringFunction`: A function `coloringFunction(faceIndex, meta)` that returns a color.
*   `passID`: An integer (0 or 1) indicating which mesh pass is being processed.
*   `rawMesh`: The output from the greedy mesher, containing `faces`. Each face `q` is an array `[v0, v1, v2, faceIndex, meta]` where `v0, v1, v2` are vertex indices.

**Outputs:**
*   Updates `this.mesh` (opaque) if `passID == 0`.
*   Updates `this.transparentMesh` (transparent) if `passID == 1`.
*   Sets `noVoxels` and `isVisible` flags on the respective meshes.

### Data Structures
*   **Voxel Indexing**: The index of a voxel at position `pos = [x, y, z]` is calculated as:
    $$ index = x + y \cdot dimensions[0] + z \cdot dimensions[0] \cdot dimensions[1] $$
*   **Colors Array**: A flat array of floats. For each vertex, 4 consecutive floats store RGBA values normalized to $[0, 1]$. The layout is `[r, g, b, a, r, g, b, a, ...]`.

### Algorithm Steps
1.  **Generate Raw Geometry**: Call the greedy mesher to obtain `rawMesh`.
    ```python
    rawMesh = this.mesher(this.voxelData.voxels, this.voxelData.dimensions, this.evaluateFunction, passID)
    ```
2.  **Initialize Arrays**: Create empty arrays for `indices` and `colors`. The `colors` array must be sized to accommodate all vertices in the raw mesh. If the raw mesh has $V$ vertices, `colors` has length $4V$.
3.  **Process Faces**: Iterate over each face `q` in `rawMesh.faces`.
    a.  **Indices**: Push vertex indices in reverse winding order to `indices`:
        ```python
        indices.push(q[2], q[1], q[0])
        ```
    b.  **Color Retrieval**: Get the color for the face using `this.coloringFunction(q[3], q[4])`.
    c.  **Color Validation**:
        *   If the result is `null` or has fewer than 3 components, set color to `[300, 75, 300, 255]`.
        *   If the result has exactly 3 components, append alpha `255`.
        *   Normalize RGB components by dividing by 255.
    d.  **Populate Colors**: For each of the three vertex indices `i2` in `q[0], q[1], q[2]`:
        *   Calculate offset: `offset = i2 * 4`.
        *   Set `colors[offset] = R`, `colors[offset+1] = G`, `colors[offset+2] = B`, `colors[offset+3] = A`.
4.  **Construct Vertex Data**: Create a `BABYLON.VertexData` object using the processed `positions` (from `rawMesh`), `indices`, `normals` (from `rawMesh`), and `colors`.
5.  **Apply to Mesh**:
    *   If `passID == 0`:
        *   Apply vertex data to `this.mesh`.
        *   If `vertexData.positions.length > 0`:
            *   Set `this.mesh.noVoxels = false`.
            *   Restore `this.mesh.isVisible` from `oldVisibility`.
        *   If `hasTransparency` is true, recursively call `updateMesh(1)`.
    *   If `passID == 1`:
        *   Apply vertex data to `this.transparentMesh`.
        *   If `vertexData.positions.length > 0`:
            *   Set `this.transparentMesh.noVoxels = false`.
            *   Restore `this.transparentMesh.isVisible` from `oldVisibility`.

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def update_mesh(voxel_data, mesher, evaluate_function, coloring_function, pass_id, has_transparency, old_visibility):
    # Step 1: Generate raw mesh
    raw_mesh = mesher(voxel_data.voxels, voxel_data.dimensions, evaluate_function, pass_id)
    
    indices = []
    # Determine size of colors array based on raw mesh vertices
    num_vertices = len(raw_mesh.positions)
    colors = [0.0] * (num_vertices * 4)
    
    # Step 3: Process faces
    for q in raw_mesh.faces:
        # 3a: Indices in reverse winding
        indices.append(q[2])
        indices.append(q[1])
        indices.append(q[0])
        
        # 3b: Get color
        color = coloring_function(q[3], q[4])
        
        # 3c: Validate color
        if color is None or len(color) < 3:
            color = [300, 75, 300, 255]
        elif len(color) == 3:
            color.append(255)
        
        # Normalize RGB
        r = color[0] / 255.0
        g = color[1] / 255.0
        b = color[2] / 255.0
        a = color[3] / 255.0
        
        # 3d: Populate colors for the three vertices of the face
        for i2 in [q[0], q[1], q[2]]:
            offset = i2 * 4
            colors[offset] = r
            colors[offset+1] = g
            colors[offset+2] = b
            colors[offset+3] = a
            
    # Step 4: Construct VertexData
    vertex_data = BABYLON.VertexData()
    vertex_data.positions = raw_mesh.positions
    vertex_data.indices = indices
    vertex_data.normals = raw_mesh.normals
    vertex_data.colors = colors
    vertex_data.applyToMesh(None) # Placeholder for application logic
    
    # Step 5: Apply to specific mesh
    if pass_id == 0:
        this.mesh.setVertexData(vertex_data)
        if len(vertex_data.positions) > 0:
            this.mesh.noVoxels = False
            this.mesh.isVisible = old_visibility
        if has_transparency:
            update_mesh(voxel_data, mesher, evaluate_function, coloring_function, 1, has_transparency, old_visibility)
    elif pass_id == 1:
        this.transparent_mesh.setVertexData(vertex_data)
        if len(vertex_data.positions) > 0:
            this.transparent_mesh.noVoxels = False
            this.transparent_mesh.isVisible = old_visibility
```

### Edge Cases and Pitfalls
*   **Color Defaulting**: If `coloringFunction` returns `null` or an incomplete array, the method defaults to magenta `[300, 75, 300, 255]`. This is a specific implementation detail of the CEWBS library [S1, unofficial].
*   **Transparency Handling**: The method relies on an external `hasTransparency` flag to decide whether to run the second pass. If this flag is incorrect, transparent faces may be rendered incorrectly or not at all [S1, unofficial].
*   **Winding Order**: Vertices are pushed in reverse order (`q[2], q[1], q[0]`) to ensure correct face culling in BabylonJS [S1, unofficial].

## Operating characteristics
*   **Time Complexity**: Linear with respect to the number of faces in the raw mesh, $O(F)$, where $F$ is the number of triangles generated by the greedy mesher. The second pass is only executed if `hasTransparency` is true, effectively doubling the work in the worst case.
*   **Memory Complexity**: Linear with respect to the number of vertices, $O(V)$, for storing the `colors` array. The `indices` array also scales with $O(F)$.
*   **Strengths**: Separates opaque and transparent geometry, allowing for correct rendering of semi-transparent voxel data in BabylonJS. Simple to implement if the greedy mesher is already available.
*   **Limitations**: Tightly coupled to BabylonJS (`BABYLON.VertexData`). The color defaulting logic is arbitrary. Performance depends heavily on the efficiency of the underlying greedy mesher.
*   **When to use**: When rendering voxel data with transparency in BabylonJS and using the CEWBS library's greedy mesher.
*   **When not to use**: For opaque-only data (single pass is sufficient), or when using a different rendering engine.

## Background
This method is part of the CEWBS (C++/JS Voxel Engine) library, specifically designed for integration with BabylonJS. It was developed to handle the visualization of voxel data with varying opacity levels by splitting the mesh generation into two passes. The approach leverages the greedy meshing algorithm to optimize geometry and then post-processes the output to separate transparent faces.

## Sources
[S1] wwwredfish/CEWBS, https://github.com/wwwredfish/CEWBS, fetched 2026-10-26 (unofficial)
