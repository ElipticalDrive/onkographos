# Mesh Configuration and Data Structures

Also called: Block ID Data Structures, ChunkView and FluidQuad Layout, SurfaceMeshNodeTypes Attribute

## Overview
This method defines the specific data structures and attribute arrays required to configure a block-based surface mesh for efficient assembly and rendering. It solves the problem of distinguishing between internal topology nodes (vertices, edges, points) and surface boundary nodes, while managing the performance-critical distinction between opaque, transparent, and fluid block types during mesh generation. It sets itself apart from sibling methods by enforcing strict type separation for block IDs (using native sets) and defining the exact integer mapping for node types used in debugging and visualization.

## How it works
### Inputs and Outputs
**Inputs:**
*   A `ChunkView` structure containing chunk coordinates, height data, and a byte view for meshing [S1, unofficial].
*   A registry of block types, where untrusted numeric inputs are converted via `BlockId.result` to ensure registry-derived validity [S1, unofficial].
*   Configuration defining the separation of transparent solids into `transparentSolidBlockIds` (atlas/alpha blend) and `waterBlockIds` (dedicated shader) [S1, unofficial].

**Outputs:**
*   A surface mesh with vertices carrying a `SurfaceMeshNodeTypes` attribute array [S2].
*   Layer-separated contiguous buffers (Opaque, Water, Transparent) with tracked `vertexOffset` and `indexOffset` [G].

### Data Structures and Layouts

#### 1. SurfaceMeshNodeTypes Attribute
An integer attribute array attached to the vertices of the surface mesh. It identifies the topological role of each vertex [S2].

| Id Value | Node Type |
| :--- | :--- |
| 2 | Normal Vertex |
| 3 | Triple Line |
| 4 | Quadruple Point |
| 12 | Normal Vertex on the outer surface |
| 13 | Triple Line on the outer surface |
| 14 | Quadruple Point on the outer surface |

#### 2. Block ID Management
Block IDs are used to determine material and rendering behavior.
*   **Storage:** Must use native JavaScript `Set` for membership testing. Custom `HashSet` implementations are forbidden due to performance constraints (~400k membership tests per chunk) [S1, unofficial].
*   **Validation:** All block IDs must be registry-derived. Untrusted numeric inputs must be converted using `BlockId.result` before being added to sets or maps [S1, unofficial].
*   **Categorization:**
    *   `transparentSolidBlockIds`: Includes glass, leaves, etc. (Atlas material + alpha blend) [S1, unofficial].
    *   `waterBlockIds`: Dedicated shader for ripples, refraction, and variable height [S1, unofficial].

#### 3. ChunkView and FluidQuad
*   **ChunkView:** Holds `coord` (coordinates), per-chunk `height`, and a byte view for meshing. It does not implicitly truncate or zero-pad different heights [S1, unofficial].
*   **FluidQuad:** Includes `direction`, `flow` (optional, containing `direction` vector and `falling` flag), and geometry properties [S1, unofficial].
*   **Storage Layout:** The `blockIndex` storage layout is identical to the reference implementation to allow direct use of reference chunk fixtures as golden inputs [S1, unofficial].

### Algorithm Steps

1.  **Initialize Block Sets:** Create native `Set` instances for `transparentSolidBlockIds` and `waterBlockIds`. Validate all incoming block IDs using `BlockId.result` before insertion [S1, unofficial].
2.  **Process Chunk View:** Iterate through the `ChunkView` data. Access `coord` and `height` without implicit truncation. Use the byte view for meshing operations [S1, unofficial].
3.  **Generate Mesh Geometry:** For each chunk, determine the surface topology. Assign `SurfaceMeshNodeTypes` values to vertices based on their position:
    *   Internal nodes receive IDs 2, 3, or 4.
    *   Outer surface nodes receive IDs 12, 13, or 14 [S2].
4.  **Assemble Layers:** Write vertices and indices into contiguous buffers. Assign vertices to layers based on their block ID:
    *   Opaque blocks -> Opaque layer.
    *   `waterBlockIds` -> Water layer.
    *   `transparentSolidBlockIds` -> Transparent layer [S1, unofficial].
5.  **Track Offsets:** Update `vertexOffset` and `indexOffset` for each layer to ensure correct mapping in the final mesh descriptor [G].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def configure_mesh(chunk_view, block_registry):
    """
    Configures mesh data structures and attributes for a given chunk.
    """
    # 1. Initialize native sets for block classification
    # Using native Set is required for performance (~400k tests/chunk) [S1, unofficial]
    transparent_solids = set()
    water_blocks = set()
    
    # 2. Populate sets from registry
    # Untrusted inputs must be converted via BlockId.result [S1, unofficial]
    for block_id in block_registry.get_all_ids():
        validated_id = block_registry.get_block_id_result(block_id)
        if validated_id in block_registry.water_types:
            water_blocks.add(validated_id)
        elif validated_id in block_registry.transparent_types:
            transparent_solids.add(validated_id)
            
    # 3. Process ChunkView
    # ChunkView holds coord, height, and byte view [S1, unofficial]
    coord = chunk_view.coord
    height = chunk_view.height
    byte_view = chunk_view.byte_view
    
    # 4. Generate Mesh and Assign Node Types
    # SurfaceMeshNodeTypes attribute mapping [S2]
    # 2: Normal Vertex, 3: Triple Line, 4: Quadruple Point
    # 12: Normal Vertex (outer), 13: Triple Line (outer), 14: Quadruple Point (outer)
    vertices = []
    node_types = []
    
    for vertex in generate_surface_vertices(byte_view, coord, height):
        is_outer_surface = is_on_outer_boundary(vertex, coord, height)
        
        # Determine topological type (2, 3, 4)
        node_type = determine_topological_type(vertex)
        
        # Map to outer surface type if applicable (12, 13, 14)
        if is_outer_surface:
            node_type += 10
            
        node_types.append(node_type)
        vertices.append(vertex)
        
    # 5. Layer Separation and Buffer Assembly [G]
    opaque_vertices = []
    water_vertices = []
    transparent_vertices = []
    
    for i, vertex in enumerate(vertices):
        block_id = get_block_id_at_vertex(vertex, byte_view)
        node_type = node_types[i]
        
        if block_id in water_blocks:
            water_vertices.append((vertex, node_type))
        elif block_id in transparent_solids:
            transparent_vertices.append((vertex, node_type))
        else:
            opaque_vertices.append((vertex, node_type))
            
    return {
        'opaque': opaque_vertices,
        'water': water_vertices,
        'transparent': transparent_vertices,
        'node_types': node_types
    }
```

### Parameters and Edge Cases
*   **ParaView Visualization:** To view the internal mesh structure, apply a Threshold filter on the `SurfaceMeshNodeTypes` array for values 2 to 4. To see triangles, select the 'Surface With Edges' rendering type [S2].
*   **Precision:** Do not rely on implicit truncation or zero-padding of heights in `ChunkView` [S1, unofficial].
*   **Performance:** Avoid custom hash sets for block ID membership testing; use native sets [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Membership testing in block ID sets is $O(1)$ on average using native sets. Mesh generation is linear with respect to the number of quads/chunks [S1, unofficial].
*   **Memory:** Contiguous buffering minimizes fragmentation. Attribute packing (e.g., normals to `Int8`) saves memory [G].
*   **Strengths:** Clear separation of concerns for debugging (via node types) and rendering (via layer separation). High performance for transparency handling.
*   **Limitations:** Requires strict adherence to native set usage for performance. Relies on registry-derived block IDs.

## Background
The `SurfaceMeshNodeTypes` attribute system is documented in the DREAM.3D tutorial for surface meshing, providing a standard way to visualize internal mesh topology [S2]. The block ID data structures and `ChunkView` layout are derived from the `mc-meshing` project, which emphasizes performance optimizations for JavaScript-based voxel meshing [S1, unofficial].

## Sources
[S1] nerima-games/mc-meshing, https://github.com/nerima-games/mc-meshing, fetched 2026-10-28 (unofficial)
[S2] SurfaceMeshing - DREAM.3D, https://dream3d.bluequartz.net/Help/2_Tutorials/SurfaceMeshing/, fetched 2026-10-30
