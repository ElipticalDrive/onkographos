# Vector-based Octree Storage

Also called: Vector3 Octree, Vector-based Sparse Octree

## Overview
Vector-based Octree Storage is a compact memory layout for sparse octrees that encodes node geometry and child pointers into `vector3` (or `vec4`) structures, typically stored in linear buffers or textures. It solves the problem of high memory overhead in traditional pointer-based octrees by replacing explicit child pointers with implicit spatial offsets and packing occupancy flags into the `w` component of the node vector [S1, unofficial]. It differs from [Implicit Octree Storage](939-implicit-octree-storage.md) by explicitly storing the node's center position and using a flags field for child existence, whereas implicit variants often rely on fixed grid hashing or Morton codes without storing position data [S1, unofficial].

## How it works
**Inputs:** A constructed octree hierarchy containing nodes and associated entities (e.g., bounding boxes or objects).
**Outputs:** A linear memory buffer (or texture) of `vector3`/`vec4` elements representing the serialized octree.

**Data Structures:**
1.  **Node Vector:** A `vector3` (or `vec4`) where:
    *   `x, y, z`: The center point coordinates of the octree node [S1, unofficial].
    *   `w`: A flags field. The lower 8 bits denote which of the 8 possible child octants exist (1 bit per child) [S1, unofficial].
2.  **Entity Vector:** A `vector3` (or `vec4`) where:
    *   `r, g, b`: Dimensions of a bounding box [S1, unofficial].
    *   `w`: Additional data or metadata [S1, unofficial].
3.  **List Marker Vector:** A special `vector3`/`vec4` indicating that a list of objects follows. A common heuristic is a magic value, such as `w + z` equaling a specific constant, or using a function `func(x, y)` to denote the count of following objects [S1, unofficial].

**Algorithm:**
1.  **Build:** Construct the octree in memory. Each node stores its center and occupancy flags. Entities are stored as bounding boxes [S1, unofficial].
2.  **Serialize:** Walk the octree (e.g., depth-first or breadth-first). For each node, output its `vector3` representation to a buffer.
3.  **Append Entities:** If a node contains entities, output the List Marker Vector, followed by the Entity Vectors for each object [S1, unofficial].
4.  **Store:** The resulting buffer can be mapped to a 2D texture. For example, a $512 \times 512$ texture can hold a fully packed 5-level deep octree (32,768 nodes) with 8 objects each, or a 4-level octree with 64 objects each [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def serialize_octree(root_node):
    buffer = []
    
    # Helper to create node vector
    def make_node_vector(center, child_flags):
        # center: (x, y, z)
        # child_flags: integer 0-255, bit i set if child i exists
        w = child_flags & 0xFF
        return (center[0], center[1], center[2], w)

    # Helper to create entity vector
    def make_entity_vector(dimensions, metadata):
        return (dimensions[0], dimensions[1], dimensions[2], metadata)

    # Helper to create list marker
    def make_list_marker(object_count):
        # Magic value heuristic: w + z = magic
        # Assuming z is unused or set to 0 for marker, w = magic
        MAGIC = 999999 
        return (0, 0, 0, MAGIC) # Implementation detail: exact magic value depends on constraints [S1, unofficial]

    def walk(node):
        if node is None:
            return
        
        # Output node vector
        node_vec = make_node_vector(node.center, node.child_flags)
        buffer.append(node_vec)
        
        # Check if node has objects
        if node.has_objects:
            # Output list marker
            marker = make_list_marker(len(node.objects))
            buffer.append(marker)
            
            # Output objects
            for obj in node.objects:
                entity_vec = make_entity_vector(obj.bbox_dimensions, obj.metadata)
                buffer.append(entity_vec)
        
        # Recurse for existing children
        for i in range(8):
            if (node.child_flags >> i) & 1:
                walk(node.children[i])

    walk(root_node)
    return buffer
```

**Parameters:**
*   **Texture Resolution:** A $512 \times 512$ texture holds 262,144 texels. This supports a 5-level deep octree (32,768 nodes) with 8 objects each, or a 4-level deep octree with 64 objects each [S1, unofficial].
*   **Flags Bits:** 8 bits in the `w` component for child occupancy [S1, unofficial].

**Edge Cases:**
*   **Magic Value Collision:** The special vector for object lists must not collide with valid node or entity data. The source suggests `w + z` as a magic check, but does not specify the exact value [S1, unofficial].
*   **Precision:** Storing center points in `vector3` requires sufficient floating-point precision to avoid overlap or gaps in large volumes [S1, unofficial].

## Subtopics

### Branch-On-Need Octree
A variant introduced by Wilhelms et al. [S2, unofficial] for non-power-of-two volumes. Unlike a full octree that pads to the next power-of-two cube, it only stores nodes inside the original volume bounds [S2, unofficial]. Internal nodes store min/max values of their sub-tree [S2, unofficial]. The number of nodes per level is calculated from the side-length using binary locational codes of the maximal position (e.g., for a $5 \times 5 \times 4$ volume in an $8 \times 8 \times 8$ grid, Level 2 has 4 nodes, Level 3 has 32, Level 4 has 100) [S2, unofficial]. Each node stores a 3-bit code for child branching and a pointer to the first child [S2, unofficial].

## Operating characteristics
*   **Memory Efficiency:** High compression for sparse data by eliminating null pointers and storing only occupied nodes [S1, unofficial].
*   **Complexity:** Serialization is $O(N)$ where $N$ is the number of nodes. Traversal requires parsing the linear buffer and interpreting flags [S1, unofficial].
*   **Performance:** Texture-based storage allows GPU-friendly access but requires careful management of texture size limits (e.g., $512 \times 512$) [S1, unofficial].
*   **Limitations:** Implicit child location is not directly supported by Morton codes; traversal must follow the explicit flag-based structure [S1, unofficial].

## Background
The method was proposed in a Stack Overflow discussion by Kevin Hsu in 2011 as a compact storage format for sparse octrees in HLSL [S1, unofficial]. It is not a peer-reviewed algorithm but a practical engineering solution for GPU-based rendering [S1, unofficial].

## Sources
[S1] Efficient storage for a sparse octree?, https://stackoverflow.com/questions/6085209/efficient-storage-for-a-sparse-octree, fetched 2026-10-26 (unofficial)
[S2] Eine adaptive, hybride Datenstruktur für Volumen Datensätze auf der GPU, https://doi.org/10.34726/hss.2016.26562, fetched 2026-10-26 (unofficial)
