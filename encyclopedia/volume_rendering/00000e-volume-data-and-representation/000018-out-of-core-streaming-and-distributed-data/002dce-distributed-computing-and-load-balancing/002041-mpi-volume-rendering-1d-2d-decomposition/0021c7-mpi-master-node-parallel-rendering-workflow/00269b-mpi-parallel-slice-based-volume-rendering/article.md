# MPI Parallel Slice-Based Volume Rendering

Also called: Distributed Slice Rendering, Domain Decomposition Slicing, MPI Parallel Slicing, Parallel Slice Rendering, Parallel Volume Rendering

## Overview
MPI Parallel Slice-Based Volume Rendering is a distributed computing method that decomposes a 3D volume dataset along the slicing axis (typically the Z-axis) among multiple MPI processes [S1, unofficial]. Each process independently renders its assigned subset of slices using front-to-back compositing and transfers the resulting local image buffers to a root process for final assembly [S1, unofficial]. It differs from ray-casting-based parallel methods by operating on discrete planar slices rather than continuous rays, making it suitable for scenarios where slice-based compositing is preferred or required [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** A 3D scalar volume dataset (e.g., VTI file), the number of MPI processes (`size`), and the current process rank (`rank`) [S1, unofficial].
- **Output:** A composite 2D image assembled by the root process [S1, unofficial].

**Data Structures:**
- **Volume Data:** A 3D NumPy array. After transformation for the view axis, the shape is $(W, H, D)$, where $W$ is width, $H$ is height, and $D$ is depth (number of slices) [S1, unofficial].
- **Local Image Buffer:** A 3D NumPy array of shape $(H, D_{local}, 3)$, where $D_{local}$ is the number of slices assigned to the current rank [S1, unofficial].
- **Total Data Buffer (Root only):** A 3D NumPy array accumulating the images from all ranks [S1, unofficial].

**Parameters:**
- `size`: Total number of MPI processes [S1, unofficial].
- `rank`: Unique identifier of the current process ($0 \le rank < size$) [S1, unofficial].
- **Transfer Function:** The source code uses a Gaussian-based transfer function for color and a piecewise function for opacity [S1, unofficial].
    - **Color Transfer Function:** Maps scalar values to RGB using Gaussian kernels centered at specific points [S1, unofficial].
    - **Opacity Transfer Function:** Defined by points $(-5000.0, 1.0)$ and $(2600.0, 0.0)$ in the source code [S1, unofficial].
    - **Scalar Range:** The source code applies `np.log` to the data before transfer function application [S1, unofficial].

**Numbered Steps:**
1. **Initialize MPI:** Create the MPI communicator and determine `rank` and `size` [S1, unofficial].
2. **Load and Transform Data:** Read the volume data (e.g., from a VTI file) and convert it to a NumPy array [S1, unofficial]. Transform the array orientation based on the view axis; for a Z-axis view, the data is transposed such that the depth dimension corresponds to the last axis [S1, unofficial].
3. **Calculate Slice Range:** Determine the range of slice indices $[p1, p2)$ assigned to the current rank [S1, unofficial]. The volume depth $D$ is divided among `size` processes [S1, unofficial]. The last process receives any remaining slices if $D$ is not evenly divisible by `size` [S1, unofficial].
4. **Extract Local Data:** Slice the `camera_grid` to extract the local subset of slices [S1, unofficial].
5. **Initialize Local Image Buffer:** Create a buffer for the rendered result with shape $(H, D_{local}, 3)$ [S1, unofficial].
6. **Render Local Slices:** Iterate through each slice in `camera_grid` [S1, unofficial]. Apply the transfer function to map scalar values to color and opacity [S1, unofficial]. Perform front-to-back compositing into the local image buffer [S1, unofficial]. The source code applies `np.log` to the slice data before transfer function evaluation [S1, unofficial].
7. **Send/Receive Images:** Non-root processes send their local `image` buffer to the root process (rank 0) [S1, unofficial]. The root process initializes `total_data` with its own image, then receives images from all other ranks and concatenates them along axis 1 [S1, unofficial].
8. **Display/Save:** The root process displays or saves the `total_data` image [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
import matplotlib.pyplot as plt

def transform_array(data, view_axis):
    if view_axis == 'z':
        return np.transpose(data, (2, 1, 0))
    raise ValueError("Only 'z' view axis implemented in source")

def color_tf(x):
    r = 1.0*np.exp(-(x - 9.0)**2/1.0) + 0.1*np.exp(-(x - 3.0)**2/0.1) + 0.1*np.exp(-(x + 3.0)**2/0.5)
    g = 1.0*np.exp(-(x - 9.0)**2/1.0) + 1.0*np.exp(-(x - 3.0)**2/0.1) + 0.1*np.exp(-(x + 3.0)**2/0.5)
    b = 0.1*np.exp(-(x - 9.0)**2/1.0) + 0.1*np.exp(-(x - 3.0)**2/0.1) + 1.0*np.exp(-(x + 3.0)**2/0.5)
    return r, g, b

def opacity_tf(x):
    alpha = np.clip(1.0 - (x + 5000.0) / (2600.0 + 5000.0), 0.0, 1.0)
    return alpha

def render_mpi_slice(comm, rank, size, original_array):
    # 1. Initialize MPI communicator and get rank and size
    # comm = MPI.COMM_WORLD
    # rank = comm.Get_rank()
    # size = comm.Get_size()

    # 2. Load and transform data
    camera_grid = transform_array(original_array, 'z')
    a = camera_grid.shape  # (W, H, D)

    # 3. Calculate slice range
    p1 = (a[2] // size) * rank
    if size != rank + 1:
        p2 = (a[2] // size) * (rank + 1)
    else:
        p2 = a[2]

    # 4. Extract local data
    camera_grid = camera_grid[:, :, p1:p2]

    # 5. Initialize local image buffer
    image = np.zeros((camera_grid.shape[1], camera_grid.shape[2], 3))

    # 6. Render local slices
    for dataslice in camera_grid:
        log_slice = np.log(dataslice)
        r, g, b = color_tf(log_slice)
        alpha = opacity_tf(log_slice)
        # Transpose to match image dimensions (H, D_local)
        alpha = alpha.T
        r = r.T
        g = g.T
        b = b.T
        image[:, :, 0] = alpha * r + (1 - alpha) * image[:, :, 0]
        image[:, :, 1] = alpha * g + (1 - alpha) * image[:, :, 1]
        image[:, :, 2] = alpha * b + (1 - alpha) * image[:, :, 2]

    # 7. Send/Receive Images
    if rank != 0:
        comm.Send(image, dest=0)
    else:
        total_data = image
        for k in range(1, size):
            comm.Recv(image, source=k)
            total_data = np.concatenate((total_data, image), 1)

    # 8. Display/Save
    if rank == 0:
        plt.imshow(total_data)
        plt.axis('off')
        plt.show()
```

**Edge Cases and Pitfalls:**
- **Uneven Division:** If the number of slices $D$ is not divisible by `size`, the last process must receive the remainder [S1, unofficial]. The conditional `if size != rank + 1` handles this by setting `p2` to the full depth for the last rank [S1, unofficial].
- **Concatenation Axis:** The source code uses `axis=1` for concatenation, but the slices are along `axis=2` [S1, unofficial]. This mismatch leads to a distorted final image if not corrected by swapping dimensions or adjusting the concatenation axis to match the slice dimension [S1, unofficial].
- **Index Calculation:** The formula `p2 = (int)((a[2] // size) * (rank + 1)) - 1` in some source variants may exclude the last slice if not adjusted, leading to data loss [S1, unofficial].

## Subtopics

### Parallel Distributed 2D Texture Slicing
An engine with parallel distributed implementation for visualizing large, remotely located, time-varying, adaptive, hierarchical scientific data that won't fit on a workstation [S2, unofficial]. It uses a 2D texture-based method with axis-aligned slicing to achieve interactive transformation on low-end workstations [S2, unofficial]. Each processor renders a subset of the volume in 2D textures, which are combined into a low-cost hardware graphics pipeline [S2, unofficial]. The resulting image is used as raw texture data applied to an axis-aligned quad for volume rendering, with image warping and depth-order compositing performed using graphics hardware [S2, unofficial]. Unlike architectures that compress and decompress images, this technique uses a stage of partition and composition [S2, unofficial]. It has a limited rotation range to avoid visual degradation, which is data-dependent and can be adjusted by changing the number of texture maps [S2, unofficial]. No compression techniques are used [S2, unofficial].

### Boundary Sharing for Adaptive-Resolution Blocks
A technique to remove boundary artifacts in parallel volume rendering of block-divided data by sharing slices between adjacent blocks, adapted for 3D blocks rather than 2D slices [S3, unofficial]. It identifies sides of a data block that coincide with actual data boundaries (ignored for sharing) and shares corresponding slices on all remaining internal sides [S3, unofficial]. A central block shares 6 slices (left, right, front, back, top, bottom), while corner blocks share only necessary internal slices [S3, unofficial]. For adaptive-resolution datasets, shared slices are expanded or contracted to match the resolution of adjacent blocks to maintain consistent resolution within each block [S3, unofficial]. Partial images are rendered and composited recursively in pairs until a single final image is generated [S3, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N/P)$ for rendering, where $N$ is the total number of slices and $P$ is the number of processes, assuming uniform distribution [S1, unofficial]. Communication overhead is $O(P)$ for sending images to the root [S1, unofficial].
- **Memory Complexity:** $O(N/P)$ per process for local data and image buffers [S1, unofficial]. The root process requires $O(N)$ memory to store the final image [S1, unofficial].
- **Performance:** Performance scales with the number of processes, limited by communication bandwidth and the speed of the root process's image assembly [S1, unofficial].
- **Strengths:** Simple implementation, leverages existing slice-based rendering logic, suitable for distributed memory systems [S1, unofficial].
- **Limitations:** Communication bottleneck at the root process, potential for incorrect image assembly due to axis mismatch, load imbalance if volume data is not uniform or slice processing times vary [S1, unofficial].

## Background
This method is based on a parallel implementation of slice-based volume rendering using MPI, as demonstrated in open-source code repositories [S1, unofficial]. It builds on the principles of domain decomposition and master-slave parallelism in volume rendering [S1, unofficial].

## Sources
[S1] SREESUBBARAYUDU/Parallel_Volume_Rendering, https://github.com/SREESUBBARAYUDU/Parallel_Volume_Rendering, fetched 2026-10-27 (unofficial)
[S2] Volumetric Medical Images Visualization on Mobile Devices, http://hdl.handle.net/2099.1/11308, fetched 2026-10-26 (unofficial)
[S3] Volume visualization of time-varying data using parallel, multiresolution and adaptive-resolution techniques, https://scholars.unh.edu/cgi/viewcontent.cgi?article=1245&context=thesis, fetched 2026-10-27 (unofficial)
