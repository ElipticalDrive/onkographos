# TetSplat Pre-processing and Data Layout

Also called: TetSplat Quantization and Bitstring Packing

## Overview
This method defines the pre-processing pipeline for the TetSplat volume rendering system, specifically focusing on the quantization of scalar field values and the binary layout of hierarchical data structures (Shell and Solid) for efficient storage and traversal [S1, unofficial]. It solves the problem of efficiently encoding multi-field scalar data on unstructured tetrahedral meshes for real-time interactive visualization by mapping continuous values to discrete bitstrings and organizing them into unbalanced trees with embedded traversal pointers [S1, unofficial]. It is distinguished from general quantization methods by its specific integration into a hierarchical tetrahedral mesh representation (Shell/Solid split) and its use of fixed-size bit-packing for field values combined with hierarchical normal encoding for surface boundaries [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A tetrahedral mesh partitioned across multiple CPU nodes, containing multiple scalar field values per element.
*   **Output:** A single binary data stream flushed to the filesystem, containing compressed hierarchical data structures for the Shell (surface) and Solid (volume) in breadth-first order.

**Data Structures and Layout:**
The data is organized into two primary hierarchical structures: the **Solid** (an unbalanced octree) and the **Shell** (an unbalanced quadtree).

1.  **Solid Node Layout:**
    *   **Base Size:** 6 bytes.
    *   **Content:** Contains two scalar field values packed into a binary bitstring.
    *   **Field Value Encoding:** Each field value is quantized to 15 bits. The 6-byte base layout accommodates two field values ($2 \times 15 = 30$ bits) plus overhead/alignment.
    *   **Additional Fields:** Each additional field value adds 2 bytes (16 bits) to the bitstring.
    *   **Pointer:** For unbalanced trees, a 4-byte integer pointer is inserted for each group of parents to the first node of their group of children to enable fast traversal [S1, unofficial].

2.  **Shell Node Layout:**
    *   **Base Size:** 8 bytes (6 bytes base + 2 bytes extra).
    *   **Content:** Inherits the field value packing of the Solid node.
    *   **Normal Encoding:** The extra 2 bytes hierarchically encode the normal vector (14 bits) and the width of the normal cone (2 bits) [S1, unofficial].
    *   **Pointer:** Includes the same 4-byte integer pointer structure as the Solid for unbalanced tree traversal [S1, unofficial].

**Parameters:**
*   **Quantization Precision:** 15 bits per scalar field value. This was found sufficient to represent all relevant field values for the studied datasets [S1, unofficial].
*   **Pointer Size:** 4 bytes (32-bit integer) for tree traversal pointers [S1, unofficial].

**Numbered Steps:**
1.  **Normalization:** Normalize each scalar field value to the range $[0, 1]$ [S1, unofficial].
2.  **Quantization:** Quantize the normalized values using a lookup table to discrete levels [S1, unofficial].
3.  **Bitstring Packing:** Pack the quantized values into a binary bitstring. The first two values occupy the base 6-byte layout; subsequent values are appended in 2-byte chunks [S1, unofficial].
4.  **Parallel Pre-processing:** Partition the tetrahedral mesh onto multiple CPU nodes. Each node runs its own pre-processing on its mesh fragment [S1, unofficial].
5.  **Cleanup and Boundary Update:** A cleanup process trims triangles from the Shell that are shared between fragments and adds them to the Solid. The boundary label assignment to tetrahedra and triangles in the Solid is updated accordingly [S1, unofficial].
6.  **Tree Construction:** Construct the unbalanced octree (Solid) and quadtree (Shell). Insert 4-byte integer pointers from parent groups to the first child node in each group to support fast traversal [S1, unofficial].
7.  **File Output:** Flush the compressed hierarchical data structures to the filesystem as a single binary data stream in breadth-first order [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def preprocess_tetsplat(mesh, scalar_fields, num_fields):
    """
    Pre-processes a tetrahedral mesh for TetSplat rendering.
    
    Args:
        mesh: Tetrahedral mesh data structure.
        scalar_fields: List of scalar field arrays, one per field.
        num_fields: Number of scalar fields to quantize.
        
    Returns:
        binary_stream: Single binary data stream for Shell and Solid.
    """
    # Step 1-3: Quantization and Packing
    # Normalize and quantize each field value to 15 bits
    quantized_fields = []
    for field in scalar_fields:
        normalized = normalize_to_0_1(field)
        quantized = quantize_with_lookup_table(normalized, bits=15)
        quantized_fields.append(quantized)
        
    # Pack into bitstring
    # Base 6 bytes for first two fields
    bitstring = pack_fields(quantized_fields[0], quantized_fields[1], bits_per_field=15)
    
    # Append additional fields in 2-byte chunks
    for i in range(2, num_fields):
        bitstring += pack_fields(quantized_fields[i], bits_per_field=15)
        
    # Step 4-5: Parallel Pre-processing and Cleanup
    # Partition mesh and process fragments in parallel
    fragments = partition_mesh(mesh)
    processed_fragments = []
    for fragment in fragments:
        # Run pre-processing on fragment
        shell_nodes, solid_nodes = process_fragment(fragment, bitstring)
        
        # Cleanup: Trim shared triangles from Shell, add to Solid
        shared_triangles = find_shared_triangles(shell_nodes)
        for tri in shared_triangles:
            shell_nodes.remove(tri)
            solid_nodes.add(tri)
            
        # Update boundary labels in Solid
        update_boundary_labels(solid_nodes)
        
        processed_fragments.append((shell_nodes, solid_nodes))
        
    # Step 6: Tree Construction with Pointers
    shell_tree = build_unbalanced_quadtree(processed_fragments, node_type='shell')
    solid_tree = build_unbalanced_octree(processed_fragments, node_type='solid')
    
    # Insert 4-byte pointers for fast traversal
    insert_traversal_pointers(shell_tree)
    insert_traversal_pointers(solid_tree)
    
    # Step 7: File Output
    binary_stream = flush_to_file(shell_tree, solid_tree, order='breadth_first')
    
    return binary_stream

def normalize_to_0_1(values):
    min_val = min(values)
    max_val = max(values)
    if max_val == min_val:
        return [0.5] * len(values)
    return [(v - min_val) / (max_val - min_val) for v in values]

def quantize_with_lookup_table(values, bits=15):
    # Implementation detail: Uses a pre-defined lookup table for 15-bit quantization
    # [G] Assumes a standard uniform or perceptual lookup table is available
    levels = 2 ** bits
    quantized = [int(v * (levels - 1)) for v in values]
    return quantized

def pack_fields(field1, field2, bits_per_field=15):
    # Pack two 15-bit values into 6 bytes (48 bits)
    # [G] Exact bit-packing order (e.g., field1 in lower bits) is not specified
    # Assuming field1 occupies lower 15 bits, field2 next 15 bits, rest padding
    packed = (field1 & 0x7FFF) | ((field2 & 0x7FFF) << 15)
    # Pad to 48 bits (6 bytes)
    packed = packed & 0xFFFFFFFFFFFF
    return packed.to_bytes(6, byteorder='little')

def insert_traversal_pointers(tree):
    # For unbalanced trees, insert 4-byte int pointer from parent group to first child
    for node in tree.nodes:
        if node.has_children():
            # [G] Logic to identify "group of parents" and "first node of their group of children"
            # is not fully specified. Assumes a standard parent-pointer insertion.
            node.traversal_pointer = node.children[0].address
```

**Edge Cases and Pitfalls:**
*   **Memory Overhead:** The relative memory overhead of 32-bit pointers is much less for the Solid than for the Shell. Although the Solid is typically several orders of magnitude larger than the Shell, its average branching factor is twice as high, leading to fewer internal nodes and thus fewer pointers [S1, unofficial].
*   **Unbalanced Trees:** Since unbalanced trees are used, pointers are necessary for traversal. The 4-byte pointer is inserted for each group of parents to the first node of their group of children [S1, unofficial].
*   **Shared Triangles:** Triangles shared between mesh fragments must be trimmed from the Shell and added to the Solid to avoid duplication and ensure correct boundary labeling [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Pre-processing is parallelizable across CPU nodes, with linear scaling relative to the number of mesh fragments [S1, unofficial].
*   **Memory Complexity:** The Solid structure has lower relative pointer overhead due to its higher branching factor, despite its larger size [S1, unofficial].
*   **Performance:** The breadth-first storage order facilitates fast top-down multi-resolution rendering [S1, unofficial].
*   **Strengths:** Efficient storage via bit-packing, fast traversal with embedded pointers, and parallel pre-processing [S1, unofficial].
*   **Limitations:** Fixed 15-bit quantization may not be sufficient for all datasets; unbalanced trees require pointer overhead [S1, unofficial].
*   **When to Use:** Real-time interactive visualization of large tetrahedral meshes with multiple scalar fields [S1, unofficial].
*   **When Not to Use:** When higher precision than 15 bits is required or when the mesh structure does not benefit from hierarchical decomposition [S1, unofficial].

## Background
The TetSplat method was introduced by Museth and Lombeyda in 2004 for real-time rendering and volume clipping of large tetrahedral meshes [S1, unofficial]. It builds on the QSplat file layout strategy, adapting it for unstructured tetrahedral data with multiple scalar fields [S1, unofficial]. The method is used in interactive visualization applications where efficient storage and fast traversal of complex volumetric data are critical [S1, unofficial].

## Sources
[S1] TetSplat: Real-time Rendering and Volume Clipping of Large ... - Museth, https://ken.museth.org/Publications_files/Museth-Lombeyda_VIS04.pdf, fetched 2026-10-28 (unofficial)
