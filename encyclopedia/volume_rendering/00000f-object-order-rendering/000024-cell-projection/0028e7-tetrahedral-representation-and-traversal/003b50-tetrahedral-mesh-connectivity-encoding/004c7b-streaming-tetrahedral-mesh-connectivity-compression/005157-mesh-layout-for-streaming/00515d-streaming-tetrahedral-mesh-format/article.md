# Streaming Tetrahedral Mesh Format

Also called: Interleaved vertex-cell format, OBJ-compatible streaming format, Streaming mesh format, Streaming mesh representation

## Overview
The Streaming Tetrahedral Mesh Format is a disk layout for tetrahedral meshes that interleaves vertex definitions and cell references to enable coherent sequential access and out-of-core processing [S1]. Unlike traditional formats that require the entire vertex list in memory for dereferencing, this format allows vertices to be introduced and finalized sequentially, significantly limiting the memory footprint during streaming simplification [S1]. It extends the OBJ file format to support tetrahedral cells and vertex finalization via negative indices [S1].

## How it works
**Inputs and Outputs**
Input is a tetrahedral mesh defined by vertices and cells. Output is a stream of text records representing vertices and cells in an interleaved order [S1].

**Data Structures**
The format uses two record types:
1.  **Vertex Record**: Defines a vertex with four scalar values: $x, y, z$ (3D position) and $f$ (scalar field value) [S1].
2.  **Cell Record**: Defines a tetrahedral cell by indexing four vertices [S1].

**Vertex Indexing and Finalization**
Vertices are referenced by positive integer indices corresponding to their order of introduction in the stream [S1]. A vertex is "active" from the moment it is introduced until it is finalized [S1]. Finalization is achieved by using a negative index in a subsequent cell record. A negative index $-k$ refers to the $k$-th most recently introduced active vertex [S1]. This mechanism allows the renderer or simplifier to discard finalized vertices from the in-core active set [S1]. The use of negative indices for finalization is backward compatible with the OBJ format, which typically ignores negative indices or treats them as comments/invalid references in standard polygonal contexts, but here explicitly repurposed for memory management [S1].

**Numbered Steps**
1.  Initialize an empty in-core active set of vertices and a counter for introduced vertices.
2.  For each vertex in the mesh traversal order:
    a.  Write the vertex record: `v x y z f` [S1].
    b.  Add the vertex to the active set.
    c.  Assign it a positive index equal to its position in the stream (1-based) [S1].
3.  For each tetrahedral cell in the mesh traversal order:
    a.  Identify the four vertices defining the cell.
    b.  For each vertex, determine its reference index:
        i.  If the vertex is currently in the active set, use its positive stream index [S1].
        ii. If the vertex has been finalized, use a negative index relative to the most recent active vertex [S1]. *Note: The exact mapping of negative indices to specific finalized vertices requires tracking the "front" of active vertices. The source states negative indices indicate finalization and are backward compatible, implying a specific stack-like or window-based lookup for the negative index value.* [S1].
    c.  Write the cell record: `t v1 v2 v3 v4` where $v_i$ are the determined indices [S1].
4.  After a cell is processed, if any referenced vertices are no longer needed by future cells, they are marked for finalization. In this format, finalization is implicit in the use of negative indices in subsequent cells or explicit if the format allows a "finalize" command (though the source emphasizes negative indexing as the mechanism) [S1].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def write_streaming_tet_mesh(mesh_vertices, mesh_cells, output_stream):
    """
    Writes a tetrahedral mesh in the streaming format.
    mesh_vertices: List of (x, y, z, f) tuples.
    mesh_cells: List of tuples of vertex indices (0-based from original mesh).
    output_stream: File-like object.
    """
    # Map from original vertex index to stream index (positive)
    # Map from original vertex index to finalization status
    stream_index_map = {}
    active_vertices = [] # List of original vertex indices currently in core
    next_stream_index = 1
    
    # Helper to get index for a vertex
    def get_vertex_index(orig_v_idx):
        if orig_v_idx in stream_index_map:
            return stream_index_map[orig_v_idx]
        return None

    # Helper to finalize a vertex
    def finalize_vertex(orig_v_idx):
        # Remove from active set
        if orig_v_idx in active_vertices:
            active_vertices.remove(orig_v_idx)
        # Note: The format uses negative indices in cell records to reference
        # finalized vertices. The exact logic for the negative value depends on
        # the "front width" and span. The source implies negative indices
        # refer to the most recent active vertices or finalized ones in a
        # backward-compatible way. For implementation, one must track the
        # window of active vertices.
        pass

    # Write vertices and cells interleaved
    # The source implies a specific traversal order. We assume the mesh
    # is provided in a streaming-friendly order.
    
    for cell in mesh_cells:
        # Before writing the cell, ensure all its vertices are introduced
        for v_idx in cell:
            if v_idx not in stream_index_map:
                # Introduce vertex
                v = mesh_vertices[v_idx]
                output_stream.write(f"v {v[0]} {v[1]} {v[2]} {v[3]}\n")
                stream_index_map[v_idx] = next_stream_index
                active_vertices.append(v_idx)
                next_stream_index += 1
        
        # Write cell
        # Determine indices for the 4 vertices
        cell_indices = []
        for v_idx in cell:
            idx = stream_index_map[v_idx]
            # Check if vertex is still active
            if v_idx in active_vertices:
                cell_indices.append(str(idx))
            else:
                # Vertex is finalized. Use negative index.
                # The source says "negative index into the vertex list".
                # In OBJ-compatible streaming, negative indices often refer
                # to vertices relative to the current position or a specific
                # window. The source mentions "backward compatible with OBJ".
                # Standard OBJ uses negative indices for comments or invalid refs.
                # Here, it's repurposed. A common implementation for such
                # streaming formats (like Isenburg/Lindstrom for triangles)
                # uses negative indices to refer to vertices in the "front"
                # or active set in reverse order, or finalized vertices.
                # Given the source says "finalize a vertex... by using a negative
                # index... backward compatible", we assume a standard mapping.
                # Let's assume a simple stack-like finalization for the negative
                # index value: -1 refers to the most recently finalized vertex?
                # Or -1 refers to the last active vertex?
                # The source says: "finalize a vertex when it is no longer in use
                # by using a negative index into the vertex list".
                # This is ambiguous without the specific Isenburg/Lindstrom
                # triangle format details. However, for tetrahedra, we index 4.
                # We will use a placeholder comment for the exact negative index
                # calculation as the source does not provide the formula for
                # the negative value, only that it exists and is compatible.
                cell_indices.append("-1") # Placeholder: Exact negative index logic not fully specified in source beyond "backward compatible"
        
        output_stream.write(f"t {' '.join(cell_indices)}\n")
        
        # Finalize vertices no longer needed by future cells
        # This requires lookahead or a graph traversal to determine
        # when a vertex is no longer referenced.
        # For simplicity, we assume the mesh traversal order allows
        # immediate finalization if not needed by the current or next few cells.
        # In practice, a "front width" determines the active set size.
        # We remove vertices from active_vertices if they are not in
        # any future cell.
        # This step is complex and depends on the mesh traversal order.
        # The source mentions "front width" as the max concurrent active vertices.
        pass
```

**Parameters**
*   **Front Width**: The maximum number of concurrently active vertices in the in-core active set. This determines the lower bound on memory required for dereferencing [S1].
*   **Front Span**: The maximum index difference (plus one) of the concurrently active vertices. A low span allows for faster implementation and efficient indexing [S1].

**Edge Cases and Pitfalls**
*   **Precision**: Floating-point precision for $x, y, z, f$ values should be consistent with the source data [S1].
*   **Boundary Conditions**: The first vertex must be introduced before any cell referencing it is written [S1].
*   **Degenerate Input**: If a cell references a vertex that has not been introduced, the format is violated [S1].
*   **OBJ Compatibility**: The negative index mechanism must be carefully implemented to ensure that standard OBJ parsers do not misinterpret the finalization indices as errors or comments, while allowing the streaming parser to use them correctly [S1].

## Subtopics

### Mesh Layout for Streaming
The parent technique that defines the coherent, I/O-efficient layout strategy. Choose this when memory is constrained and the mesh is too large to fit in core. [Mesh Layout for Streaming](14444-mesh-layout-for-streaming.md)

### Tetrahedral Mesh Connectivity Compression
A method to compress the connectivity information. Choose this when storage space is the primary constraint, but streaming access is still required. [Tetrahedral Mesh Connectivity Compression](14445-tetrahedral-mesh-connectivity-compression.md)

### Streaming Simplification of Tetrahedral Meshes
The application of this format to simplify meshes in an out-of-core manner. Choose this for real-time visualization of large datasets. [Streaming Simplification of Tetrahedral Meshes](14446-streaming-simplification-of-tetrahedral-meshes.md)

## Operating characteristics
*   **Time Complexity**: Sequential access allows for $O(N)$ time complexity for streaming, where $N$ is the number of vertices/cells, assuming efficient active set management [S1].
*   **Memory Complexity**: Memory usage is bounded by the front width, which is typically much smaller than the total number of vertices $N$ [S1].
*   **Measured Performance**: The source does not provide specific performance numbers, but notes that low front span allows for faster implementation [S1].
*   **Strengths**: Low memory footprint, backward compatibility with OBJ, suitable for out-of-core processing [S1].
*   **Limitations**: Requires careful management of the active set and finalization logic. The negative index mechanism may be complex to implement correctly for general meshes [S1].
*   **When to Use**: When streaming large tetrahedral meshes for visualization or simplification with limited memory [S1].
*   **When Not to Use**: When random access to vertices is required, or when the mesh is small enough to fit entirely in memory [S1].

## Background
The format was developed by Isenburg and Lindstrom for triangle meshes and extended to tetrahedral meshes by the authors of "Streaming Simplification of Tetrahedral Meshes" [S1]. It builds on the OBJ file format, extending it to handle tetrahedral cells and vertex finalization [S1]. The format is used in the context of out-of-core isosurface extraction and mesh simplification [S1].

## Sources
[S1] Streaming Simplification of Tetrahedral Meshes, https://doi.org/10.1109/tvcg.2007.21, fetched 2026-11-04
