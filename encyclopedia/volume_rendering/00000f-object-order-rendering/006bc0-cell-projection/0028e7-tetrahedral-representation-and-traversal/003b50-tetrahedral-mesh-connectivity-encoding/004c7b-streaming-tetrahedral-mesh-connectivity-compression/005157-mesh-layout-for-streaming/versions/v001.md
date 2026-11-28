# Mesh Layout for Streaming
Also called: Breadth-First Layout, Stream-Efficient Layout

## Overview
Mesh Layout for Streaming is a technique to organize tetrahedral meshes on disk in a coherent, I/O-efficient format to minimize the memory footprint during streaming simplification [S1]. It differs from spatial sorting by prioritizing topological contiguity to reduce the "front width" and "span" of the active mesh portion, enabling the use of fixed-size circular arrays instead of hash tables [S1].

## How it works
**Inputs and Outputs:**
- Input: A tetrahedral mesh with arbitrary initial connectivity.
- Output: A re-ordered sequence of tetrahedra written to disk in a stream-efficient order.

**Data Structures:**
- The method relies on a delay buffer to manage active vertices. The layout determines the size of this buffer.
- The output is a linear stream of tetrahedra indices.

**Steps:**
1. **Graph Construction:** Construct the dual graph of the tetrahedral mesh, where nodes represent tetrahedra and edges represent shared faces [S1].
2. **Traversal:** Perform a Breadth-First Search (BFS) starting from an arbitrary boundary tetrahedron [S1].
3. **Ordering:** Assign tetrahedra to the output stream in the order they are visited by the BFS [S1].
4. **Stream Preservation:** Ensure the stream order between input and output is preserved where possible to maintain spatial locality for the simplification algorithm [S1].

**Parameters:**
- **Buffer Size:** The size of the circular array or hash table required is determined by the "span" of the layout. The BFS layout minimizes this span, allowing fixed-size arrays [S1].
- **Typical Performance:** Laying out a mesh of 14 million tetrahedra takes approximately 40 minutes [S1].

**Edge Cases and Pitfalls:**
- **Collisions:** If the buffer size is smaller than the span of the layout, hash collisions occur. However, a simple modulo hash function suffices, avoiding complex linked-list structures [S1].
- **Accuracy:** For very large datasets (e.g., SF1), parts of the stream may not be simplified accurately due to memory constraints; increasing memory improves quality [S1].

## Subtopics

### Spectral Sequencing Layout
This variant finds the first nontrivial eigenvector (the Fiedler vector) of the mesh’s Laplacian matrix to generate the ordering [S1]. It uses an out-of-core algorithm extended from triangle meshes to handle tetrahedra [S1]. It typically results in the lowest width, ideal for minimizing memory consumption, and works particularly well for curvy triangle meshes, though tetrahedral meshes are generally less curvy and compact [S1].

### Topological Layout
This approach starts at a vertex on the boundary and grows out to neighboring vertices in a contiguous manner [S1]. It is another method for creating a topological layout to ensure coherent access [S1].

## Operating characteristics
- **Time Complexity:** The layout is a one-time operation. For a dataset of 14 million tetrahedra, it requires about 40 minutes [S1].
- **Memory Footprint:** The method enables simplification of 14 million tetrahedra using only 20 MB RAM [S1].
- **Metrics:** Compared to the algorithm of Wu and Kobbelt, this method achieves lower width (9% vs 59% on the Fighter dataset) and span (45% vs 98%) without reducing approximation quality [S1].
- **Strengths:** Significantly reduces memory footprint; allows use of fixed-size circular arrays; improves simplification speed by an order of magnitude over initial implementations [S1].
- **Limitations:** Width and span are typically larger than spatial sorting [S1].

## Background
This method was presented in the context of streaming simplification of tetrahedral meshes [S1]. It improves upon the algorithm of Wu and Kobbelt by preserving stream order and optimizing data structures for streaming [S1]. The spectral sequencing heuristic was originally shown by Isenburg and Lindstrom to be effective for low-width layouts [S1].

## Sources
[S1] Streaming Simplification of Tetrahedral Meshes, https://doi.org/10.1109/tvcg.2007.21, fetched 2026-11-04
