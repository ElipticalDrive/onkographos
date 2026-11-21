# Octant-based Distributed Volume Rendering

Also called: Distributed Ray Casting, Distributed Volume Rendering, Octant-based Distributed VR

## Overview
Octant-based Distributed Volume Rendering is a parallel image-order rendering technique that partitions a 3D volume into eight spatial octants, assigning each to an independent worker process (server) for concurrent ray casting, while a central client process composites the resulting subvolume images to produce the final frame [S1, unofficial]. It differs from single-processor volume rendering by eliminating redundant computation across the entire volume and from other distributed methods by using a fixed, non-recursive octree subdivision (static partitioning) rather than adaptive refinement, relying on a specific opacity correction in the compositing stage to handle the boundaries between independent renderers [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** A 3D scalar field volume $V$ with dimensions $D_x \times D_y \times D_z$, a camera view (position, orientation, field of view), and a target image plane resolution $W \times H$.
*   **Output:** A final composited image buffer containing color and opacity for each pixel.

**Data Structures**
1.  **Volume Partition:** The volume $V$ is divided into 8 equal-sized octants. To handle gradient calculations and interpolation at boundaries, each octant is padded with a 1-voxel wide border from its three adjacent octants, resulting in "padded octants" [S1, unofficial].
2.  **RLE File:** Each server generates a Run-Length Encoded (RLE) file. This file stores the final color ($C_{out}$) and opacity ($\alpha_{out}$) of the ray cast through that specific octant for every pixel in the image plane [S1, unofficial].
3.  **Client Buffer (`alphabuffer`):** A subclass of the standard `BUFFER` class. Unlike server buffers which shade pixels via ray casting, this buffer shades pixels by compositing the results received from the servers [S1, unofficial].
4.  **Communication:** Uses the AFTTCOM library (socket-based message passing) to send RLE file names from servers to the client, and `UNIX rsh` to launch remote server processes [S1, unofficial].

**Numbered Steps**
1.  **Initialization:** The client process (administrator) spawns eight server processes. Each server is assigned one of the eight octants of the volume.
2.  **Local Rendering:** Each server independently performs volume rendering on its assigned octant.
    *   For each pixel in the image plane, a ray is cast through the padded octant.
    *   **Boundary Handling:** If a sample point lies within the octant's bordering voxels (the pad), the pad is used to provide the eight surrounding voxels for trilinear interpolation. Otherwise, the pad is ignored [S1, unofficial].
    *   **Gradient Approximation:** Central differences are used to approximate the gradient vector for normal calculation at each voxel. The pad ensures sufficient neighbors for this calculation at shared faces [S1, unofficial].
    *   **Ray Exit:** When a ray exits an octant on a face bordering another octant, no background color is composited into the ray because the resulting opacity is zero. A zero opacity from a forward octant prevents color contribution from the octant behind it during composition [S1, unofficial].
    *   The server writes the resulting color and opacity values for all pixels into an RLE file.
3.  **Communication:** Each server sends the filename of its RLE file to the client via AFTTCOM message passing [S1, unofficial].
4.  **Buffer Loading:** The client reads the RLE files into memory buffers.
5.  **Sorting:** The client sorts the eight buffers in front-to-back image order based on the depth of each server's octant in image space. This is achieved using a bubble sort algorithm on the buffers [S1, unofficial].
6.  **Compositing:** The client composites the sorted buffers to produce the final image.
    *   The client loops through the buffers in depth order.
    *   For each pixel, it applies the distributed transparency formulas.
    *   **Optimization:** If a front pixel and a back pixel are both background, or if either is background, no composition is performed; one color is used directly. If a ray begins with no color and exists on a shared face, it may terminate with no color, skipping composition [S1, unofficial].

**Formulas**
The compositing step uses modified transparency formulas to account for the fact that each server renders a subvolume independently. The opacity term in the buffer represents the *remaining* opacity behind the octant, not the true opacity of the octant itself. Therefore, for all buffers except the first (closest), the opacity term must be subtracted from 1 to get the true opacity [S1, unofficial].

Let $C_{in}$ and $\alpha_{in}$ be the accumulated color and opacity from the closest octant(s) processed so far.
Let $C_s$ and $\alpha_s$ be the color and opacity of the current octant's buffer being composited.

The distributed transparency formulas are:

$$ C_{out} = C_{in} + (\alpha_{in})(1 - \alpha_s)(C_s) $$

$$ \alpha_{out} = \alpha_{in} - \alpha_{in}(1 - \alpha_s) $$

Where:
*   $C_{in}, \alpha_{in}$ are initialized to the pixel values of the closest octant's buffer.
*   $C_s, \alpha_s$ are the pixel values of the next closest octant's buffer.
*   The term $(1 - \alpha_s)$ represents the true opacity of the octant [S1, unofficial].

**Parameters**
*   **Octant Division:** The volume is split into 8 octants, each handled by one server processor (1/8th of the volume per server) [S1, unofficial].
*   **View Naming:** Numbers represent the Z-height of the eyepoint in object space or X/Y position along top/bottom. Letters: RS (Right Side), 3F (3D Front), LO (Left Side), SP (Single Processor), Distr (Distributed) [S1, unofficial].

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def octant_distributed_volume_rendering(volume, camera, image_width, image_height):
    # 1. Partition Volume
    # Divide volume into 8 octants. Pad each octant with 1-voxel border from adjacent octants.
    octants = partition_volume_into_octants(volume)
    
    # 2. Spawn Servers and Render
    # Each server renders its octant and saves RLE file
    rle_files = []
    for i, octant in enumerate(octants):
        # Server process logic
        server_buffer = render_octant(octant, camera, image_width, image_height)
        rle_filename = f"server_{i}.rle"
        save_rle(server_buffer, rle_filename)
        rle_files.append((rle_filename, octant))
        
    # 3. Client: Load Buffers
    client_buffers = []
    for rle_filename, octant in rle_files:
        buffer = load_rle_into_buffer(rle_filename)
        client_buffers.append((buffer, octant))
        
    # 4. Sort Buffers Front-to-Back
    # Sort based on the depth of the octant in image space
    # Using bubble sort as per source
    for i in range(len(client_buffers)):
        for j in range(0, len(client_buffers) - i - 1):
            if client_buffers[j][1].depth_in_image > client_buffers[j+1][1].depth_in_image:
                client_buffers[j], client_buffers[j+1] = client_buffers[j+1], client_buffers[j]
                
    # 5. Composite
    # Initialize final buffer
    final_buffer = create_empty_buffer(image_width, image_height)
    
    # Process closest octant first (no composition needed, just copy)
    closest_buffer, _ = client_buffers[0]
    for y in range(image_height):
        for x in range(image_width):
            final_buffer.set_pixel(x, y, closest_buffer.get_pixel(x, y))
            
    # Composite remaining octants
    for k in range(1, len(client_buffers)):
        current_buffer, _ = client_buffers[k]
        for y in range(image_height):
            for x in range(image_width):
                # Get current accumulated pixel
                c_in, a_in = final_buffer.get_pixel(x, y)
                
                # Get current octant pixel
                c_s, a_s = current_buffer.get_pixel(x, y)
                
                # Optimization: Background check
                # If either pixel is background, no composition needed
                if is_background(c_in, a_in) or is_background(c_s, a_s):
                    # Use the non-background color, or background if both are
                    if is_background(c_s, a_s):
                        pass # Keep c_in, a_in
                    else:
                        final_buffer.set_pixel(x, y, c_s, a_s)
                    continue
                
                # Optimization: Zero color termination
                # If ray begins with no color and exists on shared face, terminate
                if a_in == 0:
                    continue

                # Apply Distributed Transparency Formulas
                # True opacity of the octant is (1 - a_s)
                true_opacity = 1.0 - a_s
                
                c_out = c_in + (a_in * true_opacity * c_s)
                a_out = a_in - (a_in * true_opacity)
                
                final_buffer.set_pixel(x, y, c_out, a_out)
                
    return final_buffer

def render_octant(octant, camera, width, height):
    buffer = create_empty_buffer(width, height)
    for y in range(height):
        for x in range(width):
            ray = camera.cast_ray(x, y)
            
            # Intersect ray with octant boundaries
            t_min, t_max = intersect_octant(ray, octant)
            
            if t_max < t_min:
                buffer.set_pixel(x, y, BACKGROUND_COLOR, 0.0)
                continue
                
            # Initialize ray state
            c_acc = 0.0
            a_acc = 0.0
            
            # Sample along ray within octant
            # Note: Levoy's equally spaced sample requirement is broken at boundaries
            # because exit point != last sample point
            t = t_min
            while t < t_max:
                point = ray.origin + t * ray.direction
                
                # Check if point is in pad region
                if is_in_pad(point, octant):
                    # Use pad for trilinear interpolation
                    voxel_values = get_trilinear_values(point, octant.pad)
                else:
                    # Use octant data
                    voxel_values = get_trilinear_values(point, octant.data)
                    
                color, opacity = classify_and_shade(voxel_values)
                
                # Standard front-to-back compositing within the octant
                # Note: This is local to the octant, so standard formulas apply
                # C_out = C_in + alpha_in * (1 - alpha_s) * C_s
                # alpha_out = alpha_in - alpha_in * (1 - alpha_s)
                # However, source implies servers generate final Cout/Cout for the octant
                # So we accumulate within the octant
                
                c_acc = c_acc + (a_acc * (1 - opacity) * color)
                a_acc = a_acc - (a_acc * (1 - opacity))
                
                t += step_size
                
            # If ray exits octant, no background color is composited if opacity is zero
            # The server returns the accumulated color and opacity for this octant
            buffer.set_pixel(x, y, c_acc, a_acc)
            
    return buffer
```

**Edge Cases and Pitfalls**
*   **Sampling Anomalies at Borders:** In uniform artificial volumes, incorrect sampling at octant borders produces visible pixel color anomalies. This occurs because the ray's exit point in one octant (on the volume face) does not equal the last sampled point (within the containing voxel). The adjacent octant's entry point is the first sample point. This breaks Levoy's requirement for equally spaced samples along the ray [S1, unofficial]. In non-uniform volumes (e.g., CT chest), these anomalies are masked by tissue differences [S1, unofficial].
*   **Gradient Approximation:** Central differencing for normal approximation was investigated as a cause for shading inconsistencies at shared voxels. This was disproved for uniform box volumes because shared voxels had equal colors assigned during the shade/classify step [S1, unofficial].
*   **Ray Exit at Shared Faces:** When a ray exits an octant on a face bordering another octant, no background color is composited into the ray because the resulting opacity is zero. A zero opacity from a forward octant allows no color contribution from the octant behind it [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Preprocessing time scales inversely with the number of octants ($O(V/P)$ where $V$ is volume size, $P$ is processor count). Ray casting time depends heavily on view direction and volume dimensions (e.g., views along the greatest height dimension take longer) [S1, unofficial]. Total execution time includes communication overhead and client-side composition [S1, unofficial].
*   **Memory Complexity:** Each server stores its octant plus padding. The client stores 8 full-resolution image buffers (RLE files loaded into memory) during compositing [S1, unofficial].
*   **Measured Performance:**
    *   **CT Chest (240x164x175):** Single Processor (SP) time 10:05, Distributed (Distr) time 4:14. 58% savings in total execution time. Preprocessing/ray casting savings 70-85% [S1, unofficial].
    *   **Artificial Box (200x200x210):** SP time 6:32, Distr time 2:45. 58% savings in total execution time [S1, unofficial].
*   **Strengths:** Significant reduction in processing time (58% savings in tested scenarios). Enables cut-away viewing and intermediate subvolume viewing by accessing individual server RLE files [S1, unofficial].
*   **Limitations:** Visible sampling anomalies in uniform volumes at octant boundaries. Fixed subdivision (no adaptive refinement) may lead to load imbalance if volume density is non-uniform. Communication overhead for RLE files and compositing logic [S1, unofficial].
*   **When to Use:** When rendering large volumes on a distributed system with 8 processors, and visual fidelity in uniform regions is not critical or can be masked by data complexity.
*   **When Not to Use:** When uniform volumes require high visual fidelity at boundaries, or when adaptive refinement is needed for variable density volumes [S1, unofficial].

## Background
*   **Origin:** Developed by an unnamed author in a thesis/report (DTIC ADA248167) for distributed volume rendering on an eight-server system [S1, unofficial].
*   **History:** The method was implemented using the AFTTCOM message passing library and UNIX rsh. It was tested on CT chest, CT dog heart, and artificial box volumes to demonstrate performance savings and cut-away viewing capabilities [S1, unofficial].
*   **Usage:** Used to demonstrate the feasibility of distributed volume rendering, specifically highlighting the trade-off between processing time savings and visual artifacts at octant boundaries in uniform data [S1, unofficial].

## Sources
[S1] DTIC ADA248167: Distributed Ray Casting for High-Speed Volume Rendering, https://archive.org/details/DTIC_ADA248167, fetched 2026-10-26 (unofficial)
