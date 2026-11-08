# TetSplat Pre-processing and Data Layout

Also called: TetSplat Quantization and Bitstring Packing

## Overview
TetSplat Pre-processing and Data Layout is the method for quantizing scalar field values and packing them into binary bitstrings for efficient storage and transmission within the TetSplat volume rendering system [S1, unofficial]. It solves the problem of managing large tetrahedral meshes with multiple scalar fields by compressing hierarchical data structures (Shell and Solid) into a single binary stream [S1, unofficial]. It sets itself apart from general quantization methods by its specific integration with unbalanced tree traversal pointers and the hierarchical encoding of geometric attributes (normals) alongside scalar data [S1, unofficial].

## How it works

### Inputs and Outputs
*   **Input:** A tetrahedral mesh partitioned across multiple CPU nodes, containing multiple scalar field values per element.
*   **Output:** A single binary data stream representing the compressed hierarchical data structures (Shell and Solid) in breadth-first order, ready for `mmap`-based loading [S1, unofficial].

### Data Structures and Layout

#### 1. Scalar Field Quantization
Scalar field values are normalized to the range $[0, 1]$ and quantized using a lookup table [S1, unofficial].
*   **Precision:** 15 bits per scalar field value is sufficient for the studied datasets [S1, unofficial].
*   **Bitstring Construction:** The quantized values are appended to a bitstring. Each additional field value adds 2 bytes (16 bits) to the bitstring, though only 15 bits are strictly necessary for the data [S1, unofficial].

#### 2. Node Layouts
The method defines two primary node types for the hierarchical structures:

**Solid Node Layout**
*   **Total Size:** 6 bytes.
*   **Content:** Contains two field values packed into the bitstring.
*   **Structure:**
    *   Bytes 0-1: First field value (quantized).
    *   Bytes 2-3: Second field value (quantized).
    *   Bytes 4-5: Additional data or padding as defined by the specific implementation of the "two field values" constraint. *Note: The source specifies "6 bytes total... with two field values" and "2 extra bytes per field value". This implies a base structure plus the field data. The exact byte alignment of the 6 bytes is not explicitly detailed beyond the total size and field count.* [S1, unofficial].

**Shell Node Layout**
*   **Total Size:** 8 bytes (6 bytes base + 2 bytes extra).
*   **Content:** Contains the same scalar field data as the Solid node, plus geometric attributes.
*   **Extra 2 Bytes:** Hierarchically encode:
    *   Normal vector: 14 bits.
    *   Width of normal cone: 2 bits.
    *   *Reference:* See [20, 18] for the specific hierarchical encoding scheme [S1, unofficial].

#### 3. Tree Traversal Pointers
For unbalanced trees (Shell quadtree, Solid octree), pointers are embedded to enable fast top-down traversal [S1, unofficial].
*   **Pointer Size:** 4-byte integer.
*   **Placement:** Inserted for each group of parents to the first node of their group of children.
*   **Memory Overhead:** The relative overhead is lower for Solid than Shell because the Solid's average branching factor is twice as high, resulting in fewer internal nodes and thus fewer pointers, despite the Solid being larger in volume [S1, unofficial].

### Numbered Steps

1.  **Parallel Partitioning:** Partition the tetrahedral mesh onto multiple CPU nodes. Each node runs its own pre-processing on its fragment [S1, unofficial].
2.  **Normalization:** Normalize each scalar field value to $[0, 1]$ [S1, unofficial].
3.  **Quantization:** Quantize the normalized values using a lookup table with 15-bit precision [S1, unofficial].
4.  **Bitstring Packing:** Append the quantized values to a bitstring. For each field value, allocate 2 bytes (16 bits) in the bitstring [S1, unofficial].
5.  **Attribute Encoding (Shell Only):** For Shell nodes, hierarchically encode the normal vector (14 bits) and normal cone width (2 bits) into an extra 2 bytes [S1, unofficial].
6.  **Cleanup and Boundary Update:** Trim shared triangles from the Shell and add them to the Solid. Update the boundary label assignment to tetrahedra and triangles in the Solid [S1, unofficial].
7.  **Pointer Insertion:** For unbalanced trees, insert a 4-byte int pointer for each group of parents to the first node of their group of children [S1, unofficial].
8.  **File Output:** Flush the compressed hierarchical data structures to the filesystem as a single binary data stream in breadth-first order [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def preprocess_tetsplat(mesh, scalar_fields, num_nodes):
    """
    Pre-processes a tetrahedral mesh for TetSplat rendering.
    
    Args:
        mesh: Tetrahedral mesh object.
        scalar_fields: List of scalar field values per element.
        num_nodes: Number of CPU nodes for parallel processing.
        
    Returns:
        binary_stream: Single binary data stream of compressed structures.
    """
    # Step 1: Parallel Partitioning
    partitions = partition_mesh(mesh, num_nodes)
    
    # Step 2-5: Per-node Pre-processing
    node_streams = []
    for node_id, fragment in enumerate(partitions):
        # Normalize scalar fields to [0, 1]
        normalized_fields = normalize(fragment.scalar_fields, min_val=0.0, max_val=1.0)
        
        # Quantize using lookup table (15-bit precision)
        quantized_fields = quantize_lookup(normalized_fields, bits=15)
        
        # Pack into bitstring (2 bytes per field value)
        bitstring = pack_bitstring(quantized_fields, bytes_per_field=2)
        
        # Encode Shell attributes if applicable
        if fragment.is_shell:
            normal_bits = encode_normal(fragment.normals, bits=14)
            cone_width_bits = encode_cone_width(fragment.cone_widths, bits=2)
            extra_bytes = pack(normal_bits + cone_width_bits)
            bitstring += extra_bytes
            
        # Step 6: Cleanup (Trim shared triangles from Shell, add to Solid)
        # This requires inter-node communication or a centralized cleanup step
        # For the sake of this pseudocode, assume a cleanup function handles this
        fragment = cleanup_shell_solid(fragment)
        
        # Step 7: Insert Traversal Pointers for Unbalanced Trees
        # Insert 4-byte int pointer for each group of parents to first child
        bitstring = insert_pointers(bitstring, tree_type=fragment.tree_type)
        
        node_streams.append(bitstring)
        
    # Step 8: Merge and Output
    # In a real implementation, this would involve merging streams and handling
    # the centralized cleanup of shared boundaries.
    # Here we assume the streams are concatenated in breadth-first order.
    binary_stream = concatenate(node_streams)
    
    return binary_stream

def normalize(values, min_val, max_val):
    """Normalize values to [0, 1]."""
    return [(v - min_val) / (max_val - min_val) for v in values]

def quantize_lookup(values, bits):
    """Quantize values using a lookup table."""
    # Implementation detail: Map normalized value to integer index
    # 15 bits allows for 32768 levels
    levels = 2 ** bits
    return [int(v * levels) for v in values]

def pack_bitstring(fields, bytes_per_field):
    """Pack quantized fields into a bitstring."""
    # Each field takes bytes_per_field bytes
    # 15 bits of data stored in 16 bits (2 bytes)
    packed = b''
    for field in fields:
        # Convert integer to 2-byte big-endian representation
        packed += field.to_bytes(bytes_per_field, byteorder='big')
    return packed

def encode_normal(normals, bits):
    """Encode normal vector into bits."""
    # Hierarchical encoding as per [20, 18]
    # Not fully specified in [S1, unofficial], so placeholder for implementation
    return 0 # Placeholder

def encode_cone_width(widths, bits):
    """Encode normal cone width into bits."""
    # Hierarchical encoding as per [20, 18]
    return 0 # Placeholder

def insert_pointers(bitstring, tree_type):
    """Insert 4-byte int pointers for unbalanced tree traversal."""
    # Insert pointer for each group of parents to first child
    # Implementation depends on the specific tree structure
    return bitstring # Placeholder
```

### Parameters
*   **Quantization Precision:** 15 bits per scalar field value [S1, unofficial].
*   **Bytes per Field Value:** 2 bytes (16 bits) in the bitstring [S1, unofficial].
*   **Pointer Size:** 4 bytes (32-bit integer) [S1, unofficial].
*   **Normal Vector Bits:** 14 bits [S1, unofficial].
*   **Normal Cone Width Bits:** 2 bits [S1, unofficial].

### Edge Cases and Pitfalls
*   **Memory Overhead:** The relative memory overhead of 32-bit pointers is much less for Solid than Shell. Although Solid is larger, its average branching factor is twice as high, leading to fewer internal nodes and thus fewer pointers [S1, unofficial].
*   **Shared Triangles:** Triangles shared between mesh fragments must be trimmed from the Shell and added to the Solid. Failure to do so results in incorrect boundary labels and visualization artifacts [S1, unofficial].
*   **Unbalanced Trees:** Pointers must be correctly inserted for unbalanced trees (Shell quadtree, Solid octree) to ensure fast traversal. Incorrect pointer placement leads to traversal errors [S1, unofficial].

## Operating characteristics

### Time and Memory Complexity
*   **Time Complexity:** The pre-processing is parallelizable. Each CPU node runs its own pre-processing, followed by a cleanup process. The complexity is linear with respect to the number of elements per node [S1, unofficial].
*   **Memory Complexity:** The memory overhead includes the compressed data structures plus the traversal pointers. The relative overhead is lower for Solid due to its higher branching factor [S1, unofficial].

### Measured Performance
*   **Performance:** Not given in the sources.
*   **Hardware:** Tested on multi-CPU node systems [S1, unofficial].

### Scaling
*   **Scaling:** The method scales well with the number of CPU nodes due to parallel pre-processing [S1, unofficial].

### Strengths and Limitations
*   **Strengths:** Efficient storage and transmission of large tetrahedral meshes with multiple scalar fields. Fast top-down multi-resolution rendering facilitated by breadth-first storage and embedded pointers [S1, unofficial].
*   **Limitations:** The method is specific to the TetSplat system and its hierarchical structures. The reliance on unbalanced trees requires careful pointer management [S1, unofficial].

### When to Use
*   Use when rendering large tetrahedral meshes with multiple scalar fields in an interactive visualization context [S1, unofficial].
*   Use when fast top-down multi-resolution rendering is required [S1, unofficial].

### When Not to Use
*   Do not use for balanced tree structures where pointers are not needed [S1, unofficial].
*   Do not use if the scalar field values require higher precision than 15 bits [S1, unofficial].

## Background

### Origin
The TetSplat method was introduced by Museth and Lombeyda in 2004 [S1, unofficial].

### History and Development
The method was developed to facilitate interactive visualization of large tetrahedral meshes with multiple scalar fields. It builds on the QSplat file layout strategy, adapting it for unbalanced trees and multiple scalar fields [S1, unofficial].

### Where it is Used
Used in interactive visualization algorithms for volume rendering and volume clipping of large tetrahedral meshes [S1, unofficial].

## Sources

[S1] TetSplat: Real-time Rendering and Volume Clipping of Large ... - Museth, https://ken.museth.org/Publications_files/Museth-Lombeyda_VIS04.pdf, fetched 2026-10-28 (unofficial)
