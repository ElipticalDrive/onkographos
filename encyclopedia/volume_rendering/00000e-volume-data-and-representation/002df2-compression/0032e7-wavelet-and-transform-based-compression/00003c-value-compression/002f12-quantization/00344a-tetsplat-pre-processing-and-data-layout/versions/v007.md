# TetSplat Pre-processing and Data Layout

Also called: TetSplat Quantization and Bitstring Packing

## Overview
TetSplat Pre-processing and Data Layout is a method for compressing scalar field values on tetrahedral meshes into a compact binary format for efficient storage and interactive visualization [S1, unofficial]. It differs from general scalar quantization by integrating field value quantization directly into a hierarchical tree structure (Solid octree and Shell quadtree) that also encodes geometric topology, using specific bit-packing schemes for field values and normal vectors [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A tetrahedral mesh partitioned across multiple CPU nodes, with scalar field values defined on the mesh.
*   **Output:** A single binary data stream containing compressed hierarchical data structures for the Shell and Solid, flushed to the filesystem in breadth-first order [S1, unofficial].

**Data Structures and Layout:**
The method maintains two hierarchical structures: the **Solid** (an octree) and the **Shell** (a quadtree) [S1, unofficial].

1.  **Solid Node Layout:**
    *   Base size: 6 bytes.
    *   Content: Two scalar field values packed into a bitstring.
    *   Expansion: Each additional field value adds 2 bytes to the bitstring [S1, unofficial].
    *   Pointers: For unbalanced trees, a 4-byte integer pointer is inserted for each group of parents to the first node of their group of children to enable fast traversal [S1, unofficial].

2.  **Shell Node Layout:**
    *   Base size: Solid node size + 2 bytes.
    *   Content: The extra 2 bytes hierarchically encode the normal vector (14 bits) and the width of the normal cone (2 bits) [S1, unofficial].
    *   Pointers: Similar to Solid, 4-byte integer pointers are inserted for parent groups to children [S1, unofficial].

**Parameters:**
*   **Quantization Precision:** 15 bits per scalar field value is sufficient for the studied datasets [S1, unofficial].
*   **Field Value Count:** Arbitrary number of field values can be selected; each adds 2 bytes to the Solid node bitstring [S1, unofficial].

**Steps:**
1.  **Normalization:** Normalize each scalar field value to the range $[0, 1]$ [S1, unofficial].
2.  **Quantization:** Quantize the normalized values using a lookup table [S1, unofficial].
3.  **Bitstring Packing:** Pack the quantized values into the Solid node bitstring. The first two values occupy the base 6 bytes; subsequent values are appended in 2-byte increments [S1, unofficial].
4.  **Shell Encoding:** For Shell nodes, encode the normal vector (14 bits) and normal cone width (2 bits) into the extra 2 bytes [S1, unofficial].
5.  **Pointer Insertion:** Insert 4-byte integer pointers for parent groups to the first child in each group to support unbalanced tree traversal [S1, unofficial].
6.  **Parallel Pre-processing:** Partition the tetrahedral mesh onto multiple CPU nodes. Each node runs its own pre-processing [S1, unofficial].
7.  **Cleanup:** A cleanup process trims shared triangles from the Shell and adds them to the Solid [S1, unofficial].
8.  **Boundary Label Update:** Update the boundary label assignment to tetrahedra and triangles in the Solid after cleanup [S1, unofficial].
9.  **File Flush:** Flush the compressed hierarchical data structures to the filesystem as a single binary data stream in breadth-first order [S1, unofficial].

**Pitfalls:**
*   **Memory Overhead:** The relative memory overhead of 32-bit pointers is much higher for the Shell than for the Solid. Although the Solid is typically larger, its average branching factor is twice as high, leading to fewer internal nodes and thus fewer pointers [S1, unofficial].

## Operating characteristics
*   **Memory Efficiency:** The method reduces memory overhead for pointers in the Solid due to its higher branching factor compared to the Shell [S1, unofficial].
*   **Storage:** Data is stored in a single binary stream, facilitating fast I/O via memory mapping (`mmap`) [S1, unofficial].
*   **Precision:** 15-bit quantization is sufficient for the studied datasets, balancing precision and storage size [S1, unofficial].

## Background
The method was introduced by Museth and Lombeyda in 2004 as part of the TetSplat system for real-time rendering and volume clipping of large tetrahedral meshes [S1, unofficial]. It builds on the QSplat file layout strategy, adapting it for tetrahedral meshes with scalar fields [S1, unofficial].

## Sources
[S1] TetSplat: Real-time Rendering and Volume Clipping of Large ... - Museth, https://ken.museth.org/Publications_files/Museth-Lombeyda_VIS04.pdf, fetched 2026-10-28 (unofficial)
