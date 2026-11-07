# MPI-based Volume Partitioning

Also called: Chunk-based MPI parallelism, Spatial Division of Uniform Resolution Data

## Overview
MPI-based Volume Partitioning is a data-parallel strategy where a volume dataset is divided into spatial chunks (typically contiguous slices) and assigned to distinct MPI processes, each of which reads only its assigned portion to render or process the data [S1][S2, unofficial]. It is distinguished from sibling methods by its strict separation of I/O and computation per process, avoiding inter-process communication during the rendering phase by relying on post-processing stitching or sewing of independent results [S1][S2, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A single volume data file (representing one time step or static volume) and the number of MPI processes $n$ [S2, unofficial].
- Output: A set of scattered binary files, one per process, containing partial results (e.g., images or mesh data) [S1].

**Data Structures:**
- The global domain is partitioned along the $z$-axis into chunks of contiguous slices [S1][S2, unofficial].
- Each process maintains a local buffer for its assigned slices [S2, unofficial].
- For isosurface extraction, each process generates a local vertex table and triangle table [S1].

**Algorithm Steps:**
1. **Partition Assignment:** Determine the number of slices assigned to each processor. Let $S$ be the total number of slices and $n$ be the number of processors. The number of slices $s_i$ assigned to processor $i$ (where $i=0 \dots n-1$) is calculated as:
   $$
   s_i = \begin{cases} 
   \lfloor S/n \rfloor + 1 & \text{if } i < S \pmod n \\
   \lfloor S/n \rfloor & \text{if } i \ge S \pmod n 
   \end{cases}
   $$
   This ensures the difference in slice counts between any two processors is at most one [S2, unofficial].
   
2. **Data Access:** Each process $i$ calculates the starting byte offset in the input file corresponding to its first assigned slice. Since slice order matches storage order, this allows efficient seeking [S2, unofficial]. The process reads only its assigned slices into memory [S1].

3. **Rendering/Processing:** Each process independently performs the volume rendering or isosurface extraction on its local data chunk [S1][S2, unofficial].
   - *Note on Overlap:* To prevent boundary artifacts (e.g., black stripes in compositing or discontinuities in isosurfaces), a small overlap of data may be included among adjacent datasets. These extra/duplicated slices are used for boundary calculations but must be removed before final compositing to prevent image stretching [S2, unofficial].

4. **Output Generation:** Each process writes its result to a separate binary file [S1].
   - If generating images, the slice order in the output must match the storage order to allow efficient compositing without additional mappings [S2, unofficial].
   - If generating meshes (isosurfaces), each process outputs a partial triangle table and vertex table [S1].

5. **Sewing (Optional):** If a single unified result is required, a post-processing step combines the scattered files [S1].
   - For images: Compositing is performed by combining the partial images. If overlap was used, the duplicated boundary slices are stripped before compositing [S2, unofficial].
   - For meshes: The triangle tables are concatenated. The vertex indices in the triangles must be updated to account for the union of vertex tables from all processes (i.e., adding an offset to vertex indices for processes $i > 0$) [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def mpi_volume_partitioning(total_slices, num_processors, file_path, output_dir):
    """
    Simulates the partitioning logic for a single MPI rank.
    In a real MPI environment, this runs in parallel with rank-specific data.
    """
    rank = 0 # Placeholder for MPI rank
    size = num_processors # Placeholder for MPI size
    
    # 1. Calculate slice assignment
    base_slices = total_slices // size
    remainder = total_slices % size
    
    if rank < remainder:
        num_slices = base_slices + 1
        start_slice_index = rank * (base_slices + 1)
    else:
        num_slices = base_slices
        start_slice_index = remainder * (base_slices + 1) + (rank - remainder) * base_slices
        
    # 2. Read data
    # Assuming file structure: contiguous slices. 
    # In practice, calculate byte offset based on slice size and header.
    local_data = read_slices_from_file(file_path, start_slice_index, num_slices)
    
    # 3. Process data
    # Example: Render or extract isosurface
    local_result = process_volume(local_data)
    
    # 4. Write output
    output_filename = f"{output_dir}/rank_{rank}.bin"
    write_binary_file(output_filename, local_result)
    
    return local_result, start_slice_index, num_slices

def sew_mesh_results(process_results, num_processors):
    """
    Combines partial mesh results from MPI processes.
    process_results: list of (vertices, triangles) for each rank.
    """
    all_vertices = []
    all_triangles = []
    vertex_offset = 0
    
    for rank in range(num_processors):
        vertices, triangles = process_results[rank]
        all_vertices.extend(vertices)
        
        # Update triangle indices to reflect global vertex table
        modified_triangles = []
        for tri in triangles:
            # tri is a tuple/list of 3 vertex indices
            new_tri = [v + vertex_offset for v in tri]
            modified_triangles.append(new_tri)
        
        all_triangles.extend(modified_triangles)
        vertex_offset += len(vertices)
        
    return all_vertices, all_triangles
```

**Parameters:**
- $S$: Total number of slices.
- $n$: Number of MPI processes.
- $w_{overlap}$: Width of overlap region (if used). Not given in the sources as a specific value, but described as "small" [S2, unofficial].

**Edge Cases and Pitfalls:**
- **Boundary Artifacts:** Naive non-overlapping spatial division causes artifacts (e.g., black stripes) during compositing because opacity/color calculations are missing at the division boundaries [S2, unofficial].
- **Load Imbalance:** If the number of slices is not evenly divisible by the number of processors, the load is not perfectly balanced. The algorithm ensures the difference is at most one slice [S2, unofficial].
- **Image Stretching:** If overlap slices are not removed before compositing, the final image may be stretched due to the addition of extra slices [S2, unofficial].

## Operating characteristics
- **Time Complexity:** $O(S/n)$ for data reading and processing per process, assuming uniform slice size and processing cost. Sewing is $O(N_{vertices} + N_{triangles})$ for meshes or $O(W \times H \times n)$ for image compositing.
- **Memory Complexity:** $O(S_{local})$ per process, where $S_{local}$ is the number of slices assigned to that process.
- **Scaling:** Linear scaling is expected if I/O bandwidth scales with the number of processes and the file system supports concurrent reads.
- **Strengths:** Simple implementation; efficient I/O as each process reads only its data; no inter-process communication during rendering [S1][S2, unofficial].
- **Limitations:** Requires post-processing (sewing) for unified results; potential load imbalance if slice processing costs vary (e.g., adaptive resolution) [S2, unofficial]; boundary artifacts if overlap is not handled [S2, unofficial].

## Background
The method was described in the context of parallel volume visualization of time-varying data [S2, unofficial] and parallel isosurface extraction on GPU clusters [S1]. It is a standard approach for distributing uniform-resolution data across MPI processes [S2, unofficial].

## Sources
[S1] A parallel isosurface extraction component for visualization pipelines executing on GPU clusters, https://doi.org/10.1016/j.cam.2014.05.019, fetched 2026-10-27
[S2] Volume visualization of time-varying data using parallel, multiresolution and adaptive-resolution techniques, https://scholars.unh.edu/cgi/viewcontent.cgi?article=1245&context=thesis, fetched 2026-10-27 (unofficial)
