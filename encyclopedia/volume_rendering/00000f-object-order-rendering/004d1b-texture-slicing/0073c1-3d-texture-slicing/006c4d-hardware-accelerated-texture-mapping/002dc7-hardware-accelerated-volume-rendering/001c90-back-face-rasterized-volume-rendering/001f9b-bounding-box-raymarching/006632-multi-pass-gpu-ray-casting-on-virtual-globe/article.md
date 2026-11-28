# Multi-pass GPU Ray Casting on Virtual Globe

Also called: GCS to CCS conversion, GPU-based Direct Volume Ray-Casting on Virtual Globe, Multi-pass GPU ray-casting for EME, Spherical Volume Texture on Virtual Globes, Spherical coordinate DVR, Virtual globe ray casting

## Overview
This method renders 3D volumetric data stored in spherical coordinates onto a virtual globe by transforming bounding box positions to Cartesian space, generating rays in Cartesian coordinates, and sampling the spherical texture front-to-back [S2]. It differs from standard GPU-based ray-casting because the voxel positions use a spherical coordinate system which is irregular, requiring explicit Cartesian coordinate transformation for ray generation and texture sampling [S2].

## How it works
**Inputs and Outputs:**
*   **Input:** 3D volumetric data (e.g., electromagnetic power density) stored in a 3D floating-point texture using spherical coordinates (latitude, longitude, altitude layers) [S2].
*   **Input:** A volume bounding box mesh representing the extent of the volumetric data [S2].
*   **Input:** User-defined transfer function parameters (via color picker) [S2].
*   **Output:** A 2D composite image of the volumetric data blended onto the scene [S2].

**Data Structures:**
1.  **3D Floating-Point Texture:** Stores the volumetric data. The voxels are organized in a spherical coordinate system. The 3D space is divided into $n$ layers along the altitude direction. Each layer is divided into $m \times k$ grids along the longitude and latitude directions [S2].
2.  **Back Face Position Texture:** A 2D RGB texture used to store the termination points of rays entering the volume. This is rendered using a Frame Buffer Object (FBO) [S2].
3.  **Bounding Box Mesh:** A proxy mesh representing the volume bounds. Its vertex colors are assigned the 3D texture coordinates corresponding to the voxel positions [S2].

**Algorithm Steps:**
1.  **Pre-computation of Volumetric Data:** The volumetric data is computed on the GPU using fragment shaders. The power density $P$ at a point is calculated based on radio device parameters. For a single radio, the radiated power density is:
    $$ P = P_t - L - L_s $$
    where $P_t$ is the radio's power, $L$ is the total propagation loss, and $L_s$ represents other losses (system loss, operation loss) [S2].
    The total propagation loss $L$ is the sum of free space propagation loss $L_{fs}$ and terrain/obstruction loss $A_{ref}$:
    $$ L = L_{fs} + A_{ref} $$
    The free space propagation loss is:
    $$ L_{fs} = 32.45 + 20 \lg f + 20 \lg d $$
    where $f$ is the frequency of the radio and $d$ is the distance [S2].
    If there are $N$ radio devices, the total power density is the summation:
    $$ P = \sum_{i=1}^{N} P_i $$
    where $P_i$ is the power density of the $i$-th radio [S2]. This data is stored in the 3D texture [S2].
2.  **First Pass: Render Back Faces:** Render the back faces of the volume bounding box to the Back Face Position Texture (2D RGB texture) using an FBO [S2]. The vertex shader assigns the 3D texture coordinates (spherical coordinates mapped to the bounding box) to the vertex colors of the bounding box [S2]. The fragment shader outputs these colors, effectively storing the entry points of rays into the volume in Cartesian space [S2].
3.  **Second Pass: Ray Casting and Compositing:** Render the front faces of the volume bounding box [S2]. In the vertex shader, transform the vertex position to eye coordinates space and pass it to the fragment shader [S2]. In the fragment shader, fetch the termination point (entry point) for each fragment from the Back Face Position Texture using the transformed vertex position [S2]. Compute the projecting ray direction $\vec{d}$ as the difference between the current fragment's position (exit point) and the fetched termination point [S2]:
    $$ \vec{d} = \text{ExitPoint} - \text{EntryPoint} $$
    Sample the 3D texture of the volumetric data from front to back along the ray direction [S2]. Assign color to each sample using a transfer function based on user interactive input [S2]. Composite each sample color along the ray direction using front-to-back compositing to produce the final fragment color [S2].
4.  **Final Pass: Blending:** Blend the resulting 2D volume image to the scene [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def cartesian_to_spherical(x, y, z, max_radius):
    """Convert Cartesian position to [0,1] spherical texture coordinates."""
    r = math.sqrt(x*x + y*y + z*z)
    if r == 0:
        return (0.0, 0.0, 0.0)
    theta = math.acos(z / r)  # polar angle [0, pi]
    phi = math.atan2(y, x)    # azimuthal angle [-pi, pi]
    u = (phi + math.pi) / (2.0 * math.pi)
    v = theta / math.pi
    w = r / max_radius
    return (u, v, w)

def render_back_faces(bounding_box, back_face_texture):
    """Pass 1: Render back faces to store entry points."""
    # Bind FBO for back_face_texture
    for vertex in bounding_box.vertices:
        # Vertex shader: transform to eye space, pass to fragment
        # Fragment shader: output vertex color = 3D texture coords mapped to Cartesian bounding box
        entry_point = vertex.color  # Cartesian coordinates
        back_face_texture.write(vertex.screen_uv, entry_point)
    # Unbind FBO

def raycast_front_faces(back_face_texture, volume_texture, transfer_function, num_steps, max_radius):
    """Pass 2: Ray casting and front-to-back compositing."""
    for fragment in front_faces:
        entry_point = back_face_texture.read(fragment.screen_uv)  # Cartesian
        exit_point = fragment.vertex_position  # Cartesian (from vertex shader)
        
        ray_dir = (exit_point.x - entry_point.x, 
                   exit_point.y - entry_point.y, 
                   exit_point.z - entry_point.z)
        ray_len = math.sqrt(ray_dir.x**2 + ray_dir.y**2 + ray_dir.z**2)
        if ray_len == 0:
            return (0.0, 0.0, 0.0, 0.0)
            
        step_size = ray_len / num_steps
        ray_dir = (ray_dir.x/ray_len, ray_dir.y/ray_len, ray_dir.z/ray_len)
        
        accumulated_color = (0.0, 0.0, 0.0, 0.0)
        accumulated_alpha = 0.0
        current_pos = entry_point
        
        for i in range(num_steps):
            tex_coords = cartesian_to_spherical(current_pos.x, current_pos.y, current_pos.z, max_radius)
            density = volume_texture.sample(tex_coords)
            color, alpha = transfer_function(density)
            
            if accumulated_alpha >= 1.0:
                break
                
            weight = (1.0 - accumulated_alpha) * alpha
            accumulated_color = (
                accumulated_color[0] + weight * color[0],
                accumulated_color[1] + weight * color[1],
                accumulated_color[2] + weight * color[2],
                accumulated_color[3] + weight * alpha
            )
            accumulated_alpha += weight
            current_pos = (
                current_pos.x + ray_dir.x * step_size,
                current_pos.y + ray_dir.y * step_size,
                current_pos.z + ray_dir.z * step_size
            )
        return accumulated_color

def blend_to_scene(volume_image, scene_buffer):
    """Pass 3: Blend volume image to final scene."""
    for pixel in scene_buffer:
        pixel.color = pixel.color * (1.0 - volume_image.alpha) + volume_image.color
```

**Parameters:**
*   $P_t$: Radio's power [S2].
*   $L_s$: Other losses (system loss, operation loss) [S2].
*   $N$: Number of radio devices within the frequency bands [S2].
*   $P_i$: Power density of the $i$-th radio [S2].
*   $n$: Number of layers along the altitude direction for grid division [S2].
*   $m \times k$: Grid dimensions along longitude and latitude direction per layer [S2].

**Edge Cases and Pitfalls:**
*   **Irregular Coordinates:** Spherical voxel coordinates are irregular, so standard regular volume rendering approaches fail. Explicit coordinate transformation to Cartesian space is required for ray casting [S2].
*   **Precision:** Not given in the sources.
*   **Boundaries:** The method relies on the bounding box to define the ray entry and exit points. Degenerate bounding boxes (zero volume) would result in zero-length rays [G].
*   **Performance:** Rendering performance declines easily with either decrement of viewing distance or increment of vertical scaling when the viewing distance is large. As the viewing point gets closer, rendering performance becomes more sensitive to viewing distance than to the vertical scaling factor [S1].

## Operating characteristics
*   **Time Complexity:** Designed for interactive frame rates using high dense parallel computation on GPU [S2].
*   **Memory Complexity:** Requires storage for the 3D volumetric texture ($m \times k \times n$ samples) and the 2D back-face position texture [S2].
*   **Scaling:** The computation of 3D EME power density is independent at each grid, making it suitable for parallel computing on GPU [S2].
*   **Strengths:** Allows dynamic visualization of 3D EME power density coverage by adjusting radio device parameters; leverages GPU hardware for high-performance rendering [S2].
*   **Limitations:** Requires complex coordinate transformations due to the spherical nature of the data; not directly applicable to regular Cartesian volumes without adaptation [S2].
*   **When to use:** When visualizing 3D electromagnetic environment (EME) data on a virtual globe where data is naturally structured in spherical coordinates [S2].

## Background
*   **Origin:** The method inherits from GPU-based Direct Volume Rendering (DVR) using 3D textures [S2], extended to visualize 3D Electromagnetic Environment (EME) on a virtual globe with spherical coordinates [S2].
*   **History:** Proposed in 2010 by authors publishing in the *I.J. Image, Graphics and Signal Processing* [S2]. It builds upon earlier work on GPU-based DVR and general-purpose GPU computation [S2].
*   **Usage:** Used for visualizing 3D electromagnetic environment (EME) on a virtual globe in the described system [S2].

## Sources
[S1] Different Ray-Casting Algorithm Implementations for Volume Rendering, https://doi.org/10.35940/ijitee.g1011.0597s20, fetched 2026-10-24
[S2] GPU-Based Volume Rendering for 3D Electromagnetic Environment on Virtual Globe, https://doi.org/10.5815/ijigsp.2010.01.07, fetched 2026-10-26
