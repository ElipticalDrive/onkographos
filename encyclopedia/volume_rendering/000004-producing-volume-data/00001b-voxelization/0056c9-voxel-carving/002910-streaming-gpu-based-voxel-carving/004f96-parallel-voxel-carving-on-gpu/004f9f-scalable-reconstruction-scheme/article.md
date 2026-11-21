# Scalable Reconstruction Scheme

Also called: Streaming GPU-based Voxel Carving for Large-Scale Volumes

## Overview
A method for reconstructing large-scale urban scenes by partitioning the global volume into fixed-size sub-volumes that fit within GPU memory, processing them sequentially via streaming, and aggregating surface voxels on the CPU. It differs from monolithic GPU voxel carving by overcoming VRAM limits through a stream-processing architecture, enabling the reconstruction of volumes significantly larger than the available video memory [S1, unofficial].

## How it works
**Inputs:**
*   Triangulated 3D points (to determine bounding box).
*   Depth maps, source images, and camera projection matrices.
*   Global volume dimensions (e.g., $N_x \times N_y \times N_z$).

**Outputs:**
*   A set of surface voxels with associated positions, colors, and compressed plane information.

**Data Structures:**
*   **Sub-volume Stream:** A data structure transferred between CPU and GPU.
    *   Fields: `volume_size` (global bounds of the sub-volume), `position` (offset in global grid), `resolution` (local resolution, typically $256^3$).
    *   Voxel Array: A 3D array of voxels. Initial state: all voxels marked as "uncarved" [S1, unofficial].
*   **GPU Memory:** Stores all depth maps, images, and projection matrices prior to carving [S1, unofficial].

**Parameters:**
*   **Sub-volume Size:** Fixed at $256^3$ voxels to fit within the GPU memory of the target hardware (e.g., 512 MB VRAM) [S1, unofficial].
*   **Grid Partitioning:** The global volume is divided into a grid of sub-volumes. For a $2560 \times 256 \times 2560$ grid, this results in $10 \times 1 \times 10$ sub-volumes [S1, unofficial].

**Steps:**
1.  **Bounding Box Determination:** Calculate the minimum bounding box of the input triangulated 3D points to define the global reconstruction region [S1, unofficial].
2.  **GPU Initialization:** Upload all generated depth maps, images, and camera projection matrices to GPU memory [S1, unofficial].
3.  **Stream Preparation:** Subdivide the global volume into sub-volumes of size $256^3$. Initialize each sub-volume stream with its size, position, resolution, and set all internal voxels to "uncarved" [S1, unofficial].
4.  **Streaming Execution:**
    a. Transfer one input sub-volume stream from main memory to GPU memory.
    b. Execute the parallel voxel carving kernel on the GPU for this sub-volume. The kernel processes voxels in parallel, using the pre-loaded depth maps and projection matrices to determine visibility [S1, unofficial].
    c. Transfer the output carved sub-volume stream back to main memory. The output contains only surface voxels with color and compressed plane information [S1, unofficial].
5.  **Surface Extraction:** On the CPU, iterate through the returned streams. Extract the surface voxels and store them along with their global voxel positions [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def scalable_reconstruction(triangulated_points, depth_maps, images, projection_matrices, global_dims):
    # Step 1: Determine global bounds
    min_bbox, max_bbox = compute_min_max_bbox(triangulated_points)
    
    # Step 2: Upload static data to GPU
    gpu_depth_maps = upload_to_gpu(depth_maps)
    gpu_images = upload_to_gpu(images)
    gpu_projections = upload_to_gpu(projection_matrices)
    
    # Define sub-volume parameters
    SUB_VOL_SIZE = 256
    sub_vol_dims = compute_sub_vol_grid(global_dims, SUB_VOL_SIZE)
    
    surface_voxels = []
    
    # Step 3 & 4: Stream processing loop
    for sub_vol_pos in sub_vol_dims:
        # Create sub-volume stream
        sub_vol_stream = create_sub_volume_stream(
            position=sub_vol_pos,
            resolution=SUB_VOL_SIZE,
            global_bounds=(min_bbox, max_bbox)
        )
        # Initialize all voxels as uncarved
        sub_vol_stream.voxels = [Voxel(status="uncarved") for _ in range(SUB_VOL_SIZE**3)]
        
        # Transfer to GPU
        gpu_sub_vol = upload_to_gpu(sub_vol_stream)
        
        # Execute carving kernel
        # The kernel uses gpu_depth_maps, gpu_images, gpu_projections
        # and modifies gpu_sub_vol.voxels in place
        execute_voxel_carving_kernel(gpu_sub_vol, gpu_depth_maps, gpu_images, gpu_projections)
        
        # Transfer back to CPU
        carved_sub_vol = download_from_gpu(gpu_sub_vol)
        
        # Step 5: Extract surface voxels
        for voxel in carved_sub_vol.voxels:
            if voxel.status == "surface":
                global_pos = voxel.position + carved_sub_vol.position
                surface_voxels.append({
                    "position": global_pos,
                    "color": voxel.color,
                    "plane_info": voxel.plane_info
                })
                
    return surface_voxels
```

**Edge Cases and Pitfalls:**
*   **Memory Limits:** The sub-volume size is constrained by the specific GPU's VRAM. If VRAM is insufficient for $256^3$ voxels plus auxiliary data, the sub-volume size must be reduced, increasing overhead [S1, unofficial].
*   **Boundary Consistency:** Since subdivision does not affect the final output, boundary handling between adjacent sub-volumes must be consistent with the global projection logic to avoid artifacts [S1, unofficial].

## Operating characteristics
*   **Scalability:** The method scales to volumes significantly larger than GPU memory. Tested on grids up to $10,240 \times 256 \times 5,120$ voxels [S1, unofficial].
*   **Performance:** On an NVIDIA GeForce 9800 GT (512 MB VRAM), processing $10 \times 1 \times 10$ sub-volumes ($256^3$ each) for a $2,560 \times 256 \times 2,560$ grid yielded ~5.5M surface voxels [S1, unofficial]. Larger Walnut Creek tests ($10,240 \times 256 \times 5,120$) produced 19M surface voxels [S1, unofficial].
*   **Quality:** Produces denser, more detailed results than PMVS2, with fewer holes or gaps, by integrating multiple depth maps into a watertight model [S1, unofficial].
*   **Limitations:** Requires sequential processing of sub-volumes, which may limit throughput compared to fully parallel approaches if I/O bandwidth is a bottleneck. Dependent on GPU memory capacity for the fixed sub-volume size [S1, unofficial].

## Background
Developed for large-scale urban scene reconstruction using multi-view stereo. The method addresses the rapid performance drop and memory limitations of standard GPU-based volumetric reconstruction when applied to large datasets. It builds on parallel voxel carving techniques by introducing a streaming mechanism [S1, unofficial].

## Sources
[S1] GPU-based Scalable Volumetric Reconstruction for Multi-view Stereo, https://digital.library.unt.edu/ark:/67531/metadc831231/?q=gpu volume rendering, fetched 2026-10-28 (unofficial)
