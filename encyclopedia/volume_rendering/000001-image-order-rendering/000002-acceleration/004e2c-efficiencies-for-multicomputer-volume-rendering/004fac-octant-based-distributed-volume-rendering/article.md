# Octant-based Distributed Volume Rendering

Also called: Distributed Ray Casting, Distributed Volume Rendering, Octant-based Distributed VR

## Overview
Octant-based Distributed Volume Rendering is a parallel image-order rendering technique that partitions a 3D volume into eight spatial octants, assigns each to an independent worker process (server) for concurrent ray casting, and composites the resulting subvolume images at a central client process to produce the final frame [S1, unofficial]. It differs from single-processor volume rendering by eliminating redundant computation across the entire volume and from other distributed methods by using a fixed, non-recursive octree subdivision rather than adaptive refinement, relying on a specific opacity correction in the compositing stage to handle boundaries between independent renderers [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** A 3D scalar field volume $V$ with dimensions $D_x \times D_y \times D_z$, a camera view (position, orientation, field of view), and a target image plane resolution $W \times H$ [S1, unofficial].
*   **Output:** A final composited image buffer containing color and opacity for each pixel [S1, unofficial].

**Data Structures**
1.  **Volume Partition:** The volume $V$ is divided into 8 equal-sized octants [S1, unofficial]. To handle gradient calculations and interpolation at boundaries, each octant is padded with a 1-voxel wide border from its three adjacent octants, resulting in "padded octants" [S1, unofficial].
2.  **RLE File:** Each server generates a Run-Length Encoded (RLE) file [S1, unofficial]. This file stores the final color ($C_{out}$) and opacity ($\alpha_{out}$) of the ray cast through that specific octant for every pixel in the image plane [S1, unofficial].
3.  **Client Buffer (`alphabuffer`):** A subclass of the standard `BUFFER` class [S1, unofficial]. Unlike server buffers which shade pixels via ray casting, this buffer shades pixels by compositing the results received from the servers [S1, unofficial].
4.  **Communication:** Uses the AFTTCOM library (socket-based message passing) to send RLE file names from servers to the client, and `UNIX rsh` to launch remote server processes [S1, unofficial].

**Numbered Steps**
1.  **Initialization:** The client process (administrator) spawns eight server processes [S1, unofficial]. Each server is assigned one of the eight octants of the volume [S1, unofficial].
2.  **Local Rendering:** Each server independently performs volume rendering on its assigned octant [S1, unofficial]. For each pixel in the image plane, a ray is cast through the padded octant [S1, unofficial]. If a sample point lies within the octant's bordering voxels (the pad), the pad is used to provide the eight surrounding voxels for trilinear interpolation; otherwise, the pad is ignored [S1, unofficial]. Central differences are used to approximate the gradient vector for normal calculation at each voxel [S1, unofficial]. When a ray exits an octant on a face bordering another octant, no background color is composited into the ray because the resulting opacity is zero [S1, unofficial]. The server writes the resulting color and opacity values for all pixels into an RLE file [S1, unofficial].
3.  **Communication:** Each server sends the filename of its RLE file to the client via AFTTCOM message passing [S1, unofficial].
4.  **Buffer Loading:** The client reads the RLE files into memory buffers [S1, unofficial].
5.  **Sorting:** The client sorts the eight buffers in front-to-back image order based on the depth of each server's octant in image space [S1, unofficial]. This is achieved using a bubble sort algorithm on the buffers [S1, unofficial].
6.  **Compositing:** The client composites the sorted buffers to produce the final image [S1, unofficial]. The client loops through the buffers in depth order [S1, unofficial]. For each pixel, it applies the distributed transparency formulas [S1, unofficial]. If a front pixel and a back pixel are both background, or if either is background, no composition is performed and one color is used directly [S1, unofficial]. If a ray begins with no color and exists on a shared face, it may terminate with no color, skipping composition [S1, unofficial].

**Formulas**
The compositing step uses modified transparency formulas to account for the fact that each server renders a subvolume independently [S1, unofficial]. The opacity term in the buffer represents the *remaining* opacity behind the octant, not the true opacity of the octant itself [S1, unofficial]. Therefore, for all buffers except the first (closest), the opacity term must be subtracted from 1 to get the true opacity [S1, unofficial].

Let $C_{in}$ and $\alpha_{in}$ be the accumulated color and opacity from the closest octant(s) processed so far [S1, unofficial].
Let $C_s$ and $\alpha_s$ be the color and opacity of the current octant's buffer being composited [S1, unofficial].

The distributed transparency formulas are:

$$ C_{out} = C_{in} + (\alpha_{in})(1 - \alpha_s)(C_s) $$

$$ \alpha_{out} = \alpha_{in} - \alpha_{in}(1 - \alpha_s) $$

Where:
*   $C_{in}, \alpha_{in}$ are initialized to the pixel values of the closest octant's buffer [S1, unofficial].
*   $C_s, \alpha_s$ are the pixel values of the next closest octant's buffer [S1, unofficial].
*   The term $(1 - \alpha_s)$ represents the true opacity of the octant [S1, unofficial].

**Parameters**
*   **Octant Division:** The volume is split into 8 octants, each handled by one server processor (1/8th of the volume per server) [S1, unofficial].
*   **View Naming:** Numbers represent the Z-height of the eyepoint in object space or X/Y position along top/bottom [S1, unofficial]. Letters: RS (Right Side), 3F (3D Front), LO (Left Side), SP (Single Processor), Distr (Distributed) [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def octant_distributed_volume_rendering(volume, camera, image_width, image_height):
    # 1. Partition volume into 8 octants with 1-voxel padding from adjacent octants
    octants = []
    for i in range(2):
        for j in range(2):
            for k in range(2):
                sub_vol = extract_sub_volume(volume, i, j, k)
                padded_octant = pad_octant(sub_vol, volume)
                octants.append(padded_octant)

    # 2. Spawn servers and render each octant
    rle_data = []
    for idx, octant in enumerate(octants):
        buffer = render_octant_local(octant, camera, image_width, image_height)
        rle_data.append((buffer, octant))

    # 3. Client loads RLE data into memory buffers
    client_buffers = rle_data

    # 4. Sort buffers front-to-back by octant depth in image space (bubble sort)
    for i in range(len(client_buffers)):
        for j in range(0, len(client_buffers) - i - 1):
            if client_buffers[j][1].depth > client_buffers[j+1][1].depth:
                client_buffers[j], client_buffers[j+1] = client_buffers[j+1], client_buffers[j]

    # 5. Composite buffers
    final_buffer = create_zero_buffer(image_width, image_height)
    # Initialize with closest octant
    closest_buf, _ = client_buffers[0]
    for y in range(image_height):
        for x in range(image_width):
            final_buffer[y, x] = closest_buf[y, x]

    # Composite remaining octants
    for k in range(1, len(client_buffers)):
        current_buf, _ = client_buffers[k]
        for y in range(image_height):
            for x in range(image_width):
                c_in, a_in = final_buffer[y, x]
                c_s, a_s = current_buf[y, x]

                # Optimization: background check
                if is_background(c_in, a_in) or is_background(c_s, a_s):
                    if not is_background(c_s, a_s):
                        final_buffer[y, x] = (c_s, a_s)
                    continue

                # Optimization: zero color termination
                if a_in == 0.0:
                    continue

                # Distributed transparency formulas
                true_opacity = 1.0 - a_s
                c_out = c_in + (a_in * true_opacity * c_s)
                a_out = a_in - (a_in * true_opacity)
                final_buffer[y, x] = (c_out, a_out)

    return final_buffer

def render_octant_local(octant, camera, width, height):
    buffer = create_zero_buffer(width, height)
    for y in range(height):
        for x in range(width):
            ray = camera.cast_ray(x, y)
            t_min, t_max = ray_intersect_box(ray, octant.bounds)
            if t_max < t_min:
                buffer[y, x] = (0.0, 0.0)
                continue

            c_acc, a_acc = 0.0, 0.0
            t = t_min
            step = compute_step_size(ray, octant.resolution)
            while t < t_max:
                point = ray.origin + t * ray.direction
                # Trilinear interpolation using pad if near boundary
                color, opacity = trilinear_sample(point, octant.data)
                # Standard local compositing
                c_acc = c_acc + (a_acc * (1.0 - opacity) * color)
                a_acc = a_acc - (a_acc * (1.0 - opacity))
                t += step

            buffer[y, x] = (c_acc, a_acc)
    return buffer

# Helper definitions (standard graphics operations)
def extract_sub_volume(vol, ix, iy, iz):
    dx, dy, dz = vol.shape
    sx, ex = ix * dx // 2, (ix + 1) * dx // 2
    sy, ey = iy * dy // 2, (iy + 1) * dy // 2
    sz, ez = iz * dz // 2, (iz + 1) * dz // 2
    return vol[sx:ex, sy:ey, sz:ez]

def pad_octant(sub, full_vol):
    # Pad with 1 voxel from adjacent octants in full_vol
    pad = np.pad(sub, 1, mode='edge')
    return pad

def trilinear_sample(point, grid):
    # Standard trilinear interpolation [G]
    coords = point_to_grid_coords(point, grid)
    return interpolate_trilinear(coords, grid)

def is_background(color, opacity):
    return opacity == 0.0

def create_zero_buffer(w, h):
    return np.zeros((h, w, 2))

def compute_step_size(ray, resolution):
    return 1.0 / max(resolution)
```

**Edge Cases and Pitfalls**
*   **Sampling Anomalies at Borders:** In uniform artificial volumes, incorrect sampling at octant borders produces visible pixel color anomalies [S1, unofficial]. This occurs because the ray's exit point in one octant (on the volume face) does not equal the last sampled point (within the containing voxel) [S1, unofficial]. The adjacent octant's entry point is the first sample point [S1, unofficial]. This breaks Levoy's requirement for equally spaced samples along the ray [S1, unofficial]. In non-uniform volumes (e.g., CT chest), these anomalies are masked by tissue differences [S1, unofficial].
*   **Gradient Approximation:** Central differencing for normal approximation was investigated as a cause for shading inconsistencies at shared voxels [S1, unofficial]. This was disproved for uniform box volumes because shared voxels had equal colors assigned during the shade/classify step [S1, unofficial].
*   **Ray Exit at Shared Faces:** When a ray exits an octant on a face bordering another octant, no background color is composited into the ray because the resulting opacity is zero [S1, unofficial]. A zero opacity from a forward octant allows no color contribution from the octant behind it [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Preprocessing time scales inversely with the number of octants ($O(V/P)$ where $V$ is volume size, $P$ is processor count) [S1, unofficial]. Ray casting time depends heavily on view direction and volume dimensions (e.g., views along the greatest height dimension take longer) [S1, unofficial]. Total execution time includes communication overhead and client-side composition [S1, unofficial].
*   **Memory Complexity:** Each server stores its octant plus padding [S1, unofficial]. The client stores 8 full-resolution image buffers (RLE files loaded into memory) during compositing [S1, unofficial].
*   **Measured Performance:**
    *   **CT Chest (240x164x175):** Single Processor (SP) time 10:05, Distributed (Distr) time 4:14 [S1, unofficial]. 58% savings in total execution time [S1, unofficial].
    *   **Artificial Box (200x200x210):** SP time 6:32, Distr time 2:45 [S1, unofficial]. 58% savings in total execution time [S1, unofficial].
*   **Strengths:** Significant reduction in processing time [S1, unofficial]. Enables cut-away viewing and intermediate subvolume viewing by accessing individual server RLE files [S1, unofficial].
*   **Limitations:** Visible sampling anomalies in uniform volumes at octant boundaries [S1, unofficial]. Fixed subdivision (no adaptive refinement) may lead to load imbalance if volume density is non-uniform [S1, unofficial]. Communication overhead for RLE files and compositing logic [S1, unofficial].
*   **When to Use:** When rendering large volumes on a distributed system with 8 processors, and visual fidelity in uniform regions is not critical or can be masked by data complexity [S1, unofficial].
*   **When Not to Use:** When uniform volumes require high visual fidelity at boundaries, or when adaptive refinement is needed for variable density volumes [S1, unofficial].

## Background
*   **Origin:** Developed for distributed volume rendering on an eight-server system [S1, unofficial].
*   **History:** The method was implemented using the AFTTCOM message passing library and UNIX rsh [S1, unofficial]. It was tested on CT chest, CT dog heart, and artificial box volumes to demonstrate performance savings and cut-away viewing capabilities [S1, unofficial].
*   **Usage:** Used to demonstrate the feasibility of distributed volume rendering, specifically highlighting the trade-off between processing time savings and visual artifacts at octant boundaries in uniform data [S1, unofficial].

## Sources
[S1] DTIC ADA248167: Distributed Ray Casting for High-Speed Volume Rendering, https://archive.org/details/DTIC_ADA248167, fetched 2026-10-26 (unofficial)
