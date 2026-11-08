# TetSplat Pre-processing and Data Layout

Also called: TetSplat Quantization and Bitstring Packing

## Overview
TetSplat Pre-processing and Data Layout is a method for compressing scalar field values and geometric metadata into a binary bitstring format suitable for efficient storage and transmission in interactive volume rendering [S1, unofficial]. It solves the problem of managing large tetrahedral meshes with multiple scalar fields by normalizing values to $[0, 1]$, quantizing them into a fixed-width bitstring, and packing them into hierarchical tree structures (Shell and Solid) [S1, unofficial]. It differs from sibling methods like [Attribute Quantization](1912-attribute-quantization.md) by specifically targeting the hierarchical node layout of unbalanced quadtrees and octrees, embedding traversal pointers directly into the data stream to facilitate fast top-down rendering without loading the entire structure into memory [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A tetrahedral mesh partitioned across multiple CPU nodes, containing multiple scalar field values per element.
*   **Output:** A single binary data stream containing compressed hierarchical data structures for the Shell (surface) and Solid (volume), written to the filesystem in breadth-first order [S1, unofficial].

**Data Structures and Layout:**
The method constructs two types of nodes: Solid nodes (for the volume) and Shell nodes (for the surface).

1.  **Solid Node Layout:**
    *   **Base Size:** 6 bytes.
    *   **Content:** Contains two scalar field values packed into a bitstring.
    *   **Expansion:** Each additional scalar field value adds 2 bytes to the bitstring [S1, unofficial].
    *   **Pointer:** For unbalanced octrees, a 4-byte integer pointer is inserted for each group of parents to the first node of their group of children to enable fast traversal [S1, unofficial].

2.  **Shell Node Layout:**
    *   **Base Size:** 8 bytes (6 bytes base + 2 bytes extra).
    *   **Content:** The extra 2 bytes hierarchically encode:
        *   Normal vector: 14 bits.
        *   Width of normal cone: 2 bits [S1, unofficial].
    *   **Pointer:** For unbalanced quadtrees, a 4-byte integer pointer is inserted for each group of parents to the first node of their group of children [S1, unofficial].

**Algorithm Steps:**

1.  **Partitioning:** Partition the tetrahedral mesh onto multiple CPU nodes. Each node runs its own pre-processing independently [S1, unofficial].
2.  **Normalization:** Normalize each scalar field value to the range $[0, 1]$ [S1, unofficial].
3.  **Quantization:** Quantize the normalized values using a lookup table. The sources specify that 15 bits per scalar field value is sufficient to represent all relevant field values for the studied datasets [S1, unofficial].
4.  **Bitstring Packing:** Append the quantized field values to the node's bitstring. The first two values occupy the base 6 bytes; subsequent values add 2 bytes each [S1, unofficial].
5.  **Cleanup (Parallel):** Trim triangles from the Shell that are shared between mesh fragments and add them to the Solid. Update boundary label assignments for tetrahedra and triangles in the Solid [S1, unofficial].
6.  **Pointer Insertion:** For unbalanced trees, insert 4-byte integer pointers from parent groups to the first child node in the group [S1, unofficial].
7.  **Flushing:** Write the compressed hierarchical data structures to the filesystem as a single binary data stream in breadth-first order [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def preprocess_tetsplat(mesh, num_fields):
    """
    Pre-processes a tetrahedral mesh for TetSplat rendering.
    
    Args:
        mesh: Partitioned tetrahedral mesh fragment.
        num_fields: Number of scalar field values to quantize.
        
    Returns:
        binary_stream: Flushed binary data stream.
    """
    # Step 1: Normalize scalar fields to [0, 1]
    normalized_fields = []
    for field in mesh.scalar_fields:
        min_val = min(field)
        max_val = max(field)
        # Avoid division by zero
        if max_val == min_val:
            normalized_fields.append([0.0] * len(field))
        else:
            normalized_fields.append([(v - min_val) / (max_val - min_val) for v in field])
            
    # Step 2: Quantize using lookup table (15 bits precision)
    # Note: Exact lookup table implementation is not specified in sources [G]
    # Assuming a standard uniform quantization to 15 bits
    quantized_fields = []
    for norm_field in normalized_fields:
        # Map [0, 1] to [0, 2^15 - 1]
        quantized = [int(v * (2**15 - 1)) for v in norm_field]
        quantized_fields.append(quantized)
        
    # Step 3: Construct Nodes
    solid_nodes = []
    shell_nodes = []
    
    for element in mesh.elements:
        if element.is_solid:
            # Solid Node: 6 bytes base + 2 bytes per additional field
            # Base 6 bytes hold first 2 fields (3 bytes each? No, 15 bits is < 2 bytes)
            # Source says "6 bytes total... two field values... each additional adds 2 bytes"
            # This implies the first two fields are packed into 6 bytes, or 6 bytes is the container.
            # Let's assume the 6 bytes is the fixed overhead for the first two fields.
            node_data = bytearray(6)
            
            # Pack first two fields into the 6-byte base
            # Source does not specify exact bit-packing within the 6 bytes for the first two fields.
            # It states "15 bits per scalar field value". 15 bits * 2 = 30 bits.
            # 6 bytes = 48 bits. So there is padding or specific packing.
            # We pack the first two 15-bit values.
            pack_15bit(node_data, 0, quantized_fields[0][element.id])
            pack_15bit(node_data, 15, quantized_fields[1][element.id])
            
            # Pack additional fields (2 bytes = 16 bits each)
            for i in range(2, num_fields):
                # Add 2 bytes to the bitstring
                node_data.extend(bytearray(2))
                # Pack the 15-bit value into the new 2 bytes (16 bits available)
                # Assuming left-aligned or standard packing
                pack_15bit(node_data, len(node_data)*8 - 16, quantized_fields[i][element.id])
                
            solid_nodes.append(node_data)
            
        elif element.is_shell:
            # Shell Node: 8 bytes (6 bytes base + 2 bytes extra)
            node_data = bytearray(8)
            
            # Pack first two fields into the 6-byte base
            pack_15bit(node_data, 0, quantized_fields[0][element.id])
            pack_15bit(node_data, 15, quantized_fields[1][element.id])
            
            # Pack additional fields
            for i in range(2, num_fields):
                node_data.extend(bytearray(2))
                pack_15bit(node_data, len(node_data)*8 - 16, quantized_fields[i][element.id])
                
            # Pack normal vector (14 bits) and cone width (2 bits) into the last 2 bytes
            # Bytes 6 and 7 (indices 6, 7)
            normal_bits = encode_normal_14bit(element.normal)
            cone_bits = encode_cone_width_2bit(element.normal_cone_width)
            
            # Combine into 16 bits
            combined = (normal_bits << 2) | cone_bits
            node_data[6] = (combined >> 8) & 0xFF
            node_data[7] = combined & 0xFF
            
            shell_nodes.append(node_data)
            
    # Step 4: Cleanup (Parallel)
    # Trim shared triangles from Shell and add to Solid
    # Update boundary labels
    cleanup_shell_and_solid(shell_nodes, solid_nodes)
    
    # Step 5: Insert Pointers for Unbalanced Trees
    # Insert 4-byte int pointer for each group of parents to first child
    solid_stream = insert_pointers(solid_nodes, tree_type='octree')
    shell_stream = insert_pointers(shell_nodes, tree_type='quadtree')
    
    # Step 6: Flush to Filesystem
    # Breadth-first order
    binary_stream = solid_stream + shell_stream
    flush_to_filesystem(binary_stream)
    
    return binary_stream

def pack_15bit(data, offset_bits, value):
    """Packs a 15-bit integer value into a byte array at a specific bit offset."""
    # Implementation details for bit-packing are not provided in sources [G]
    # Standard bit-packing logic:
    byte_idx = offset_bits // 8
    bit_idx = offset_bits % 8
    
    # This is a placeholder for the actual bit manipulation
    # which depends on endianness and specific packing scheme not detailed in [S1, unofficial]
    pass

def encode_normal_14bit(normal):
    """Encodes a normal vector into 14 bits."""
    # Implementation details not provided in sources [G]
    pass

def encode_cone_width_2bit(width):
    """Encodes normal cone width into 2 bits."""
    # Implementation details not provided in sources [G]
    pass

def cleanup_shell_and_solid(shell_nodes, solid_nodes):
    """Trims shared triangles and updates boundary labels."""
    # Implementation details not provided in sources [G]
    pass

def insert_pointers(nodes, tree_type):
    """Inserts 4-byte pointers for unbalanced tree traversal."""
    # Implementation details not provided in sources [G]
    pass
```

**Parameters:**
*   **Quantization Precision:** 15 bits per scalar field value [S1, unofficial].
*   **Pointer Size:** 4 bytes (32-bit integer) [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Memory Overhead:** The relative memory overhead of 32-bit pointers is much less for Solid nodes than for Shell nodes. Although the Solid structure is typically several orders of magnitude larger than the Shell, its average branching factor is twice as high, leading to fewer internal nodes and thus fewer pointers [S1, unofficial].
*   **Unbalanced Trees:** Pointers are required for both Shell (quadtree) and Solid (octree) because the trees are unbalanced. The 3 bits in the pointer encoding (if used for branching factor) encode numbers 0, 2, 3, 4, 5, 6, 7, and 8, as a branch is terminated once a leaf node is reached [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** The method reduces memory usage through quantization (15 bits vs 32/64 bits) and hierarchical compression. However, it incurs overhead from 4-byte pointers in unbalanced trees. The relative overhead is lower for Solid nodes due to their higher branching factor [S1, unofficial].
*   **Performance:** The breadth-first layout facilitates fast top-down multi-resolution rendering by allowing the OS `mmap` function to map the file directly, avoiding loading the entire structure into physical memory [S1, unofficial].
*   **Strengths:** Efficient storage of multiple scalar fields; fast traversal via embedded pointers; parallelizable pre-processing.
*   **Limitations:** Requires unbalanced tree traversal logic; fixed bit-packing scheme may not be optimal for all data distributions.

## Background
The method was introduced by Museth and Lombeyda in 2004 in the paper "TetSplat: Real-time Rendering and Volume Clipping of Large Tetrahedral Meshes" [S1, unofficial]. It builds on the QSplat file layout strategy, adapting it for tetrahedral meshes with multiple scalar fields. The approach allows users to interactively edit transfer functions associated with each field value to highlight interesting regions during visualization [S1, unofficial].

## Sources
[S1] TetSplat: Real-time Rendering and Volume Clipping of Large ... - Museth, https://ken.museth.org/Publications_files/Museth-Lombeyda_VIS04.pdf, fetched 2026-10-28 (unofficial)
