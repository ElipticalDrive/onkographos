# MPI Parallel Slice-Based Volume Rendering
Also called: Distributed Slice Rendering, Domain Decomposition Slicing, MPI Parallel Slicing, Parallel Slice Rendering, Parallel Volume Rendering

## Overview
MPI Parallel Slice-Based Volume Rendering is a distributed computing method that decomposes a 3D volume dataset along the slicing axis (typically the Z-axis) among multiple MPI processes, where each process renders its local subset of slices and sends the resulting image fragments to a root process for final assembly [S1, unofficial]. It differs from [MPI Master Node Parallel Rendering Workflow](2625-mpi-master-node-parallel-rendering-workflow.md) by explicitly handling the domain decomposition logic and image concatenation within the rendering loop itself, rather than relying on a pre-defined master-slave workflow abstraction, and it differs from [MPI Master-Slave Octree Volume Rendering](2744-mpi-master-slave-octree-volume-rendering.md) by operating on uniform slice blocks rather than hierarchical octree structures [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D scalar volume dataset (e.g., VTI file), a transfer function (color and opacity), the number of MPI processes (`size`), and the current process rank (`rank`).
*   **Output:** A composite 2D image assembled by the root process (rank 0).

**Data Structures:**
*   **Volume Data:** A 3D NumPy array `camera_grid` of shape $(H, W, D)$, where $H$ is height, $W$ is width, and $D$ is depth (number of slices). The data is transposed such that the slicing axis corresponds to the last dimension (axis 2) [S1, unofficial].
*   **Local Image Buffer:** A 3D NumPy array `image` of shape $(H, W, 3)$, initialized to zeros, storing the RGBA or RGB result for the local slices [S1, unofficial].
*   **MPI Communicator:** `MPI.COMM_WORLD` [S1, unofficial].

**Parameters:**
*   `size`: Total number of MPI processes.
*   `rank`: Unique identifier for the current process ($0 \le rank < size$).
*   **Transfer Function Range:** Scalar values are mapped from -5000 to 2600 [S1, unofficial].
*   **Opacity Points:** Opacity is 1.0 at -5000.0 and 0.0 at 2600.0 (linear interpolation implied by `vtkPiecewiseFunction` usage in source) [S1, unofficial]. Note: Another variant uses opacity 0.8 at -5000.0 and 0.2 at 2600.0 [S1, unofficial].

**Numbered Steps:**
1.  **Initialize MPI:** Create the communicator, get `rank` and `size` [S1, unofficial].
2.  **Load and Transform Data:** Read the volume file (e.g., VTI) and convert to a NumPy array. Transpose the array so the view/slicing axis is the last dimension (Z-axis) [S1, unofficial].
3.  **Calculate Slice Range:** Determine the local slice indices $[p1, p2)$ for the current rank. The volume depth $D$ is divided by `size`. The last rank handles any remainder [S1, unofficial]:
    $$ p1 = \lfloor D / size \rfloor \times rank $$
    $$ p2 = \begin{cases} \lfloor D / size \rfloor \times (rank + 1) & \text{if } rank < size - 1 \\ D & \text{if } rank = size - 1 \end{cases} $$
    *Note: Some implementations use $p2 = (\lfloor D / size \rfloor \times (rank + 1)) - 1$, which may exclude the last slice if not handled carefully [S1, unofficial].*
4.  **Extract Local Data:** Slice the volume data: `camera_grid = camera_grid[:, :, p1:p2]` [S1, unofficial].
5.  **Initialize Local Buffer:** Create `image = np.zeros((camera_grid.shape[1], camera_grid.shape[2], 3))` [S1, unofficial].
6.  **Render Local Slices:** For each slice in `camera_grid`:
    *   Apply logarithmic scaling to the scalar values: $x = \log(\text{slice})$.
    *   Compute Color $(R, G, B)$ using a Gaussian mixture transfer function or lookup table [S1, unofficial].
    *   Compute Opacity $(\alpha)$ using a piecewise function [S1, unofficial].
    *   Perform front-to-back compositing for the slice:
        $$ I_{new} = \alpha \cdot C + (1 - \alpha) \cdot I_{old} $$
        where $I$ is the image buffer and $C$ is the slice color [S1, unofficial].
7.  **Communicate Results:**
    *   If `rank != 0`: Send `image` to rank 0 using `comm.Send(image, dest=0)` [S1, unofficial].
    *   If `rank == 0`: Initialize `total_data` with its own `image`. Then, loop $k$ from 1 to `size-1`:
        *   Receive `image` from source $k$ using `comm.Recv(image, source=k)` [S1, unofficial].
        *   Concatenate `image` to `total_data`.
8.  **Display/Save:** The root process displays or saves `total_data` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
import MPI

def transfer_function_color(x):
    """
    Gaussian mixture transfer function for RGB.
    x: scalar value (log-scaled).
    Returns: (r, g, b)
    """
    r = 1.0 * np.exp(-(x - 9.0)**2 / 1.0) + 0.1 * np.exp(-(x - 3.0)**2 / 0.1) + 0.1 * np.exp(-(x - (-3.0))**2 / 0.5)
    g = 1.0 * np.exp(-(x - 9.0)**2 / 1.0) + 1.0 * np.exp(-(x - 3.0)**2 / 0.1) + 0.1 * np.exp(-(x - (-3.0))**2 / 0.5)
    b = 0.1 * np.exp(-(x - 9.0)**2 / 1.0) + 0.1 * np.exp(-(x - 3.0)**2 / 0.1) + 1.0 * np.exp(-(x - (-3.0))**2 / 0.5)
    return r, g, b

def transfer_function_opacity(x, opacity_func):
    """
    Linear opacity function defined by points (-5000, 1.0) and (2600, 0.0).
    x: scalar value (log-scaled).
    opacity_func: A simple linear interpolation function or lookup.
    Returns: alpha
    """
    # Simplified linear interpolation for demonstration
    # In practice, use vtkPiecewiseFunction or similar
    min_val, max_val = -5000.0, 2600.0
    min_alpha, max_alpha = 1.0, 0.0
    if x < min_val: return min_alpha
    if x > max_val: return max_alpha
    return min_alpha + (x - min_val) * (max_alpha - min_alpha) / (max_val - min_val)

def main():
    comm = MPI.COMM_WORLD
    rank = comm.Get_rank()
    size = comm.Get_size()

    # 1. Load and Transform Data
    # Assume load_volume() returns array of shape (D, H, W)
    original_array = load_volume('data.vti') 
    # Transpose to (H, W, D) for Z-axis slicing
    camera_grid = np.transpose(original_array, (2, 1, 0))
    
    a = camera_grid.shape
    D = a[2] # Depth
    
    # 2. Calculate Slice Range
    p1 = (D // size) * rank
    if rank < size - 1:
        p2 = (D // size) * (rank + 1)
    else:
        p2 = D
        
    # Extract local slices
    local_grid = camera_grid[:, :, p1:p2]
    
    # 3. Initialize Local Image Buffer
    H, W, _ = local_grid.shape
    image = np.zeros((H, W, 3), dtype=np.float32)
    
    # 4. Render Local Slices
    for dataslice in local_grid:
        # Log scale
        x = np.log(dataslice)
        
        # Get Color
        r, g, b = transfer_function_color(x)
        
        # Get Opacity
        alpha = transfer_function_opacity(x, None) # Placeholder for actual TF object
        
        # Composite
        # image[:, :, 0] = alpha * r + (1 - alpha) * image[:, :, 0]
        # image[:, :, 1] = alpha * g + (1 - alpha) * image[:, :, 1]
        # image[:, :, 2] = alpha * b + (1 - alpha) * image[:, :, 2]
        
        # Vectorized operation for speed
        color = np.stack([r, g, b], axis=-1)
        image = alpha * color + (1 - alpha) * image

    # 5. Communicate Results
    if rank != 0:
        comm.Send(image, dest=0)
    else:
        total_data = image
        for k in range(1, size):
            recv_image = comm.Recv(source=k)
            # Concatenate along axis 1 (Width) as per source code, 
            # though logically slices are along axis 2 (Depth). 
            # This implies a specific layout assumption in the source.
            total_data = np.concatenate((total_data, recv_image), axis=1)
            
        # Optional: Crop or adjust based on p2 if necessary
        # total_data = total_data[:, p2:(p2 + a[2]), :] # Source code snippet
        
    # 6. Display
    if rank == 0:
        display_image(total_data)

if __name__ == "__main__":
    main()
```

**Edge Cases and Pitfalls:**
*   **Concatenation Axis Mismatch:** The source code concatenates images along `axis=1` (width), but the data is split along `axis=2` (depth/slices). This results in an incorrect spatial arrangement unless the image dimensions are swapped or the view axis is interpreted differently [S1, unofficial].
*   **Index Calculation for Last Rank:** The formula `p2 = (int)(((a[2] // size) * (rank + 1)) - 1)` may exclude the last slice if the volume size is not evenly divisible by the number of processes. The robust method uses `p2 = D` for the last rank [S1, unofficial].
*   **Precision:** Logarithmic scaling of negative or zero values will result in `-inf` or `NaN`. The source assumes positive scalar values or handles them via the transfer function's domain [S1, unofficial].

## Subtopics

### [Boundary Sharing for Adaptive-Resolution Blocks](7524-boundary-sharing-for-adaptive-resolution-blocks.md)
Boundary Sharing for Adaptive-Resolution Blocks is a technique to eliminate visual artifacts (such as black strips or visible seams) in parallel volume rendering when the volume data is partitioned into 3D blocks rather than 2D slices. It solves the problem of lost ray samples at subvolume boundaries by sharing boundary slices between adjacent blocks. Unlike standard slice-based sharing which only handles 2D cuts, this method manages 3D block interfaces and adapts to multi-resolution data where adjacent blocks may have different voxel densities.

### Parallel Distributed 2D Texture Slicing
Parallel Distributed 2D Texture Slicing is an engine with parallel distributed implementation for visualizing large, remotely located, time-varying, adaptive, hierarchical scientific data that won't fit on a workstation [S2, unofficial]. It uses a 2D texture-based method with axis-aligned slicing to achieve interactive transformation on low-end workstations [S2, unofficial]. Each processor renders a subset of the volume in 2D textures, which are then combined into a low-cost hardware graphics pipeline [S2, unofficial]. The resulting image is used as raw texture data applied to an axis-aligned quad for volume rendering, with image warping and depth-order compositing performed using graphics hardware [S2, unofficial]. Unlike standard architectures that compress/decompress images, this technique uses a stage of partition and composition [S2, unofficial]. A limitation is a restricted rotation range to avoid visual degradation, which is data-dependent and can be adjusted by changing the number of texture maps [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N/P)$ for rendering, where $N$ is the total number of slices and $P$ is the number of processes, assuming uniform data distribution. Communication overhead is $O(P)$ for gathering images [S1, unofficial].
*   **Memory Complexity:** $O(N/P)$ per process for local volume data and image buffers [S1, unofficial].
*   **Performance:** Scales linearly with the number of processes for large volumes, limited by network bandwidth for image transfer [S1, unofficial].
*   **Strengths:** Simple implementation, effective for large uniform volumes, leverages MPI for robust distributed computing [S1, unofficial].
*   **Limitations:** Load imbalance if volume density is non-uniform, potential visual artifacts at boundaries if not handled (see Subtopics), and concatenation axis errors in naive implementations [S1, unofficial].
*   **When to Use:** Large static volumes that fit in aggregate memory but not in a single node's memory, where uniform distribution is acceptable [S1, unofficial].
*   **When Not to Use:** Adaptive resolution data requiring complex boundary sharing, or when interactive rotation is required with strict visual fidelity (see Parallel Distributed 2D Texture Slicing) [S2, unofficial].

## Background
The method originates from parallel implementations of slice-based volume rendering using MPI, as demonstrated in open-source repositories and academic projects [S1, unofficial]. It builds on the fundamental concept of domain decomposition in distributed computing, applied to the specific case of volume rendering where the slicing axis is the natural decomposition dimension [S1, unofficial]. Variants like Parallel Distributed 2D Texture Slicing were proposed by Bethel et al. to handle large, remotely located scientific data on low-end workstations [S2, unofficial].

## Sources
[S1] SREESUBBARAYUDU/Parallel_Volume_Rendering, https://github.com/SREESUBBARAYUDU/Parallel_Volume_Rendering, fetched 2026-10-27 (unofficial)
[S2] Volumetric Medical Images Visualization on Mobile Devices, http://hdl.handle.net/2099.1/11308, fetched 2026-10-26 (unofficial)
