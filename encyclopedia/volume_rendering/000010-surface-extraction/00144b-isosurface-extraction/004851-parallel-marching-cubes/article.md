# Parallel Marching Cubes

Also called: Marching Cubes with Slice Strategy, PBIG, Parallel Isosurface Generator, Piecewise Marching Cubes

## Overview
Parallel Marching Cubes is a parallel implementation of the Marching Cubes algorithm designed to compute isosurfaces of huge datasets that cannot fit entirely into a single machine's memory [S1, unofficial]. It solves the memory bottleneck of traditional isosurface extraction by using a piecewise slice strategy along the k-direction, requiring only that at least two slices fit into physical memory [S1, unofficial]. Unlike commercial packages that assume the entire volume fits in memory, this method distributes the workload across multiple processes or threads, processing grid slices concurrently [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** Two unformatted PLOT3D files: a grid file (`xyz`) defining the structured mesh (regular or irregular) and a solution file (`Q`) containing scalar fields [S1, unofficial]. The grid must be oriented such that it can be sliced in the k-direction [S1, unofficial].
*   **Output:** A Bop (Bag-O-Polygons) file containing three-sided polygons defining the isosurface where $F(x,y,z) = S$, along with a second scalar mapped onto the surface [S1, unofficial].

**Data Structures**
*   **Grid/Solution Data:** Structured 3D arrays indexed by $i, j, k$. The grid file provides coordinates; the solution file provides scalar values $v(x,y,z)$ [S1, unofficial].
*   **Partition Structure:** A C structure created by the parent process containing information for each child process to determine which part of the grid to read and process [S1, unofficial].
*   **Bop Polygon (`boply`):** A C structure representing a single polygon:
    *   `int num_vertices`: Number of vertices (typically 3).
    *   `float x, y, z`: Coordinates for each vertex.
    *   `float scalar`: The mapped scalar value for each vertex [S1, unofficial].

**Parameters**
*   `argv[1]`: PLOT3D grid filename.
*   `argv[2]`: PLOT3D solution filename.
*   `argv[3]`: Output Bop filename.
*   `argv[4]`: Numeric index into the Q_file for the scalar defining the isosurface.
*   `argv[5]`: Numeric index into the Q_file for the scalar mapped onto the surface for color.
*   `argv[6]`: Floating-point isosurface value $S$ (Typical value: 5.0) [S1, unofficial].
*   `argv[7]`: Integer number of planes to step in the k-direction ($Kstep$) (Typical value: 2 or 3) [S1, unofficial].
*   `argv[8]`: Integer number of processes to use ($Num\_procs$) [S1, unofficial].

**Algorithm Steps**
1.  **Initialization:** The parent process opens the grid and solution files to read header information, specifically the number of grids (expected to be one) and the dimensions in $i, j,$ and $k$ directions [S1, unofficial].
2.  **Partitioning:** The parent calls a partitioning routine to divide the workload. The grid is sliced in the k-direction. The routine creates a partition structure containing the start and end indices for each child process [S1, unofficial].
3.  **Process Creation:** The parent creates $Num\_procs$ child processes. The mechanism depends on the platform: `fork` on SGI/Sun workstations, `sproc` on SGI workstations, or `pthreads` on the KSR1 machine [S1, unofficial].
    *   *Note on Memory:* `fork` uses copy-on-write, so writes are not visible to other processes. `sproc` creates a share group process where the child shares the virtual address space (shared memory, mapped files) with the parent [S1, unofficial].
4.  **Slice Processing (Child Process):**
    a.  Each child reads its assigned slice of the grid and solution data into local memory [S1, unofficial].
    b.  The child iterates through the cubes in its local slice. For each cube, it determines the intersection topology:
        i.  Assign a bit value of 1 to a node if the scalar value $S$ at that node is $\ge$ the user-specified isosurface value, and 0 otherwise [S1, unofficial].
        ii. Form an 8-bit index from these 8 bits [S1, unofficial].
        iii. Look up the topology in a table of 15 major cases (reduced from 256 via symmetry) [S1, unofficial].
    c.  For the retrieved triangulation case, use linear interpolation to determine the exact intersection points of the surface with the cube edges [S1, unofficial].
    d.  Form a list of three-sided polygons (vertices and mapped scalars) [S1, unofficial].
5.  **Serialization and Output:** Child processes write their generated polygon data to the output Bop file. To prevent collisions from simultaneous writes, semaphores enforce mutual exclusion, allowing only one child to write or network polygons at a time [S1, unofficial]. Children with smaller or no output complete execution before those with larger output requirements [S1, unofficial].

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Main Process
def parallel_marching_cubes(grid_file, sol_file, bop_file, scalar_idx, map_idx, isovalue, kstep, num_procs):
    # 1. Read headers
    grid_header = read_p3d_header(grid_file)
    sol_header = read_p3d_header(sol_file)
    dims_i, dims_j, dims_k = grid_header.dimensions
    
    # 2. Partition workload
    # Divide k-dimension into num_procs chunks. 
    # Each chunk has a start_k and end_k.
    partitions = create_partitions(dims_k, num_procs)
    
    # 3. Spawn children
    children = []
    for p_id in range(num_procs):
        child = spawn_child(fork_or_sproc_or_pthread) # Platform dependent
        child.args = {
            'grid_file': grid_file,
            'sol_file': sol_file,
            'bop_file': bop_file,
            'scalar_idx': scalar_idx,
            'map_idx': map_idx,
            'isovalue': isovalue,
            'partition': partitions[p_id],
            'semaphore': get_semaphore() # For output serialization
        }
        children.append(child)
        
    # 4. Wait for completion
    for child in children:
        child.join()

# Child Process
def child_process(grid_file, sol_file, bop_file, scalar_idx, map_idx, isovalue, partition, semaphore):
    start_k, end_k = partition
    
    # 5. Load slice into memory
    # Only load the k-range [start_k, end_k]
    grid_slice = read_p3d_grid_slice(grid_file, start_k, end_k)
    sol_slice = read_p3d_sol_slice(sol_file, start_k, end_k, scalar_idx)
    map_slice = read_p3d_sol_slice(sol_file, start_k, end_k, map_idx)
    
    polygons = []
    
    # 6. Marching Cubes on local slice
    for k in range(start_k, end_k - 1):
        for j in range(0, dims_j - 1):
            for i in range(0, dims_i - 1):
                # Get 8 vertices of the cube
                vertices = get_cube_vertices(grid_slice, i, j, k)
                values = get_cube_values(sol_slice, i, j, k)
                
                # Determine case index
                case_index = 0
                for v_idx in range(8):
                    if values[v_idx] >= isovalue:
                        case_index |= (1 << v_idx)
                
                # Lookup topology
                triangles = lookup_marching_cubes_table(case_index)
                
                # Interpolate vertices
                for tri in triangles:
                    poly = create_polygon(vertices, tri, map_slice, i, j, k)
                    polygons.append(poly)
    
    # 7. Write output with serialization
    with semaphore.lock():
        write_bop_file(bop_file, polygons)
```

**Edge Cases and Pitfalls**
*   **Grid Orientation:** The grid must be oriented such that it can be sliced in the k-direction. If the grid orientation does not accommodate this, the method may fail or require preprocessing [S1, unofficial].
*   **Output Collisions:** Simultaneous writes by multiple child processes to the Bop format output function cause collisions. This is handled by semaphores [S1, unofficial].
*   **Memory Constraints:** The method requires that at least two slices fit into physical memory. If the slice size exceeds available memory, the method fails [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of cubes processed, $O(N_{cubes})$, distributed across $P$ processes.
*   **Memory Complexity:** $O(N_{slice})$, where $N_{slice}$ is the number of voxels in a slice. This is significantly lower than the $O(N_{volume})$ required by sequential methods.
*   **Measured Performance:** On a dataset of ~2.2M grid values and ~11M scalar values, generating ~60k polygons, wall-clock time reduced from ~20 min (sequential) to ~90 s (parallel) [S1, unofficial].
*   **Hardware:** Tested on SUN4spare (2 processors), SGI Challenge (8 processors), and KSR1 (8 processors) [S1, unofficial].
*   **Strengths:** Handles datasets too large for main memory; scalable with number of processes; supports irregular grids.
*   **Limitations:** Requires specific grid orientation (k-slicing); output serialization can become a bottleneck if many polygons are generated; platform-specific process creation code (`fork` vs `sproc` vs `pthreads`) [S1, unofficial].

## Background
Parallel Marching Cubes was developed as a parallel C implementation to compute isosurfaces of huge datasets, specifically applied to computational data from CTH and HULL codes, and medical resonance data [S1, unofficial]. It runs on UNIX-compatible systems (SGI IRIX, Sun SunOS, KSR KSR OS) using `fork`, `sproc`, or `pthreads` for parallelism [S1, unofficial]. The method builds on the original Marching Cubes algorithm by Lorenson and Cline (1987) [S1, unofficial].

## Sources
[S1] DTIC ADA284030: Parallel Isosurface Generator for Huge Datasets, https://archive.org/details/DTIC_ADA284030, fetched 2026-10-26 (unofficial)
