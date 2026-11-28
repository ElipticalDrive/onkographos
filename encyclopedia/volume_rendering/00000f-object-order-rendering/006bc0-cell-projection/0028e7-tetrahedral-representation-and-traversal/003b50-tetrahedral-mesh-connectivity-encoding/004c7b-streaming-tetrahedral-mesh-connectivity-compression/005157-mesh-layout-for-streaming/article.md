# Mesh Layout for Streaming
Also called: Breadth-First Layout, Stream-Efficient Layout

## Overview
Mesh Layout for Streaming is a technique to organize tetrahedral meshes on disk in a coherent, I/O-efficient format to minimize the memory footprint during streaming simplification [S1]. It builds on [Streaming Tetrahedral Mesh Connectivity Compression](13244-streaming-tetrahedral-mesh-connectivity-compression.md) and differs from spatial sorting by prioritizing topological contiguity to reduce the front width and span of the active mesh portion [S1].

## How it works
**Inputs and Outputs:**
- Input: A tetrahedral mesh with arbitrary initial connectivity [S1].
- Output: A re-ordered sequence of tetrahedra written to disk in a stream-efficient order [S1].

**Data Structures:**
- The method uses a fixed-size circular array or hash table to track active vertices, with the required size determined by the layout's span [S1].
- The output is a linear stream of tetrahedra indices [S1].

**Steps:**
1. Construct the adjacency graph of the tetrahedral mesh [S1].
2. Initiate a Breadth-First Search (BFS) starting from an arbitrary boundary tetrahedron [S1].
3. Traverse the graph and assign tetrahedra to the output stream in visitation order [S1].
4. Preserve the stream order between input and output to maintain spatial locality for the simplification algorithm [S1].

**Parameters:**
- Buffer size is determined by the span of the layout, which the BFS approach minimizes to allow fixed-size arrays [S1].
- Typical performance requires approximately 40 minutes to layout a mesh of 14 million tetrahedra [S1].

**Edge Cases and Pitfalls:**
- Collisions occur if the buffer size is smaller than the span, but a simple modulo hash function resolves them without linked lists [S1].
- For very large datasets like SF1, memory constraints may cause inaccurate simplification in certain stream parts, but increasing memory slightly greatly improves quality [S1].

## Subtopics

### [Streaming Tetrahedral Mesh Format](14453-streaming-tetrahedral-mesh-format.md)
This disk layout interleaves vertex definitions and cell references to enable coherent sequential access and out-of-core processing. It extends the OBJ file format to support tetrahedral cells and vertex finalization via negative indices. It is chosen when out-of-core streaming simplification is required, as it limits memory footprint compared to traditional formats that require the entire vertex list in memory.

### Spectral Sequencing Layout
This variant uses the first nontrivial eigenvector of the mesh Laplacian matrix to generate the ordering [S1]. It is chosen when the lowest possible width is critical for minimizing memory consumption, though it is less effective on compact tetrahedral meshes than on curvy triangle meshes [S1].

### Topological Layout
This approach starts at a boundary vertex and grows contiguously to neighboring vertices [S1]. It is selected when a simple topological growth strategy is preferred over spectral or breadth-first methods [S1].

## Operating characteristics
- Time complexity is a one-time operation, requiring about 40 minutes for 14 million tetrahedra [S1].
- The method enables simplification of 14 million tetrahedra using only 20 MB RAM [S1].
- Compared to Wu and Kobbelt's algorithm, it achieves lower width (9% vs 59% on the Fighter dataset) and span (45% vs 98%) without reducing approximation quality [S1].
- Strengths include a significantly reduced memory footprint, support for fixed-size circular arrays, and an order-of-magnitude speedup over initial Garland and Zhou implementations [S1].
- Limitations include front width and span being typically much larger than spatial sorting [S1].

## Background
This method was presented in the context of streaming simplification of tetrahedral meshes [S1]. It improves upon the algorithm of Wu and Kobbelt by preserving stream order and optimizing data structures for streaming [S1]. The spectral sequencing heuristic was originally shown by Isenburg and Lindstrom to be effective for low-width layouts [S1].

## Sources
[S1] Streaming Simplification of Tetrahedral Meshes, https://doi.org/10.1109/tvcg.2007.21, fetched 2026-11-04
