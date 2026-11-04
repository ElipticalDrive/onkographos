# MPI Volume Rendering (1D/2D Decomposition)

Also called: Domain Decomposition Ray Casting, Domain Decomposition Volume Rendering, MPI-based Parallel Volume Rendering with Domain Decomposition, MPI-based Ray Casting, MPICH Volume Rendering, Parallel Volume Rendering with MPI

## Overview
MPI Volume Rendering (1D/2D Decomposition) is a distributed-memory parallel volume rendering technique that partitions a 3D scalar dataset spatially along the X and/or Y axes (1D or 2D decomposition) to distribute the rendering workload across multiple MPI processes [S1, S2, S4, unofficial]. Each process independently ray casts its assigned subdomain using front-to-back compositing and early ray termination, then sends the resulting sub-image to a root process for assembly [S2, S4, unofficial]. It differs from [Scalable Distributed Volume Rendering (SDVR)](863-scalable-distributed-volume-rendering-sdvr.md) by using a static spatial decomposition of the data volume rather than a dynamic object-space or pixel-based distribution, and from [Round-Robin Data Distribution](2667-round-robin-data-distribution.md) by assigning contiguous spatial blocks to processes rather than interleaved slices [S2, S4, unofficial].

## How it works

### Inputs and Outputs
**Inputs:**
*   **Volume Data:** A 3D scalar dataset stored as a contiguous array of `float32` values. Dimensions are typically $X \times Y \times Z$ (e.g., $1000 \times 1000 \times 200$ or $2000 \times 2000 \times 400$) [S1, S2, S4, unofficial].
*   **Transfer Functions:**
    *   **Opacity TF:** A set of scalar-to-opacity mappings. Format varies: alternating lines (key/value) or comma-separated pairs [S2, S4, unofficial].
    *   **Color TF:** A set of scalar-to-RGB mappings. Format varies: groups of 4 lines (key, R, G, B) or comma-separated quads [S2, S4, unofficial].
*   **Parameters:**
    *   `step_size` (float): Distance between sample points along the ray (typical value: 0.5) [S2, S4, unofficial].
    *   `partitioning` (int): 1 for 1D decomposition (along X), 2 for 2D decomposition (along X and Y) [S2, S4, unofficial].
    *   `bounding_box` (optional): $[x_{min}, x_{max}, y_{min}, y_{max}]$ to restrict rendering region [S2, S4, unofficial].
*   **MPI Communicator:** `MPI_COMM_WORLD` with `rank` and `size` (number of processes) [S1, S2, S4, unofficial].

**Outputs:**
*   **Final Image:** A 2D RGBA image assembled by the root process (rank 0) [S2, S4, unofficial].
*   **Metrics:** Count of early ray terminations and maximum computation time across processes [S2, unofficial].

### Data Structures

**1. Volume Data Layout:**
*   Stored as a contiguous 1D array of `float32` or a NumPy array of shape $(X, Y, Z)$ in C-order (row-major) after loading [S1, S2, S4, unofficial].
*   Index calculation for voxel $(x, y, z)$ in a flattened array of width $W$ and height $H$:
    $$ \text{index} = z \cdot (W \cdot H) + y \cdot W + x $$
    [S1, unofficial]

**2. Transfer Function Dictionaries:**
*   Precomputed locally on each process to avoid I/O during rendering [S2, unofficial].
*   `opacity_dict`: Maps scalar value $\to$ opacity.
*   `color_dict`: Maps scalar value $\to$ $(R, G, B)$.
*   Keys are sorted arrays `opacity_keys` and `color_keys` for binary search [S2, unofficial].

**3. Local Image Buffer:**
*   Each process maintains a local buffer `sub_image` of shape $(H_{local}, W_{local}, 3)$ for RGB and `sub_opacity` of shape $(H_{local}, W_{local})$ for accumulated opacity [S2, S4, unofficial].
*   Dimensions $H_{local}, W_{local}$ depend on the decomposition [S2, unofficial].

**4. Decomposition Boundaries:**
*   **1D:** Process $i$ owns $x \in [x_{start}, x_{end}]$, $y \in [0, Y-1]$, $z \in [0, Z-1]$ [S2, S4, unofficial].
*   **2D:** Process $(p_x, p_y)$ owns $x \in [x_{start}, x_{end}]$, $y \in [y_{start}, y_{end}]$, $z \in [0, Z-1]$ [S2, S4, unofficial].

### Numbered Steps

1.  **Initialize MPI:** Rank 0 parses command-line arguments (filename, partitioning type, step size, bounding box). All ranks initialize the MPI communicator [S2, S4, unofficial].
2.  **Load Transfer Functions:** All ranks read opacity and color TF files and build local dictionaries/sorted key arrays [S2, S4, unofficial].
3.  **Load and Decompose Volume Data:**
    *   Rank 0 loads the full volume data [S1, S2, S4, unofficial].
    *   Rank 0 determines the decomposition grid:
        *   For 1D: $N_{proc}$ chunks along X. Chunk size $C_x = X // N_{proc}$. Remainder distributed to first $X \% N_{proc}$ processes [S2, S4, unofficial].
        *   For 2D: Find best square-ish split $(N_x, N_y)$ such that $N_x \cdot N_y = N_{proc}$ and $N_x \ge N_y$ [S2, S4, unofficial].
        *   Chunk sizes: $C_x = X // N_x$, $C_y = Y // N_y$. Remainders distributed to last processes in each dimension [S2, S4, unofficial].
    *   Rank 0 scatters the appropriate sub-volumes to each process using `MPI_Send`/`MPI_Recv` or `MPI_Scatterv` [S1, S2, S4, unofficial].
4.  **Local Ray Casting:** Each process $p$ renders its assigned sub-domain:
    *   Iterate over local pixels $(x, y)$ in the sub-domain [S1, S2, S4, unofficial].
    *   For each pixel, initialize accumulated color $C' = (0,0,0)$ and accumulated opacity $A' = 0$ [S1, S2, unofficial].
    *   March along the Z-axis from $z=0$ to $z=Z$ with step `step_size`:
        *   Calculate sample point $z_{sample}$ [S2, S4, unofficial].
        *   **Interpolate Volume Value:**
            $$ V_p = (1 - \alpha) V_1 + \alpha V_2 $$
            where $V_1, V_2$ are voxel values at $\lfloor z_{sample} \rfloor$ and $\lceil z_{sample} \rceil$, and $\alpha = z_{sample} - \lfloor z_{sample} \rfloor$ [S2, unofficial].
        *   **Evaluate Transfer Functions:**
            *   Opacity: $A = (1 - \alpha_o) A_{left} + \alpha_o A_{right}$ using `opacity_keys` [S2, unofficial].
            *   Color: $C = (1 - \alpha_c) C_{left} + \alpha_c C_{right}$ using `color_keys` [S2, unofficial].
            *   Clamp indices to valid range (e.g., if index equals length, clamp to last valid index) [S2, S4, unofficial].
        *   **Front-to-Back Compositing:**
            $$ C_i' = C_{i-1}' + (1 - A_{i-1}') C_i A_i $$
            $$ A_i' = A_{i-1}' + (1 - A_{i-1}') A_i $$
            where $C_i, A_i$ are the interpolated color and opacity of the current sample [S2, unofficial].
        *   **Early Ray Termination:** If $A_i' \ge 0.97$ (or 1.0 in some implementations), break the Z-loop and increment `early_ray_terminations` counter [S1, S2, S4, unofficial].
        *   **Clamping:** If any RGB channel $> 1.0$, clamp to 1.0. If $A' > 1.0$, clamp to 0.999 and break [S2, S3, S4, unofficial].
5.  **Gather Results:**
    *   Each process sends its `sub_image`, `sub_opacity`, `early_ray_terminations`, and timing information to Rank 0 [S2, S4, unofficial].
    *   Rank 0 receives data from all processes [S2, S4, unofficial].
6.  **Final Image Assembly:**
    *   Rank 0 creates a full-sized image buffer [S2, S4, unofficial].
    *   Place each process's `sub_image` into the corresponding spatial location in the full image based on the decomposition boundaries [S2, S4, unofficial].
    *   Save the final image to disk (e.g., PNG) [S1, S2, S4, unofficial].
7.  **Report Metrics:** Rank 0 reports the maximum computation time among all processes and the total count of early ray terminations [S2, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
from mpi4py import MPI
import bisect

def interpolate_1d(keys, values, target):
    """Linear interpolation using sorted keys and values."""
    if target <= keys[0]:
        return values[0]
    if target >= keys[-1]:
        return values[-1]
    
    idx = bisect.bisect_right(keys, target) - 1
    # Clamp index to valid range
    if idx >= len(keys) - 1:
        idx = len(keys) - 2
        
    k1, k2 = keys[idx], keys[idx+1]
    v1, v2 = values[idx], values[idx+1]
    
    if k2 == k1:
        return v1
    alpha = (target - k1) / (k2 - k1)
    return (1 - alpha) * v1 + alpha * v2

def ray_cast_subdomain(local_data, opacity_keys, opacity_values, color_keys, color_values, step_size, x_start, y_start):
    """
    Renders a sub-domain of the volume.
    local_data: numpy array of shape (H_local, W_local, Z)
    Returns: sub_image (H_local, W_local, 3), sub_opacity (H_local, W_local), early_terminations
    """
    H, W, Z = local_data.shape
    sub_image = np.zeros((H, W, 3), dtype=np.float32)
    sub_opacity = np.zeros((H, W), dtype=np.float32)
    early_terminations = 0
    
    for i in range(H):
        for j in range(W):
            acc_r, acc_g, acc_b = 0.0, 0.0, 0.0
            acc_a = 0.0
            
            z_sample = 0.0
            while z_sample < Z - 1:
                z_floor = int(np.floor(z_sample))
                z_ceil = z_floor + 1
                alpha_z = z_sample - z_floor
                
                # Interpolate volume value
                v1 = local_data[i, j, z_floor]
                v2 = local_data[i, j, z_ceil]
                v_sample = (1 - alpha_z) * v1 + alpha_z * v2
                
                # Evaluate Transfer Functions
                op = interpolate_1d(opacity_keys, opacity_values, v_sample)
                
                # Interpolate Color
                # Find index for color keys
                idx_c = bisect.bisect_right(color_keys, v_sample) - 1
                if idx_c >= len(color_keys) - 1:
                    idx_c = len(color_keys) - 2
                k1, k2 = color_keys[idx_c], color_keys[idx_c+1]
                c1 = color_values[idx_c]
                c2 = color_values[idx_c+1]
                
                if k2 == k1:
                    alpha_c = 0
                else:
                    alpha_c = (v_sample - k1) / (k2 - k1)
                    
                r_sample = (1 - alpha_c) * c1[0] + alpha_c * c2[0]
                g_sample = (1 - alpha_c) * c1[1] + alpha_c * c2[1]
                b_sample = (1 - alpha_c) * c1[2] + alpha_c * c2[2]
                
                # Front-to-Back Compositing
                one_minus_acc_a = 1.0 - acc_a
                factor = one_minus_acc_a * op
                
                acc_r += factor * r_sample
                acc_g += factor * g_sample
                acc_b += factor * b_sample
                acc_a += factor
                
                # Early Termination
                if acc_a >= 0.97: # Threshold from [S1, unofficial]
                    early_terminations += 1
                    break
                
                # Clamping
                if acc_a > 1.0:
                    acc_a = 0.999
                    break
                    
                z_sample += step_size
            
            sub_image[i, j, 0] = acc_r
            sub_image[i, j, 1] = acc_g
            sub_image[i, j, 2] = acc_b
            sub_opacity[i, j] = acc_a
            
    return sub_image, sub_opacity, early_terminations

def main():
    comm = MPI.COMM_WORLD
    rank = comm.Get_rank()
    size = comm.Get_size()
    
    # 1. Parse Arguments (Rank 0)
    if rank == 0:
        filename = sys.argv[1]
        partitioning = int(sys.argv[2])
        step_size = float(sys.argv[3])
        # ... parse bounding box ...
        
        # Load TFs
        opacity_dict = load_opacity_tf("opacity_TF.txt")
        color_dict = load_color_tf("color_TF.txt")
        
        opacity_keys = np.array(sorted(opacity_dict.keys()))
        opacity_values = np.array([opacity_dict[k] for k in opacity_keys])
        
        color_keys = np.array(sorted(color_dict.keys()))
        color_values = np.array([color_dict[k] for k in color_keys])
        
        # Broadcast TFs
        comm.bcast(opacity_keys, root=0)
        comm.bcast(opacity_values, root=0)
        comm.bcast(color_keys, root=0)
        comm.bcast(color_values, root=0)
        
        # Load Volume
        volume = np.fromfile(filename, dtype=np.float32)
        # ... reshape and transpose to (X, Y, Z) ...
        
        # Decompose
        if partitioning == 1:
            # 1D Decomposition
            chunk_size = volume.shape[0] // size
            remainder = volume.shape[0] % size
            # Calculate start/end for each rank
            # ...
        elif partitioning == 2:
            # 2D Decomposition
            # Find best split Nx, Ny
            # ...
            
        # Scatter Volume Data
        # ...
    else:
        # Receive TFs
        opacity_keys = comm.bcast(None, root=0)
        opacity_values = comm.bcast(None, root=0)
        color_keys = comm.bcast(None, root=0)
        color_values = comm.bcast(None, root=0)
        
        # Receive Volume Data
        # ...

    # 2. Local Rendering
    start_time = MPI.Wtime()
    sub_image, sub_opacity, early_terms = ray_cast_subdomain(
        local_data, opacity_keys, opacity_values, color_keys, color_values, step_size, x_start, y_start
    )
    end_time = MPI.Wtime()
    local_time = end_time - start_time
    
    # 3. Gather Results
    if rank == 0:
        final_image = np.zeros((global_H, global_W, 3), dtype=np.float32)
        max_time = local_time
        
        # Place own sub-image
        # ...
        
        for r in range(1, size):
            recv_data = comm.recv(source=r, tag=0)
            recv_sub_image, recv_early_terms, recv_time = recv_data
            
            # Place recv_sub_image into final_image at correct location
            # ...
            
            max_time = max(max_time, recv_time)
            
        # Save final_image
        save_image(final_image, "output.png")
        print(f"Max Time: {max_time}, Early Terminations: {total_early_terms}")
    else:
        comm.send((sub_image, early_terms, local_time), dest=0, tag=0)

if __name__ == "__main__":
    main()
```

### Parameters and Typical Values
*   **Step Size:** 0.5 (voxel units) [S2, S4, unofficial].
*   **Early Termination Threshold:** 0.97 or 1.0 [S1, S2, unofficial].
*   **Volume Dimensions:** $1000 \times 1000 \times 200$ [S1, unofficial] or $2000 \times 2000 \times 400$ [S2, unofficial].
*   **Transfer Function Points:** 4 for opacity, 55 for color [S1, S2, unofficial].

### Edge Cases and Pitfalls
*   **Index Clamping:** When using binary search (`bisect`) for transfer function interpolation, if the index equals the length of the keys array, it must be clamped to the last valid index to prevent out-of-bounds access [S2, S4, unofficial].
*   **Non-Divisible Dimensions:** When dividing volume dimensions among processes, the last process in a dimension receives the remainder of the division to ensure all data is covered [S2, S4, unofficial].
*   **Opacity Clamping:** If accumulated opacity exceeds 1.0, it is clamped to 0.999 and the ray loop breaks to prevent over-saturation [S3, unofficial].
*   **RGB Clamping:** RGB channels are clamped to 1.0 if they exceed this value [S3, S4, unofficial].
*   **Boundary Checks:** The ray marching loop must check if the next Z index exceeds the volume depth to avoid out-of-bounds memory access [S1, unofficial].

## Subtopics

### [MPI Volume Rendering (3D Decomposition)](2254-mpi-volume-rendering-3d-decomposition.md)
Partitions the volume into a 3D grid of subdomains, distributing data along X, Y, and Z axes. Offers finer-grained load balancing but requires more complex communication for stitching.

### [Parallel Volume Rendering with Static Load Balancing](2547-parallel-volume-rendering-with-static-load-balancing.md)
Combines spatial partitioning with static load balancing for unstructured meshes, aiming for high performance with modest memory [unofficial].

### [MPI Master Node Parallel Rendering Workflow](2625-mpi-master-node-parallel-rendering-workflow.md)
Coordinates rendering by spawning slave processes, distributing parameters, receiving per-slice images, and assembling/encoding the final image [unofficial].

### [Parallel Volume Ray Casting (PVR) Library Interface](3206-parallel-volume-ray-casting-pvr-library-interface.md)
A library interface for distributed memory environments handling rectilinear grids, allowing application programs to pass subvolume pointers and rendering specs [unofficial].

### [Parallel Distributed Axis-aligned Volume Rendering using MPI](3804-parallel-distributed-axis-aligned-volume-rendering-using-mpi.md)
Performs axis-aligned rendering by decomposing data spatially, distributing data along the view axis rather than rays, eliminating per-ray communication.

### [Parallel DVR Algorithm (Object-Space Decomposition)](4258-parallel-dvr-algorithm-object-space-decomposition.md)
Uses clustering, remapping, local rendering, and pixel merging phases, with view-dependent remapping [unofficial].

## Operating characteristics

*   **Time Complexity:** $O(N \cdot P^{-1})$ for rendering, where $N$ is the number of voxels/rays and $P$ is the number of processes, assuming perfect load balancing [S2, S4, unofficial].
*   **Memory Complexity:** $O(V/P)$ per process, where $V$ is the volume size [S2, S4, unofficial].
*   **Performance:** Early ray termination can reduce execution time by up to 15% [S1, unofficial]. Performance scales with the number of processes, but communication overhead for gathering results can become a bottleneck on the root process [S3, S4, unofficial].
*   **Strengths:** Simple implementation, good scalability for large volumes, effective use of early termination [S1, S2, unofficial].
*   **Limitations:** Static decomposition may lead to load imbalance if the volume has varying density [S2, unofficial]. Root process bottleneck for image assembly [S3, unofficial].
*   **When to Use:** Large static volumes on distributed memory clusters where load balancing is relatively uniform [S2, S4, unofficial].
*   **When Not to Use:** Highly dynamic scenes or volumes with extreme density variations requiring dynamic load balancing [S2, unofficial].

## Background

MPI Volume Rendering (1D/2D Decomposition) is a distributed memory parallel volume rendering technique that uses the Message Passing Interface (MPI) to decompose the image domain and volume data across multiple processes [S1, S2, unofficial]. It optimizes 1D/2D domain decomposition to process 3D volume data faster, implementing front-back ray casting with early termination [S1, unofficial]. The method is used in projects for visualizing large datasets (e.g., 2000x2000x400) on MPI clusters [S2, unofficial]. It differs from global ray casting by distributing data spatially rather than distributing rays, which eliminates per-ray communication overhead [S2, S3, unofficial].

## Sources

[S1] Abuster147/Parallel-Volume-Rendering-using-MPI, https://github.com/Abuster147/Parallel-Volume-Rendering-using-MPI, fetched 2026-10-26 (unofficial)
[S2] roshankrm/VolumeRendering, https://github.com/roshankrm/VolumeRendering, fetched 2026-10-27 (unofficial)
[S3] rishit-AI/parallel-volume-rendering-with-binary-swap, https://github.com/rishit-AI/parallel-volume-rendering-with-binary-swap, fetched 2026-10-27 (unofficial)
[S4] yugul7714/Parallel-Volume-Rendering-using-MPI-and-Visualization, https://github.com/yugul7714/Parallel-Volume-Rendering-using-MPI-and-Visualization, fetched 2026-10-27 (unofficial)
