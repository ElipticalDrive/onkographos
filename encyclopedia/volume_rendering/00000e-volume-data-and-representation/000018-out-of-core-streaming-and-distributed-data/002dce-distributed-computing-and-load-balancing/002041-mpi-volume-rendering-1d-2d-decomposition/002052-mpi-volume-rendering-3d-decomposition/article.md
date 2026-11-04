# MPI Volume Rendering (3D Decomposition)

Also called: MPI Domain Decomposition, Distributed 3D Domain Decomposition, MPI Ray Casting

## Overview
MPI Volume Rendering (3D Decomposition) is a parallel volume rendering technique that partitions a 3D scalar dataset into a 3D grid of subdomains distributed across multiple MPI processes [S1, S2, S3, S4, unofficial]. Each process independently ray casts its assigned volume chunk and image region, then stitches the resulting image slices to form a final composite image [S1, S4, unofficial]. It differs from 1D/2D decomposition by distributing data along all three spatial axes (X, Y, Z), allowing for finer-grained load balancing and scalability on distributed memory systems [S1, S2, S3, S4, unofficial].

## How it works
### Inputs and Outputs
**Inputs:**
*   **Volume Dataset:** A `.raw` file containing 3D scalar volume data, typically `float32` format (e.g., `Isabel_1000x1000x200_float32.raw`) [S1, S3, S4, unofficial].
*   **Color Transfer Function (TF):** A text file with entries formatted as `(scalar, R, G, B)` mapping scalar values to RGB color components [S1, S3, S4, unofficial].
*   **Opacity Transfer Function (TF):** A text file with entries formatted as `(scalar, opacity)` mapping scalar values to opacity [S1, S3, S4, unofficial].
*   **Decomposition Parameters:** Integers `xdiv`, `ydiv`, `zdiv` (or `PX`, `PY`, `PZ`) specifying the number of divisions along each axis, such that `P = xdiv * ydiv * zdiv` equals the total number of MPI processes [S1, S2, S3, unofficial].
*   **Ray Step Size:** A float `step_size` determining the sampling interval along the ray [S1, S3, S4, unofficial].
*   **Optional Bounding Box:** `x_min`, `x_max`, `y_min`, `y_max` for cropping the dataset before decomposition [S4, unofficial].

**Outputs:**
*   **Final Composite Image:** A 2D RGB image stitched from the sub-images rendered by each process [S1, S4, unofficial].
*   **Performance Metrics:** Timing for decomposition, data loading, ray casting, and communication [S1, S4, unofficial].

### Data Structures
*   **Volume Dataset:** Stored as a contiguous 1D array or 3D NumPy array with shape `(X, Y, Z)` in Fortran order (`order='F'`) [S1, S3, S4, unofficial].
*   **Boundaries:** Arrays `xBound`, `yBound`, `zBound` of size `xdiv+1`, `ydiv+1`, `zdiv+1` respectively, defining the start and end indices for each partition [S2, unofficial].
*   **Local Subdomain:** A 1D or 3D array `eachDataset` of size `eachXlen * eachYlen * eachZlen` containing the volume data assigned to the current rank [S2, S4, unofficial].
*   **Transfer Functions:** Lists of tuples `(value, color_tuple)` or `(value, opacity)` parsed from text files [S3, S4, unofficial].
*   **Image Slice:** A 2D array of shape `(height, width, 3)` representing the rendered portion of the image plane assigned to the rank [S3, S4, unofficial].

### Numbered Steps

1.  **Initialize MPI:** Initialize the MPI environment, get the process rank and total size [S3, S4, unofficial].
2.  **Parse Arguments:** Read command-line arguments for dataset filename, decomposition dimensions (`xdiv`, `ydiv`, `zdiv`), step size, and TF file paths [S1, S3, S4, unofficial].
3.  **Calculate Boundaries (Rank 0):**
    *   Initialize `xBound[0] = 0` and `xBound[xdiv] = X`.
    *   Calculate remainder `rem = X % xdiv`.
    *   For `i` from 1 to `xdiv-1`: `xBound[i] = xBound[i-1] + X // xdiv + (1 if rem > 0 else 0)`, then decrement `rem` [S2, unofficial].
    *   Repeat for Y and Z dimensions [S2, unofficial].
4.  **Load Data (Rank 0):**
    *   Load the volume dataset from the `.raw` file using `np.fromfile` with `dtype=np.float32` and reshape to `(X, Y, Z)` using Fortran order [S1, S3, S4, unofficial].
    *   If bounding box coordinates are provided, crop the dataset to `[x_min, x_max]`, `[y_min, y_max]`, clamping to data bounds [S4, unofficial].
5.  **Decompose Data (Rank 0):**
    *   Split the volume into subdomains based on the 3D grid defined by `xBound`, `yBound`, `zBound` [S2, S4, unofficial].
    *   Alternatively, use Type 1 (split along X) or Type 2 (split into 2D grid of X/Y slices) decomposition logic if specified [S4, unofficial].
6.  **Distribute Data:**
    *   Rank 0 sends each subdomain to the corresponding MPI rank using `MPI_Send` with a unique tag [S2, S4, unofficial].
    *   Non-zero ranks receive their assigned subdomain using `MPI_Recv` with the tag matching their rank [S2, S4, unofficial].
    *   Alternatively, use `communicator.scatter` to distribute the list of subdomains [S3, unofficial].
7.  **Load Transfer Functions (All Ranks):**
    *   Rank 0 reads the Color and Opacity TF files, parses them into lists of tuples, and broadcasts them to all ranks using `MPI_Bcast` [S3, S4, unofficial].
    *   Non-zero ranks receive the transfer functions [S3, S4, unofficial].
8.  **Ray Casting (All Ranks):**
    *   For each pixel `(x, y)` in the local image slice:
        *   Initialize `accumulated_color = [0, 0, 0]` and `accumulated_opacity = 0` [S4, unofficial].
        *   March along the Z-axis with `step_size`:
            *   Calculate the continuous Z coordinate `z` [S4, unofficial].
            *   Sample the volume data at `(x, y, z)` using trilinear interpolation (or linear interpolation along Z if only Z is varied) [S3, S4, unofficial].
                *   `z0 = int(z)`, `z1 = min(z0 + 1, data.shape[2] - 1)` [S4, unofficial].
                *   `t = z - z0` [S4, unofficial].
                *   `voxel_val = (1 - t) * data[x, y, z0] + t * data[x, y, z1]` [S4, unofficial].
            *   Interpolate Color: Find bracketing interval `[v0, v1]` in Color TF. If `v1 == v0`, use `c0`. Else, `t = (voxel_val - v0) / (v1 - v0)`, `color = c0 + t * (c1 - c0)` [S3, S4, unofficial].
            *   Interpolate Opacity: Find bracketing interval `[v0, v1]` in Opacity TF. If `v1 == v0`, use `o0`. Else, `t = (voxel_val - v0) / (v1 - v0)`, `opacity = o0 + t * (o1 - o0)` [S3, S4, unofficial].
            *   Composite: `accumulated_color += (1 - accumulated_opacity) * color * opacity` [S3, S4, unofficial].
            *   `accumulated_opacity += (1 - accumulated_opacity) * opacity` [S3, S4, unofficial].
            *   **Early Termination:** If `accumulated_opacity >= 0.98` (or 0.95 in some implementations), break the ray march and count as terminated [S3, S4, unofficial].
            *   `z += step_size` [S3, S4, unofficial].
        *   Store `accumulated_color` in the local image slice at `(x, y)` [S3, S4, unofficial].
9.  **Gather Results (All Ranks):**
    *   Each rank sends its rendered image slice to Rank 0 using `MPI_Send` or `communicator.gather` [S3, S4, unofficial].
    *   Rank 0 gathers all image slices [S3, S4, unofficial].
10. **Stitch Image (Rank 0):**
    *   Reconstruct the final image from the gathered slices based on the decomposition type [S4, unofficial].
    *   **Type 1 (1D split):** Stack vertically using `np.vstack` [S3, S4, unofficial].
    *   **Type 2 (2D split):** Reconstruct the 2D grid by stacking columns vertically then horizontally using `np.hstack` and `np.vstack` [S3, S4, unofficial].
    *   Convert to `uint8` and save as PNG [S3, S4, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
from mpi4py import MPI

def interpolate_tf(tf_list, value):
    """Interpolate value from a list of (scalar, result) tuples."""
    for i in range(len(tf_list) - 1):
        s0, r0 = tf_list[i]
        s1, r1 = tf_list[i+1]
        if s0 <= value <= s1:
            if s1 == s0:
                return r0
            t = (value - s0) / (s1 - s0)
            if isinstance(r0, (list, np.ndarray)):
                return [r0[j] + t * (r1[j] - r0[j]) for j in range(len(r0))]
            else:
                return r0 + t * (r1 - r0)
    return tf_list[0][1] if tf_list else 0

def ray_cast_local(subdomain, color_tf, opacity_tf, step_size):
    height, width, depth = subdomain.shape
    img = np.zeros((height, width, 3))
    terminated_rays = 0
    
    for y in range(width):
        for x in range(height):
            acc_color = np.zeros(3)
            acc_opacity = 0.0
            z = 0.0
            while z < depth:
                # Linear interpolation along Z
                z0 = int(z)
                z1 = min(z0 + 1, depth - 1)
                t = z - z0
                voxel_val = (1 - t) * subdomain[x, y, z0] + t * subdomain[x, y, z1]
                
                # Interpolate TFs
                color = interpolate_tf(color_tf, voxel_val)
                opacity = interpolate_tf(opacity_tf, voxel_val)
                
                # Front-to-back compositing
                acc_color += (1 - acc_opacity) * color * opacity
                acc_opacity += (1 - acc_opacity) * opacity
                
                if acc_opacity >= 0.98:
                    terminated_rays += 1
                    break
                z += step_size
            img[x, y, :] = acc_color
    return img, terminated_rays

def main():
    comm = MPI.COMM_WORLD
    rank = comm.Get_rank()
    size = comm.Get_size()
    
    # Parse args (example for S4 style)
    dataset_name = sys.argv[1]
    xdiv = int(sys.argv[2])
    ydiv = int(sys.argv[3])
    zdiv = int(sys.argv[4])
    step_size = float(sys.argv[5])
    color_tf_file = sys.argv[6]
    opacity_tf_file = sys.argv[7]
    
    # Load TFs on Rank 0
    if rank == 0:
        with open(color_tf_file) as f:
            color_tf = [tuple(map(float, line.split())) for line in f if line.strip()]
        with open(opacity_tf_file) as f:
            opacity_tf = [tuple(map(float, line.split())) for line in f if line.strip()]
        # Load volume data
        # Assume shape is known or parsed from filename
        volume = np.fromfile(dataset_name, dtype=np.float32).reshape((X, Y, Z), order='F')
        
        # Calculate boundaries
        x_bound = [0] * (xdiv + 1)
        y_bound = [0] * (ydiv + 1)
        z_bound = [0] * (zdiv + 1)
        
        rem = X % xdiv
        for i in range(1, xdiv):
            x_bound[i] = x_bound[i-1] + X // xdiv + (1 if rem > 0 else 0)
            rem -= 1
        x_bound[xdiv] = X
        
        rem = Y % ydiv
        for i in range(1, ydiv):
            y_bound[i] = y_bound[i-1] + Y // ydiv + (1 if rem > 0 else 0)
            rem -= 1
        y_bound[ydiv] = Y
        
        rem = Z % zdiv
        for i in range(1, zdiv):
            z_bound[i] = z_bound[i-1] + Z // zdiv + (1 if rem > 0 else 0)
            rem -= 1
        z_bound[zdiv] = Z
        
        # Decompose and send
        for r in range(1, size):
            px = r % xdiv
            py = (r // xdiv) % ydiv
            pz = (r // (xdiv * ydiv)) % zdiv
            
            x_start, x_end = x_bound[px], x_bound[px+1]
            y_start, y_end = y_bound[py], y_bound[py+1]
            z_start, z_end = z_bound[pz], z_bound[pz+1]
            
            sub_vol = volume[x_start:x_end, y_start:y_end, z_start:z_end]
            comm.send(sub_vol, dest=r, tag=r)
            
        # Rank 0 also processes its own chunk
        px = 0; py = 0; pz = 0
        x_start, x_end = x_bound[px], x_bound[px+1]
        y_start, y_end = y_bound[py], y_bound[py+1]
        z_start, z_end = z_bound[pz], z_bound[pz+1]
        local_vol = volume[x_start:x_end, y_start:y_end, z_start:z_end]
    else:
        local_vol = comm.recv(source=0, tag=rank)
        color_tf = None
        opacity_tf = None
        
    # Broadcast TFs
    color_tf = comm.bcast(color_tf, root=0)
    opacity_tf = comm.bcast(opacity_tf, root=0)
    
    # Ray Cast
    img_slice, terminated = ray_cast_local(local_vol, color_tf, opacity_tf, step_size)
    
    # Gather
    all_imgs = comm.gather(img_slice, root=0)
    all_terminated = comm.gather(terminated, root=0)
    
    if rank == 0:
        # Stitch image
        # Assuming Type 2 decomposition for 3D grid
        # Reconstruct grid based on xdiv, ydiv, zdiv
        # Note: The image plane is 2D (X, Y). Z is depth.
        # The decomposition splits the volume, but the image plane is split by X and Y partitions.
        # The number of image rows/cols depends on how the image plane is partitioned.
        # In 3D decomposition, the image plane is typically partitioned into xdiv * ydiv regions.
        
        # Flatten the list of images into a 2D grid of images
        # The order of ranks in comm.gather is 0..size-1
        # Rank r corresponds to (px, py, pz)
        # The image slice for rank r covers x in [x_start, x_end) and y in [y_start, y_end)
        
        # Create a large canvas
        final_img = np.zeros((Y, X, 3))
        
        for r in range(size):
            px = r % xdiv
            py = (r // xdiv) % ydiv
            pz = (r // (xdiv * ydiv)) % zdiv
            
            x_start = x_bound[px]
            y_start = y_bound[py]
            
            img_slice = all_imgs[r]
            h, w, _ = img_slice.shape
            
            final_img[y_start:y_start+h, x_start:x_start+w, :] = img_slice
            
        final_img = (final_img * 255).astype(np.uint8)
        from PIL import Image
        Image.fromarray(final_img).save('Rendered_Image.png')

if __name__ == '__main__':
    main()
```

### Parameters
*   `xdiv`, `ydiv`, `zdiv`: Number of divisions along X, Y, Z axes. `P = xdiv * ydiv * zdiv` [S1, S2, S3, unofficial].
*   `step_size`: Ray marching step size (e.g., 0.5) [S1, S3, S4, unofficial].
*   `input_dataset`: Path to `.raw` file [S1, unofficial].
*   `color_tf`, `opacity_tf`: Paths to TF files [S1, unofficial].

### Edge Cases and Pitfalls
*   **Boundary Calculation:** The remainder `X % xdiv` must be distributed to the first blocks. If `rem` is not decremented correctly, block sizes will be incorrect [S2, unofficial].
*   **Division by Zero:** Transfer function interpolation must check if `v1 == v0` to avoid division by zero [S3, unofficial].
*   **Index Out of Bounds:** When interpolating Z values, `z1` must be clamped to `data.shape[2] - 1` [S4, unofficial].
*   **Data Order:** Volume data must be loaded and reshaped in the correct order (Fortran order is specified in some implementations) [S1, S3, S4, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(P \cdot \frac{V}{P} \cdot R)$, where $P$ is the number of processes, $V$ is the volume size, and $R$ is the number of rays per process. Communication overhead is $O(V)$ for data distribution and $O(I)$ for image gathering, where $I$ is the image size [S1, S4, unofficial].
*   **Memory Complexity:** $O(V/P)$ per process for the local volume chunk [S2, S4, unofficial].
*   **Scaling:** Scales well with the number of processes if the volume is large enough to balance the communication overhead [S1, S4, unofficial].
*   **Strengths:** Distributed memory parallelism allows rendering of very large datasets that do not fit in a single node's memory [S1, S4, unofficial].
*   **Limitations:** Communication overhead for data distribution and image gathering can become significant for small datasets or high process counts [S1, S4, unofficial]. Load imbalance can occur if the volume density is uneven [S4, unofficial].

## Background
This method is implemented in various open-source projects such as `Parallel_Distribution_Axis-Aligned_Volume_Rendering` and `Parallel-3D-Volume-Rendering-with-MPI` [S1, S4, unofficial]. It builds on the principles of MPI-based parallel computing and volume ray casting [S1, S4, unofficial].

## Sources
[S1] adarshmudgal4612/Parallel-Volume-Rendering, https://github.com/adarshmudgal4612/Parallel-Volume-Rendering, fetched 2026-10-27 (unofficial)
[S2] rishit-AI/parallel-volume-rendering-with-binary-swap, https://github.com/rishit-AI/parallel-volume-rendering-with-binary-swap, fetched 2026-10-27 (unofficial)
[S3] yugul7714/Parallel-Volume-Rendering-using-MPI-and-Visualization, https://github.com/yugul7714/Parallel-Volume-Rendering-using-MPI-and-Visualization, fetched 2026-10-27 (unofficial)
[S4] jhadivyanshu24/Parallel-3D-Volume-Rendering-with-MPI-CS677, https://github.com/jhadivyanshu24/Parallel-3D-Volume-Rendering-with-MPI-CS677, fetched 2026-10-27 (unofficial)
