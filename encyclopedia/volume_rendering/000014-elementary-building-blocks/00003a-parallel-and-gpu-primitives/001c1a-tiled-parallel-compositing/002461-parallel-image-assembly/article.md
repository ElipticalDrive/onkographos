# Parallel Image Assembly

Also called: Final Image Assembly & Alpha Blending, Image Stitching, Spatial Compositing, Z-Order Compositing

## Overview
Parallel Image Assembly is the final step in spatial decomposition volume rendering where sub-images rendered by independent threads or processes are combined into a single final image [S1, unofficial]. It solves the problem of merging disjoint spatial data portions into a coherent visual output by placing sub-images at their correct spatial coordinates and blending them along the z-axis using front-to-back alpha compositing [S2, unofficial]. Unlike temporal sequencing, which has minimal overhead, spatial compositing incurs significant overhead due to the data collection and blending requirements [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of composited sub-images (RGBA) and their corresponding opacities from all participating ranks/threads. Each sub-image corresponds to a specific spatial tile $(px, py)$ and potentially a depth slice $pz$.
*   **Output:** A single final image array of shape $(X, Y, 4)$ with dtype `float32`, representing the fully composited volume, often saved as `uint8` PNG [S2, unofficial].

**Data Structures:**
*   `final_image`: A numpy array of shape $(x\_dim, y\_dim, 4)$ storing the accumulated color and opacity [S2, unofficial].
*   `z_split_images`: A collection of sub-images arranged by their spatial coordinates [S2, unofficial].
*   `total_opacity`: An array tracking the accumulated opacity for each pixel location [S3, unofficial].

**Parameters:**
*   `LOCATION_OF_Z`: A sorted list of unique z-start coordinates for each process chunk, used to determine depth ordering [S2, unofficial].
*   `PZ`: Total number of depth slices [S3, unofficial].

**Steps:**
1.  **Gather Sub-images:** Collect the composited subdomain images and opacities from all ranks to a root rank (typically Rank 0) [S2, unofficial][S3, unofficial].
2.  **Initialize Final Image:** On the root rank, initialize `final_image` with zeros and shape $(PX \times sub\_height, PY \times sub\_width, 3)$ for color, and a corresponding opacity array [S3, unofficial].
3.  **Place Sub-images:** Iterate through all gathered ranks. For each rank, determine its spatial coordinates $(px, py, pz)$ [S3, unofficial].
4.  **Depth Handling:** If the rank corresponds to a specific depth slice (e.g., the middle slice $pz == PZ // 2$), place its composited color image into the `final_image` grid at the appropriate $(px, py)$ location [S3, unofficial]. Add the opacity from this slice to the `total_opacity` map for the corresponding $(px, py)$ location [S3, unofficial].
5.  **Compositing:** For each z-slice (or depth layer), blend the sub-image into the `final_image` using front-to-back alpha compositing [S2, unofficial]. The blending follows the standard volume rendering equation:
    $$ C_{final} = C_{prev} + C_{z} \cdot (1 - \alpha_{prev}) $$
    $$ \alpha_{final} = \alpha_{prev} + \alpha_{z} \cdot (1 - \alpha_{prev}) $$
    Where $C$ is color (RGB) and $\alpha$ is opacity [S2, unofficial].
6.  **Finalization:** Convert the `final_image` from `float32` to `uint8` RGBA and save as a PNG file [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def parallel_image_assembly(gathered_sub_images, gathered_opacities, px, py, pz, sub_width, sub_height, x_dim, y_dim):
    """
    Assembles sub-images into a final image using spatial compositing.
    
    Args:
        gathered_sub_images: List of arrays from all ranks, each of shape (sub_width, sub_height, 4)
        gathered_opacities: List of arrays from all ranks, each of shape (sub_width, sub_height, 1)
        px, py, pz: Spatial coordinates of the current rank's sub-image
        sub_width, sub_height: Dimensions of each sub-image
        x_dim, y_dim: Total dimensions of the final image
        
    Returns:
        final_image: numpy array of shape (x_dim, y_dim, 4)
    """
    # Initialize final image arrays
    final_color = np.zeros((x_dim, y_dim, 3), dtype=np.float32)
    final_opacity = np.zeros((x_dim, y_dim, 1), dtype=np.float32)
    
    # Determine the position in the final image grid
    start_x = px * sub_width
    start_y = py * sub_height
    
    # Place the sub-image into the final image grid
    # Note: In a full parallel assembly, this loop would iterate over all ranks
    # Here we assume the current rank's data is being placed
    final_color[start_x:start_x+sub_width, start_y:start_y+sub_height] = gathered_sub_images[:, :, :3]
    final_opacity[start_x:start_x+sub_width, start_y:start_y+sub_height] = gathered_sub_images[:, :, 3:4]
    
    # Perform front-to-back alpha compositing if multiple layers exist
    # This step assumes 'final_color' and 'final_opacity' already contain previous layers
    # and 'gathered_sub_images' is the current layer to blend in.
    # For a single rank's contribution to a global buffer, this is often handled by the root
    # gathering all and then compositing.
    
    # If compositing is done per-slice on the root:
    # for each z_slice in sorted_z_slices:
    #     z_split = get_sub_image_for_slice(z_slice)
    #     final_color[:, :, :3] += z_split[:, :, :3] * (1.0 - final_opacity)
    #     final_opacity[:, :, 3] += z_split[:, :, 3] * (1.0 - final_opacity[:, :, 3])
    
    # Convert to uint8 and save
    final_rgba = np.zeros((x_dim, y_dim, 4), dtype=np.uint8)
    final_rgba[:, :, :3] = np.clip(final_color * 255, 0, 255).astype(np.uint8)
    final_rgba[:, :, 3] = np.clip(final_opacity * 255, 0, 255).astype(np.uint8)
    
    return final_rgba
```

**Edge Cases and Pitfalls:**
*   **Overhead:** Spatial compositing has huge overhead compared to temporal sequencing [S1, unofficial].
*   **Precision:** Using `float32` for intermediate calculations is standard, but conversion to `uint8` must be clipped to avoid overflow [S2, unofficial].
*   **Boundary Conditions:** Ensure sub-images are placed at correct $(px, py)$ coordinates to avoid gaps or overlaps [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** Dominated by the communication cost of gathering sub-images to the root rank and the compositing step, which is $O(X \cdot Y)$ per depth slice [S1, unofficial].
*   **Memory Complexity:** Requires storing the full final image $(X \cdot Y \cdot 4)$ and all sub-images during the gather phase [S2, unofficial].
*   **Strengths:** Produces a single, coherent final image suitable for display or storage [S1, unofficial].
*   **Limitations:** High overhead due to data collection and potential bottleneck at the root rank [S1, unofficial]. Not suitable for real-time animation without optimization [S1, unofficial].
*   **When to Use:** When a final static image or a small set of frames is required [S1, unofficial].
*   **When Not to Use:** For real-time, high-frame-rate visualization where temporal sequencing is preferred [S1, unofficial].

## Background
Parallel Image Assembly, also known as spatial compositing, is a technique used in parallel volume rendering to combine results from spatially decomposed data [S1, unofficial]. It was described in the context of parallel, multiresolution, and adaptive-resolution techniques for time-varying data visualization [S1, unofficial]. The method involves collecting sub-images from different threads and compositing them in a front-to-back order [S1, unofficial].

## Sources
[S1] Volume visualization of time-varying data using parallel, multiresolution and adaptive-resolution techniques, https://scholars.unh.edu/cgi/viewcontent.cgi?article=1245&context=thesis, fetched 2026-10-27 (unofficial)
[S2] vooranagendra/Parallel-3D-Volume-Rendering-using-MPI, https://github.com/vooranagendra/Parallel-3D-Volume-Rendering-using-MPI, fetched 2026-10-27 (unofficial)
[S3] jhadivyanshu24/Parallel-3D-Volume-Rendering-with-MPI-CS677, https://github.com/jhadivyanshu24/Parallel-3D-Volume-Rendering-with-MPI-CS677, fetched 2026-10-27 (unofficial)
