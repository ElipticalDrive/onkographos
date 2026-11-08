# TetSplat Pre-processing and Data Layout

Also called: TetSplat Quantization and Bitstring Packing

## Overview
TetSplat Pre-processing and Data Layout is a method for compressing scalar field values on tetrahedral meshes by quantizing them into a binary bitstring and packing them into hierarchical data structures for efficient storage and transmission [S1, unofficial]. It solves the problem of managing large volumes of multi-field scalar data in interactive visualization by reducing precision and organizing data into unbalanced Shell (quadtree) and Solid (octree) structures [S1, unofficial]. It sets itself apart from sibling methods like [Attribute Quantization](1912-attribute-quantization.md) and [Scalar Quantization](5726-scalar-quantization.md) by specifically targeting tetrahedral mesh topology, using a hybrid layout that distinguishes between surface-adjacent "Shell" nodes and interior "Solid" nodes, each with distinct bit-packing schemes [S1, unofficial].

## How it works
**Inputs:** A tetrahedral mesh partitioned across multiple CPU nodes, with multiple scalar field values associated with the mesh geometry.
**Outputs:** A single binary data stream containing compressed hierarchical data structures (Shell and Solid) flushed to the filesystem.

**Data Structures and Layout:**
The method employs two primary hierarchical structures:
1.  **Solid Octree:** Represents the interior volume.
2.  **Shell Quadtree:** Represents the boundary/surface.

**Bitstring Packing:**
Scalar field values are normalized to the range $[0, 1]$ and quantized using a lookup table [S1, unofficial].
*   **Solid Node Layout:** A base Solid node occupies 6 bytes. This layout contains two field values packed into the bitstring. Each additional field value adds 2 bytes to the bitstring [S1, unofficial].
*   **Shell Node Layout:** A Shell node contains the same base layout as the Solid node plus an extra 2 bytes. These 2 bytes hierarchically encode the normal vector (14 bits) and the width of the normal cone (2 bits) [S1, unofficial].
*   **Tree Traversal Pointers:** Because the trees are unbalanced, 4-byte integer pointers are embedded into the data structures. A pointer is inserted for each group of parents to the first node of their group of children to enable fast traversal [S1, unofficial].

**Algorithm Steps:**
1.  **Parallel Partitioning:** Partition the tetrahedral mesh onto multiple CPU nodes. Each node independently runs the pre-processing on its mesh fragment [S1, unofficial].
2.  **Normalization and Quantization:** For each scalar field value in the mesh fragment, normalize the value to $[0, 1]$ and quantize it using a lookup table. The sources indicate that 15 bits per scalar field value is sufficient to represent all relevant field values for the studied datasets [S1, unofficial].
3.  **Bitstring Construction:** Construct the bitstring for each node.
    *   For Solid nodes, pack the quantized field values into the 6-byte base layout, appending 2 bytes per additional field value.
    *   For Shell nodes, append the 2-byte normal vector and cone width encoding to the base layout.
4.  **Cleanup and Boundary Update:** A cleanup process trims triangles from the Shell that are shared between mesh fragments and adds them to the Solid. After this cleanup, the boundary label assignment to tetrahedra and triangles in the Solid must be updated [S1, unofficial].
5.  **Pointer Embedding:** Insert 4-byte integer pointers for parent groups to the first child node in the unbalanced Shell and Solid trees [S1, unofficial].
6.  **File Output:** Flush the compressed hierarchical data structures to the filesystem as a single binary data stream, saved in breadth-first order to facilitate fast top-down multi-resolution rendering [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def preprocess_tetsplat(mesh, scalar_fields, num_bits=15):
    """
    Pre-processes a tetrahedral mesh for TetSplat rendering.
    
    Args:
        mesh: Partitioned tetrahedral mesh fragment.
        scalar_fields: List of scalar field values associated with the mesh.
        num_bits: Quantization precision in bits (default 15).
        
    Returns:
        binary_stream: Single binary data stream of Shell and Solid nodes.
    """
    # Step 1: Normalize and Quantize
    # Normalize fields to [0, 1] and quantize using lookup table
    quantized_fields = []
    for field in scalar_fields:
        normalized = normalize_to_0_1(field)
        quantized = quantize_with_lookup_table(normalized, num_bits)
        quantized_fields.append(quantized)
        
    # Step 2: Construct Node Layouts
    solid_nodes = []
    shell_nodes = []
    
    for tet in mesh.tetrahedra:
        # Determine if tet is in Solid or Shell based on boundary labels
        if is_boundary_tet(tet):
            # Shell Node: Base 6 bytes + 2 bytes for normals
            # Pack two field values into base 6 bytes
            base_bits = pack_two_fields(quantized_fields[0], quantized_fields[1])
            # Pack normal vector (14 bits) and cone width (2 bits)
            normal_bits = pack_normal(tet.normal, tet.cone_width)
            shell_node_bits = base_bits + normal_bits
            shell_nodes.append(shell_node_bits)
        else:
            # Solid Node: Base 6 bytes + 2 bytes per additional field
            base_bits = pack_two_fields(quantized_fields[0], quantized_fields[1])
            extra_bits = b''
            for i in range(2, len(quantized_fields)):
                extra_bits += pack_single_field(quantized_fields[i])
            solid_node_bits = base_bits + extra_bits
            solid_nodes.append(solid_node_bits)
            
    # Step 3: Cleanup and Boundary Update
    # Trim shared triangles from Shell and add to Solid
    # Update boundary labels for Solid tetrahedra
    shell_nodes, solid_nodes = cleanup_shared_triangles(shell_nodes, solid_nodes)
    update_boundary_labels(solid_nodes)
    
    # Step 4: Embed Pointers for Unbalanced Trees
    # Insert 4-byte int pointer for each parent group to first child
    solid_tree = build_unbalanced_octree(solid_nodes)
    shell_tree = build_unbalanced_quadtree(shell_nodes)
    solid_tree = embed_pointers(solid_tree, pointer_size=4)
    shell_tree = embed_pointers(shell_tree, pointer_size=4)
    
    # Step 5: Flush to Binary Stream
    # Save in breadth-first order
    binary_stream = serialize_breadth_first(solid_tree) + serialize_breadth_first(shell_tree)
    
    return binary_stream

def normalize_to_0_1(value):
    # [G] Implementation detail: (value - min) / (max - min)
    return (value - min_value) / (max_value - min_value)

def quantize_with_lookup_table(normalized_value, num_bits):
    # [G] Implementation detail: Map normalized value to discrete level
    level = int(normalized_value * (2**num_bits - 1))
    return level

def pack_two_fields(field1, field2):
    # [G] Implementation detail: Pack two 15-bit values into 6 bytes (48 bits)
    # Assuming specific bit-packing scheme described in source
    return (field1 << 15) | field2

def pack_single_field(field):
    # [G] Implementation detail: Pack one 15-bit value into 2 bytes (16 bits)
    return field

def pack_normal(normal, cone_width):
    # [G] Implementation detail: Encode normal (14 bits) and cone width (2 bits)
    normal_bits = encode_normal(normal, 14)
    width_bits = encode_cone_width(cone_width, 2)
    return (normal_bits << 2) | width_bits

def cleanup_shared_triangles(shell_nodes, solid_nodes):
    # [G] Implementation detail: Identify shared triangles, move from Shell to Solid
    # Update labels accordingly
    pass

def update_boundary_labels(solid_nodes):
    # [G] Implementation detail: Recalculate boundary labels after cleanup
    pass

def build_unbalanced_octree(nodes):
    # [G] Implementation detail: Construct octree from solid nodes
    pass

def build_unbalanced_quadtree(nodes):
    # [G] Implementation detail: Construct quadtree from shell nodes
    pass

def embed_pointers(tree, pointer_size):
    # [G] Implementation detail: Insert 4-byte pointers for parent groups
    pass

def serialize_breadth_first(tree):
    # [G] Implementation detail: Serialize tree in breadth-first order
    pass
```

**Parameters:**
*   **Quantization Precision:** 15 bits per scalar field value is sufficient for the studied datasets [S1, unofficial].
*   **Pointer Size:** 4 bytes (32-bit integer) for tree traversal pointers [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Memory Overhead:** The relative memory overhead of 32-bit pointers is much less for the Solid structure than for the Shell. Although the Solid is typically several orders of magnitude larger than the Shell, its average branching factor is twice as high, leading to fewer internal nodes and thus fewer pointers [S1, unofficial].
*   **Unbalanced Trees:** The use of unbalanced trees necessitates the embedding of pointers, which adds complexity to the data structure layout compared to balanced trees [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources, but the method introduces memory overhead for 4-byte pointers in unbalanced trees, which is relatively lower for Solid nodes due to their higher branching factor [S1, unofficial].
*   **Measured Performance:** Not given in the sources.
*   **Strengths:** Efficient storage and transmission of multi-field scalar data on tetrahedral meshes; supports interactive editing of transfer functions; parallelizable pre-processing [S1, unofficial].
*   **Limitations:** Requires unbalanced tree traversal logic; memory overhead for pointers in Shell nodes is higher relative to data size [S1, unofficial].
*   **When to Use:** Interactive visualization of large tetrahedral meshes with multiple scalar fields where bandwidth and memory are constrained [S1, unofficial].
*   **When Not to Use:** When balanced tree structures are required or when scalar field precision requirements exceed 15 bits [S1, unofficial].

## Background
TetSplat Pre-processing and Data Layout was introduced by Museth and Lombeyda in 2004 in the paper "TetSplat: Real-time Rendering and Volume Clipping of Large Tetrahedral Meshes" [S1, unofficial]. The method builds on the QSplat file layout strategy, adapting it for tetrahedral meshes and unbalanced trees [S1, unofficial]. It is used in the context of interactive visualization algorithms for large scientific datasets [S1, unofficial].

## Sources
[S1] TetSplat: Real-time Rendering and Volume Clipping of Large ... - Museth, https://ken.museth.org/Publications_files/Museth-Lombeyda_VIS04.pdf, fetched 2026-10-28 (unofficial)
