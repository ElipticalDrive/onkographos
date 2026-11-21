# MPI-based Volume Partitioning

Also called: Chunk-based MPI parallelism, Spatial Division of Uniform Resolution Data

## Overview
A data-parallel strategy where a volumetric dataset is partitioned into contiguous chunks of slices, with each MPI process reading and rendering only its assigned portion [S1][S2, unofficial]. It differs from object-space decomposition by partitioning the data grid directly rather than clustering geometric primitives, and from ShardedVolume by relying on static MPI process assignment rather than dynamic sharding [S1][S2, unofficial]. This method solves the problem of efficient I/O and parallel rendering for uniform-resolution volumes by ensuring each process accesses contiguous memory regions, minimizing seek overhead [S2, unofficial].

## How it works
**Inputs:** A volumetric dataset stored as a sequence of 2D slices (e.g., along the z-axis); the number of MPI processes $P$.
**Outputs:** A set of scattered binary result files (one per process) containing partial renderings or mesh data; optionally, a sewed final image or mesh.

**Data Structures:**
*   **Slice Index:** Integer identifying the slice position in the global volume.
*   **Chunk:** A contiguous range of slice indices $[start, end)$ assigned to a single process.
*   **Triangle Table (for isosurface extraction):** A list of triangles defined by vertex indices. Each process maintains a local vertex table.

**Algorithm Steps:**

1.  **Calculate Slice Assignment:**
    Determine the number of slices $S$ and processors $P$. Assign slices to processors $i \in [0, P-1]$ such that the difference in slice count between any two processors is at most one. The number of slices $s_i$ for processor $i$ is calculated as:
    $$
    s_i = \begin{cases} 
    \lfloor S/P \rfloor + 1 & \text{if } i < S \pmod P \\
    \lfloor S/P \rfloor & \text{otherwise}
    \end{cases}
    $$
    The starting slice index $start_i$ for processor $i$ is:
    $$
    start_i = \sum_{j=0}^{i-1} s_j
    $$
    [S2, unofficial]

2.  **Handle Boundary Overlap:**
    To prevent boundary artifacts (e.g., black stripes) caused by missing opacity/color calculations at division boundaries, assign a small overlap of data to adjacent chunks [S2, unofficial].
    *   Let $O$ be the overlap size (number of extra slices).
    *   Processor $i$ reads slices from $start_i$ to $end_i + O$ (where $end_i = start_i + s_i$).
    *   Processor $0$ reads from $start_0$ to $end_0 + O$.
    *   Processor $P-1$ reads from $start_{P-1}$ to $end_{P-1}$.
    *   Intermediate processors $i$ read from $start_i$ to $end_i + O$.
    *   *Note:* The overlap ensures that ray casting or marching cubes operations near the boundary have valid neighbor data. [S2, unofficial]

3.  **Read and Render:**
    Each MPI process $i$ seeks to the file offset corresponding to its first slice (including overlap) and reads its assigned chunk. The process renders its portion of the volume.
    *   For ray casting: Compositing is performed locally for the assigned slices.
    *   For isosurface extraction (Marching Cubes): Generate local triangles and vertices. [S1][S2, unofficial]

4.  **Write Local Results:**
    Write the rendered image or mesh data to a scattered binary file specific to process $i$.
    *   If rendering an image: Write the pixel buffer for the assigned slices.
    *   If extracting isosurfaces: Write the local triangle table and vertex table. [S1][S2, unofficial]

5.  **Sewing (Optional for Isosurfaces):**
    If the output is a mesh, combine the partial triangle tables from all processes.
    *   Concatenate the vertex tables from all processes.
    *   Update the vertex indices in each process's triangle table to account for the offset in the global vertex table.
    *   Remove duplicate vertices if necessary (though the source notes mention updating indices rather than merging duplicates explicitly in the sewing step, it implies index remapping) [S1].
    *   *Note:* Sewing is optional if the scattered files are used as input for subsequent parallel processing steps [S1].

6.  **Compositing (Optional for Images):**
    If the output is a final image, composite the scattered image files.
    *   Remove the extra/duplicated overlap slices from the edges of each process's image to prevent stretching.
    *   Combine the remaining slices in order. [S2, unofficial]

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def mpi_volume_partitioning(num_slices, num_procs, slice_size_bytes, overlap_slices):
    """
    Simulates the slice assignment logic for MPI volume partitioning.
    """
    # Calculate base slices per processor and remainder
    base_slices = num_slices // num_procs
    remainder = num_slices % num_procs
    
    # Determine slice counts for each processor
    slice_counts = []
    for i in range(num_procs):
        if i < remainder:
            slice_counts.append(base_slices + 1)
        else:
            slice_counts.append(base_slices)
            
    # Calculate start indices
    start_indices = [0] * num_procs
    current_start = 0
    for i in range(num_procs):
        start_indices[i] = current_start
        current_start += slice_counts[i]
        
    # Assign chunks with overlap
    chunks = []
    for i in range(num_procs):
        start = start_indices[i]
        end = start + slice_counts[i]
        
        # Apply overlap
        # First processor overlaps with nothing before, but needs overlap after
        # Last processor overlaps with previous, but no overlap after
        # Middle processors overlap both sides? 
        # Source [S2, unofficial] says "small overlap of data among adjacent datasets".
        # Typically, this means each chunk extends into the neighbor's territory.
        # To simplify for implementation: each chunk reads its base slices + overlap.
        # The overlap is removed during compositing.
        
        # Let's assume overlap is added to the end of each chunk, 
        # except the last one which doesn't need overlap after it.
        # And the first one doesn't need overlap before it (or it's handled by boundary conditions).
        # However, to avoid artifacts, the boundary calculation needs data from the neighbor.
        # A common pattern: Chunk i reads [start_i, end_i + overlap).
        # But Chunk i+1 reads [start_{i+1}, end_{i+1} + overlap).
        # The region [end_i, start_{i+1} + overlap) is duplicated.
        
        read_start = start
        read_end = end + overlap_slices
        
        # Clamp read_end to total slices for the last processor
        if i == num_procs - 1:
            read_end = min(read_end, num_slices)
            
        chunks.append({
            'proc_id': i,
            'read_start': read_start,
            'read_end': read_end,
            'base_start': start,
            'base_end': end
        })
        
    return chunks

def sew_isosurface_meshes(partial_meshes):
    """
    Combines partial triangle tables from MPI processes.
    """
    global_vertices = []
    global_triangles = []
    vertex_offset = 0
    
    for mesh in partial_meshes:
        # Append local vertices to global list
        global_vertices.extend(mesh.vertices)
        
        # Update triangle indices
        for tri in mesh.triangles:
            # tri is a tuple/list of 3 vertex indices
            new_indices = [v + vertex_offset for v in tri]
            global_triangles.append(new_indices)
            
        vertex_offset += len(mesh.vertices)
        
    return global_vertices, global_triangles
```

**Parameters:**
*   **Overlap Size:** Not given in the sources. Must be chosen to cover the stencil size of the rendering/extraction algorithm (e.g., 1-2 voxels for Marching Cubes).
*   **Slice Order:** Must match storage order for efficient seeking [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Boundary Artifacts:** Naive non-overlapping division causes black stripes or missing geometry at boundaries because opacity/color calculations are missing at the division line [S2, unofficial].
*   **Image Stretching:** If overlap slices are not removed before final compositing, the image will be stretched [S2, unofficial].
*   **Uneven Workload:** For uniform resolution data, workload is proportional to slice count. If $S$ is not divisible by $P$, some processors get $\lfloor S/P \rfloor + 1$ slices and others $\lfloor S/P \rfloor$ [S2, unofficial].
*   **Adaptive Resolution:** This method is designed for uniform resolution data. For adaptive-resolution data, workload balancing requires different strategies (e.g., dividing by computational cost rather than slice count) [S2, unofficial].

## Subtopics

### Object-space decomposition with local clustering
Partitions the scene geometry rather than the data grid. Better for sparse or complex geometry. [Link](4156-object-space-decomposition-with-local-clustering.md)

### ShardedVolume
Uses dynamic sharding and potentially more complex data structures for distributed storage. Better for very large, dynamic datasets. [Link](12593-shardedvolume.md)

## Operating characteristics
*   **Time Complexity:** $O(S/P)$ for reading and rendering, where $S$ is the number of slices and $P$ is the number of processors. Sewing is $O(V + T)$ where $V$ is total vertices and $T$ is total triangles [S1][S2, unofficial].
*   **Memory Complexity:** $O(S_{chunk} \times \text{slice\_size})$ per process. Each process only loads its assigned chunk [S1].
*   **Scaling:** Linear scaling is expected if I/O and computation are balanced. Efficiency depends on the uniformity of the data and the overhead of sewing/compositing [S2, unofficial].
*   **Strengths:** Simple implementation; efficient I/O due to contiguous slice access; no need for complex load balancing for uniform data [S2, unofficial].
*   **Limitations:** Not suitable for adaptive-resolution data without modification; requires post-processing (sewing/compositing) to combine results; boundary artifacts if overlap is not handled [S2, unofficial].
*   **When to use:** Uniform-resolution volumes where I/O efficiency is critical and the number of processors is known and static [S1][S2, unofficial].
*   **When not to use:** Adaptive-resolution data; scenes with highly non-uniform computational cost per slice [S2, unofficial].

## Background
The method was described in the context of parallel isosurface extraction on GPU clusters [S1] and volume visualization of time-varying data [S2, unofficial]. It is a standard approach for data-parallel volume rendering where the data is partitioned spatially along one axis.

## Sources
[S1] A parallel isosurface extraction component for visualization pipelines executing on GPU clusters, https://doi.org/10.1016/j.cam.2014.05.019, fetched 2026-10-27
[S2] Volume visualization of time-varying data using parallel, multiresolution and adaptive-resolution techniques, https://scholars.unh.edu/cgi/viewcontent.cgi?article=1245&context=thesis, fetched 2026-10-27 (unofficial)
