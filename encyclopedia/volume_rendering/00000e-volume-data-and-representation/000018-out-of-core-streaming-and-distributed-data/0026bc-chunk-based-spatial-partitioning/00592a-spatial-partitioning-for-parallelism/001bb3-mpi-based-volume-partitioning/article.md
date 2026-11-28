# MPI-based Volume Partitioning

Also called: Chunk-based MPI parallelism, Spatial Division of Uniform Resolution Data

## Overview
A data-parallel strategy where a volumetric dataset is partitioned into contiguous chunks of slices, with each MPI process reading and rendering only its assigned portion [S1][S2, unofficial]. Builds on [Spatial Partitioning for Parallelism](16344-spatial-partitioning-for-parallelism.md). This method solves the problem of efficient I/O and parallel rendering for uniform-resolution volumes by ensuring each process accesses contiguous memory regions, minimizing seek overhead [S2, unofficial].

## How it works
**Inputs:** A volumetric dataset stored as a sequence of 2D slices (e.g., along the z-axis); the number of MPI processes $P$ [S1][S2, unofficial].
**Outputs:** A set of scattered binary result files (one per process) containing partial renderings or mesh data; optionally, a sewed final image or mesh [S1].

**Data Structures:**
*   **Slice Index:** Integer identifying the slice position in the global volume [G].
*   **Chunk:** A contiguous range of slice indices $[start, end)$ assigned to a single process [G].
*   **Triangle Table (for isosurface extraction):** A list of triangles defined by vertex indices [S1]. Each process maintains a local vertex table [S1].

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
    *   Let $O$ be the overlap size (number of extra slices) [G].
    *   Processor $i$ reads slices from $start_i$ to $end_i + O$ (where $end_i = start_i + s_i$).
    *   Processor $0$ reads from $start_0$ to $end_0 + O$.
    *   Processor $P-1$ reads from $start_{P-1}$ to $end_{P-1}$.
    *   Intermediate processors $i$ read from $start_i$ to $end_i + O$.
    *   The overlap ensures that opacity and color calculations at the division boundaries are complete, preventing black stripes during compositing [S2, unofficial].

3.  **Read and Render:**
    Each MPI process $i$ seeks to the file offset corresponding to its first slice (including overlap) and reads its assigned chunk [S2, unofficial]. The process renders its portion of the volume.
    *   For ray casting: Compositing is performed locally for the assigned slices [S2, unofficial].
    *   For isosurface extraction: Generate local triangles and vertices [S1].

4.  **Write Local Results:**
    Write the rendered image or mesh data to a scattered binary file specific to process $i$ [S1].
    *   If rendering an image: Write the pixel buffer for the assigned slices [S2, unofficial].
    *   If extracting isosurfaces: Write the local triangle table and vertex table [S1].

5.  **Sewing (Optional for Isosurfaces):**
    If the output is a mesh, combine the partial triangle tables from all processes [S1].
    *   Concatenate the vertex tables from all processes [S1].
    *   Update the vertex indices in each process's triangle table to account for the offset in the global vertex table [S1].
    *   Sewing is optional if the scattered files are used as input for subsequent parallel processing steps [S1].

6.  **Compositing (Optional for Images):**
    If the output is a final image, composite the scattered image files.
    *   Remove the extra/duplicated overlap slices from the edges of each process's image to prevent stretching [S2, unofficial].
    *   Combine the remaining slices in order [S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def mpi_volume_partitioning(num_slices, num_procs, overlap_slices):
    """
    Calculates slice assignments and read ranges for MPI volume partitioning.
    """
    base_slices = num_slices // num_procs
    remainder = num_slices % num_procs
    
    slice_counts = []
    for i in range(num_procs):
        slice_counts.append(base_slices + 1 if i < remainder else base_slices)
        
    start_indices = [0] * num_procs
    current_start = 0
    for i in range(num_procs):
        start_indices[i] = current_start
        current_start += slice_counts[i]
        
    chunks = []
    for i in range(num_procs):
        start = start_indices[i]
        end = start + slice_counts[i]
        
        # Apply overlap: first and intermediate processors extend by O,
        # last processor does not extend beyond total slices.
        read_end = end + overlap_slices if i < num_procs - 1 else end
        read_end = min(read_end, num_slices)
        
        chunks.append({
            'proc_id': i,
            'read_start': start,
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
        global_vertices.extend(mesh.vertices)
        for tri in mesh.triangles:
            global_triangles.append([v + vertex_offset for v in tri])
        vertex_offset += len(mesh.vertices)
        
    return global_vertices, global_triangles
```

**Parameters:**
*   **Overlap Size:** Not given in the sources. Must be chosen to cover the stencil size of the rendering/extraction algorithm [G].
*   **Slice Order:** Must match storage order for efficient seeking and compositing without additional mappings [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Boundary Artifacts:** Naive non-overlapping division causes black stripes during compositing because opacity and color calculations are missing at the division boundaries [S2, unofficial].
*   **Image Stretching:** If overlap slices are not removed before final compositing, the image will be stretched [S2, unofficial].
*   **Uneven Workload:** For uniform resolution data, workload is proportional to slice count. If $S$ is not divisible by $P$, some processors get $\lfloor S/P \rfloor + 1$ slices and others $\lfloor S/P \rfloor$ [S2, unofficial].
*   **Adaptive Resolution:** This method partitions uniform-resolution data. For adaptive-resolution data, workload balancing requires dividing by computational cost rather than slice count [S2, unofficial].

## Subtopics

### Multi-Reader File Striping
Breaks a single large input volume file into multiple smaller sub-files to be read simultaneously by multiple processes, minimizing disk controller conflicts and network load [S3, unofficial]. The file is apportioned in a round-robin fashion or simple linear division into $n$ parts [S3, unofficial]. Up to six reader processes are typically assigned, matching the number of local SCSI disks [S3, unofficial]. Stripping into more than six units causes SCSI controller contention and requires re-striping if the processor count changes [S3, unofficial]. For datasets upwards of $128^3$ data points, striping into six units yields a speedup of six in data read time [S3, unofficial]. This manual striping differs from traditional OS-managed striped file systems [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(S/P)$ for reading and rendering, where $S$ is the number of slices and $P$ is the number of processors [S2, unofficial]. Sewing complexity is not given in the sources.
*   **Memory Complexity:** $O(S_{chunk} \times \text{slice\_size})$ per process. Each process only loads its assigned chunk [S1].
*   **Scaling:** Linear scaling is expected if I/O and computation are balanced. Efficiency depends on the uniformity of the data and the overhead of sewing/compositing [S2, unofficial].
*   **Strengths:** Simple implementation; efficient I/O due to contiguous slice access; no need for complex load balancing for uniform data [S2, unofficial].
*   **Limitations:** Not suitable for adaptive-resolution data without modification; requires post-processing (sewing/compositing) to combine results; boundary artifacts if overlap is not handled [S2, unofficial].
*   **When to use:** Uniform-resolution volumes where I/O efficiency is critical and the number of processors is known and static [S1][S2, unofficial].
*   **When not to use:** Not given in the sources.

## Background
The method was described in the context of parallel isosurface extraction on GPU clusters [S1] and volume visualization of time-varying data [S2, unofficial]. It is a standard approach for data-parallel volume rendering where the data is partitioned spatially along one axis [S2, unofficial].

## Sources
[S1] A parallel isosurface extraction component for visualization pipelines executing on GPU clusters, https://doi.org/10.1016/j.cam.2014.05.019, fetched 2026-10-27
[S2] Volume visualization of time-varying data using parallel, multiresolution and adaptive-resolution techniques, https://scholars.unh.edu/cgi/viewcontent.cgi?article=1245&context=thesis, fetched 2026-10-27 (unofficial)
[S3] DTIC ADA296601: An Investigation of the Parallel Implementation of Two Volume Visualization Techniques on the nCUBE 2., https://archive.org/details/DTIC_ADA296601, fetched 2026-10-26 (unofficial)
