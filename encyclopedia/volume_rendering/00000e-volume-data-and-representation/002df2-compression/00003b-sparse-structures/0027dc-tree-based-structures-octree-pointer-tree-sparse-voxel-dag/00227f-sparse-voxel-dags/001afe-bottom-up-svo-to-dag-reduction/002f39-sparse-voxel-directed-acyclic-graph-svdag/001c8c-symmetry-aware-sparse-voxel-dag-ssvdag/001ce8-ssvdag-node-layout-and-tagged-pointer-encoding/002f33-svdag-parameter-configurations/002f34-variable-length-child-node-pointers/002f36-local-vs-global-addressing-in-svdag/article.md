# Local vs Global Addressing in SVDAG

Also called: Global addressing, Layer-local addressing

## Overview
This method defines two distinct schemes for interpreting node pointers within a Sparse Voxel Directed Acyclic Graph (SVDAG): **Global Addressing**, which uses a single contiguous address space for the entire data structure, and **Local Addressing**, which uses separate address spaces for each node layer [S1]. The primary distinction lies in scalability: Global Addressing limits the total size of the SVDAG by the pointer's address space capacity, whereas Local Addressing applies this limit per layer, allowing the total structure to exceed the pointer range provided individual layers fit within it [S1].

## How it works

### Inputs and Outputs
*   **Input:** A Sparse Voxel Directed Acyclic Graph (SVDAG) structure with defined node layers, child node pointers (PT) of rational lengths (8b, 16b, or 32b), and an addressing word length (8b, 16b, or 32b) [S1].
*   **Output:** A mechanism for resolving node addresses during traversal or construction, determining whether the address space is global or local [S1].

### Data Structures
The SVDAG consists of layers of nodes. Each node contains a Child Node Header Map (CHNM) and child node pointers (PT) [S1].
*   **Pointer Length ($p_{ptl}$):** The bit length of the pointer. Rational values are 8b, 16b, and 32b [S1].
*   **Addressing Word Length:** The bit length of the memory word used for addressing. Rational values are 8b, 16b, and 32b [S1].
*   **Alignment:** The CHNM is padded with reserved bits to align with the addressing word length. For example, if $p_{ptl} = 32b$, an 8b CHNM is aligned to 32b using 24 reserved bits ($r=24$) [S1].

### Numbered Steps

1.  **Determine Addressing Scheme:** Select either Global Addressing or Local Addressing based on scene size and memory constraints [S1].
2.  **Global Addressing Interpretation:**
    *   Assign a unique address to the root node.
    *   Allocate addresses for subsequent layers sequentially in a single global address space [S1].
    *   The pointer value represents the absolute address of the child node within this global space [S1].
    *   The entire SVDAG must fit within the addressable space defined by the pointer length and addressing word length [S1].
3.  **Local Addressing Interpretation:**
    *   Identify the starting address in the global address space where the encoding of a specific node layer commences [S1].
    *   The pointer value represents an offset from this starting address [S1].
    *   Addressing restarts from 0 for each new layer [S1].
    *   The limit imposed by the pointer length applies to each layer separately, not the entire structure [S1].
    *   It is sufficient for the starting address of a layer to be within the addressable space; the entire layer does not need to fit within the pointer range if the offset logic is correctly applied [S1].

### Formulas and Parameters

**Addressable Space Capacity:**
The maximum size of the addressable space depends on the pointer length ($p_{ptl}$) and the addressing word length ($w_{addr}$) [S1].

| Pointer Length ($p_{ptl}$) | Addressing Word Length ($w_{addr}$) | Addressable Space Capacity |
| :--- | :--- | :--- |
| 8b | 8b | 256 B |
| 8b | 16b | 512 B |
| 8b | 32b | 1024 B |
| 16b | 16b | 128 KB |
| 16b | 32b | 256 KB |
| 32b | 32b | 16 GB |

*Note: The source table [S1] lists specific combinations. For 8b pointers, capacities are 256B, 512B, and 1024B for 8b, 16b, and 32b words respectively. For 16b pointers, capacities are 128KB and 256KB for 16b and 32b words. For 32b pointers, the capacity is 16GB for 32b words.*

**Pointer Address Space Limits:**
*   8b pointer: $2^8 = 256$ addresses [S1].
*   16b pointer: $2^{16} = 65,536$ addresses [S1].
*   32b pointer: $2^{32} = 4,294,967,296$ addresses [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def resolve_address(node_layer_start_address, pointer_value, addressing_mode):
    """
    Resolves the absolute address of a child node.
    
    Args:
        node_layer_start_address (int): The global address where the current node layer begins.
        pointer_value (int): The value stored in the child node pointer (PT).
        addressing_mode (str): 'GLOBAL' or 'LOCAL'.
        
    Returns:
        int: The absolute global address of the child node.
    """
    if addressing_mode == 'GLOBAL':
        # In global addressing, the pointer value is the absolute address
        return pointer_value
    elif addressing_mode == 'LOCAL':
        # In local addressing, the pointer value is an offset from the layer start
        return node_layer_start_address + pointer_value
    else:
        raise ValueError("Invalid addressing mode")

def check_feasibility_global(total_nodes, pointer_capacity):
    """
    Checks if the entire SVDAG fits within the global address space.
    
    Args:
        total_nodes (int): Total number of nodes in the SVDAG.
        pointer_capacity (int): Maximum number of addresses supported by the pointer length.
        
    Returns:
        bool: True if feasible, False otherwise.
    """
    return total_nodes <= pointer_capacity

def check_feasibility_local(max_layer_nodes, pointer_capacity):
    """
    Checks if the largest node layer fits within the local address space.
    
    Args:
        max_layer_nodes (int): The number of nodes in the largest layer.
        pointer_capacity (int): Maximum number of addresses supported by the pointer length.
        
    Returns:
        bool: True if feasible, False otherwise.
    """
    return max_layer_nodes <= pointer_capacity
```

### Edge Cases and Pitfalls
*   **Pointer Length Selection:** An 8b pointer is considered impractical even for very small 3D scenes [S1].
*   **Word Length Correlation:** The addressing word length should correlate with the pointer length. For example, 32b pointers should use 32b addressing words, and 8b pointers should use 8b addressing words [S1].
*   **Layer Capacity:** In Local Addressing, the determining factor for feasibility is whether the largest node layer fits within the local address space, not the total number of nodes [S1].

## Subtopics

### Variable-Length Child Node Pointers
A technique where child node pointers have varying bit lengths (8b, 16b, or 32b) to optimize address space usage based on scene size and detail requirements. It builds on [Variable-Length Child Node Pointers](5948-variable-length-child-node-pointers.md).

## Operating characteristics

*   **Global Addressing:**
    *   **Limitation:** The entire data structure is limited by the pointer's address space capacity (e.g., 4GB for 32b pointers) [S1].
    *   **Use Case:** Suitable for smaller scenes or when the total number of nodes is known to fit within the pointer range [S1].
*   **Local Addressing:**
    *   **Advantage:** The limit applies to each layer separately, allowing larger total structures if individual layers fit within the pointer range [S1].
    *   **Use Case:** Suitable for large scenes where the total number of nodes exceeds the pointer capacity, but no single layer exceeds the local address space limit [S1].
*   **Complexity:** Not given in the sources.
*   **Memory:** The addressable space capacity is determined by the pointer length and addressing word length, as detailed in the formulas section [S1].

## Background

*   **Origin:** The concepts of Global and Local Addressing in SVDAGs are described in the paper "Parameter Optimization in Sparse Voxel DAGs for Efficient Geometry Representation of Voxelized 3D Scenes" by B. Madoš et al. [S1].
*   **History:** The paper was published in 2025 [S1].
*   **Usage:** These addressing schemes are used in the context of Sparse Voxel Directed Acyclic Graphs (SVDAGs) for efficient geometry representation of voxelized 3D scenes [S1].

## Sources

[S1] Parameter Optimization in Sparse Voxel DAGs for Efficient Geometry Representation of Voxelized 3D Scenes, https://doi.org/10.12700/aph.22.7.2025.7.20, fetched 2026-10-27
