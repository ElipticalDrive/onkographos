# Parallel Image Assembly

Also called: Final Image Assembly & Alpha Blending, Image Stitching, Spatial Compositing, Z-Order Compositing

## Overview
Parallel Image Assembly is the final step in spatial decomposition volume rendering where sub-images rendered by independent threads or processes are combined into a single final image [S1, unofficial]. It solves the problem of merging disjoint spatial data portions into a coherent visual output by placing sub-images at their correct spatial coordinates and blending them along the z-axis using front-to-back alpha compositing [S2, unofficial]. Unlike temporal sequencing, which has minimal overhead, spatial compositing incurs significant overhead due to the data collection and blending requirements [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of composited sub-images (RGBA) and their corresponding opacities from all participating ranks/threads [S3, unofficial]. Each sub-image corresponds to a specific spatial tile $(px, py)$ and potentially a depth slice $pz$ [S3, unofficial].
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
3.  **Place Sub-images:** Iterate through all gathered ranks [S3, unofficial].
4.  **Depth Handling:** For each rank, determine its spatial coordinates $(px, py, pz)$ [S3, unofficial]. If the rank corresponds to the middle depth slice ($pz == PZ // 2$), place its composited color image into the `final_image` grid at the appropriate $(px, py)$ location [S3, unofficial]. Add the opacity from this slice to the `total_opacity` map for the corresponding $(px, py)$ location [S3, unofficial].
5.  **Compositing:** For each z-slice (or depth layer), blend the sub-image into the `final_image` using front-to-back alpha compositing [S2, unofficial]. The blending follows the standard volume rendering equation:
    $$ C_{final} = C_{prev} + C_{z} \cdot (1 - \alpha_{prev}) $$
    $$ \alpha_{final} = \alpha_{prev} + \alpha_{z} \cdot (1 - \alpha_{prev}) $$
    Where $C$ is color (RGB) and $\alpha$ is opacity [S2, unofficial].
6.  **Finalization:** Convert the `final_image` from `float32` to `uint8` RGBA and save as a PNG file [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def parallel_image_assembly(gathered_sub_images, gathered_opacities, px, py, pz, sub_width, sub_height, x_dim, y_dim, PZ):
    """
    Assembles sub-images into a final image using spatial compositing.
    
    Args:
        gathered_sub_images: List of arrays from all ranks, each of shape (sub_width, sub_height, 4)
        gathered_opacities: List of arrays from all ranks, each of shape (sub_width, sub_height, 1)
        px, py, pz: Spatial coordinates of the current rank's sub-image
        sub_width, sub_height: Dimensions of each sub-image
        x_dim, y_dim: Total dimensions of the final image
        PZ: Total number of depth slices
        
    Returns:
        final_rgba: List of lists of lists representing uint8 RGBA image of shape (x_dim, y_dim, 4)
    """
    # Initialize final image arrays with zeros
    def create_zeros(shape):
        return [[[0.0 for _ in range(shape[2])] for _ in range(shape[1])] for _ in range(shape[0])]
    
    final_color = create_zeros((x_dim, y_dim, 3))
    final_opacity = create_zeros((x_dim, y_dim, 1))
    
    # Place sub-images into the final image grid at spatial coordinates
    start_x = px * sub_width
    start_y = py * sub_height
    for dy in range(sub_height):
        for dx in range(sub_width):
            final_color[start_x + dx][start_y + dy] = gathered_sub_images[dy][dx][:3]
            final_opacity[start_x + dx][start_y + dy] = [gathered_opacities[dy][dx][0]]
            
    # Depth handling: if middle slice, accumulate opacity to total_opacity map
    if pz == PZ // 2:
        for dy in range(sub_height):
            for dx in range(sub_width):
                final_opacity[start_x + dx][start_y + dy][0] += gathered_opacities[dy][dx][0]
                
    # Front-to-back alpha compositing across z-slices
    # Sources do not specify the exact iteration order for multiple slices beyond front-to-back [G]
    for z_idx in range(len(gathered_sub_images)):
        z_img = gathered_sub_images[z_idx]
        z_op = gathered_opacities[z_idx]
        for dy in range(y_dim):
            for dx in range(x_dim):
                c_prev = final_color[dx][dy]
                a_prev = final_opacity[dx][dy][0]
                c_z = z_img[dy][dx][:3]
                a_z = z_op[dy][dx][0]
                
                # Apply compositing formulas
                final_color[dx][dy] = [c_prev[j] + c_z[j] * (1.0 - a_prev) for j in range(3)]
                final_opacity[dx][dy][0] = a_prev + a_z * (1.0 - a_prev)
                
    # Convert to uint8 RGBA with clipping to avoid overflow
    final_rgba = create_zeros((x_dim, y_dim, 4))
    for dy in range(y_dim):
        for dx in range(x_dim):
            for j in range(3):
                val = final_color[dx][dy][j] * 255.0
                final_rgba[dx][dy][j] = max(0, min(255, int(val)))
            val = final_opacity[dx][dy][0] * 255.0
            final_rgba[dx][dy][3] = max(0, min(255, int(val)))
            
    return final_rgba
```

**Edge Cases and Pitfalls:**
*   **Overhead:** Spatial compositing has huge overhead compared to temporal sequencing [S1, unofficial].
*   **Precision:** Using `float32` for intermediate calculations is standard, but conversion to `uint8` must be clipped to avoid overflow [S2, unofficial].
*   **Boundary Conditions:** Ensure sub-images are placed at correct $(px, py)$ coordinates to avoid gaps or overlaps [S3, unofficial].

## Subtopics

### Image Space Scaling via Tiling and Reassembly
A technique for rendering high-resolution viewports that intersect multiple display tiles (projectors) by tiling the image and reassembling it, requiring dynamic progress properties for frame reassembly [S4]. It renders a single image stream (e.g., 1600x1200 RGBA) at the target resolution and distributes the image tiles to the appropriate projectors based on viewport intersection [S4]. If the viewport intersects multiple logical pipelines, a new routability proof is required to account for image sources contributing to more than one pipeline, and a dynamic progress property must be implemented for frame reassembly [S4].

## Operating characteristics
*   **Time Complexity:** Dominated by the communication cost of gathering sub-images to the root rank and the compositing step, which is $O(X \cdot Y)$ per depth slice [S1, unofficial].
*   **Memory Complexity:** Requires storing the full final image $(X \cdot Y \cdot 4)$ and all sub-images during the gather phase [S2, unofficial].
*   **Strengths:** Produces a single, coherent final image suitable for display or storage [S1, unofficial].
*   **Limitations:** High overhead due to data collection and potential bottleneck at the root rank [S1, unofficial]. Not suitable for real-time animation without optimization [S1, unofficial].

## Background
Parallel Image Assembly, also known as spatial compositing, is a technique used in parallel volume rendering to combine results from spatially decomposed data [S1, unofficial]. It involves collecting sub-images from different threads and compositing them in a front-to-back order [S1, unofficial]. The method is applied when each thread has completely processed its own data portion and the main thread must merge the separate ray segments into one final image [S1, unofficial].

## Sources
[S1] Volume visualization of time-varying data using parallel, multiresolution and adaptive-resolution techniques, https://scholars.unh.edu/cgi/viewcontent.cgi?article=1245&context=thesis, fetched 2026-10-27 (unofficial)
[S2] vooranagendra/Parallel-3D-Volume-Rendering-using-MPI, https://github.com/vooranagendra/Parallel-3D-Volume-Rendering-using-MPI, fetched 2026-10-27 (unofficial)
[S3] jhadivyanshu24/Parallel-3D-Volume-Rendering-with-MPI-CS677, https://github.com/jhadivyanshu24/Parallel-3D-Volume-Rendering-with-MPI-CS677, fetched 2026-10-27 (unofficial)
[S4] ``Scalable interactive volume rendering using off-the-shelf components,'', http://www.cacr.caltech.edu/~slombey/viz/pvr/pvr-paper.pdf, fetched 2026-10-28
