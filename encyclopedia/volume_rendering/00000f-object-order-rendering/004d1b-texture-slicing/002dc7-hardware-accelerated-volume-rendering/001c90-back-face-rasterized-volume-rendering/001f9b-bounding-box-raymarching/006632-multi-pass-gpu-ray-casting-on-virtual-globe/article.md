# Multi-pass GPU Ray Casting on Virtual Globe

Also called: GPU-based Direct Volume Ray-Casting on Virtual Globe, Multi-pass GPU ray-casting for EME, Spherical coordinate DVR

## Overview
This method renders 3D volumetric data stored in spherical coordinates onto a virtual globe by transforming bounding box positions to Cartesian space, generating rays in Cartesian coordinates, and sampling the spherical texture front-to-back [S1]. It differs from standard GPU-based ray-casting because the voxel positions use a spherical coordinate system which is irregular, requiring explicit Cartesian coordinate transformation for ray generation and texture sampling [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** 3D volumetric data (e.g., electromagnetic power density) stored in a 3D floating-point texture using spherical coordinates (latitude, longitude, altitude layers) [S1].
*   **Input:** A volume bounding box mesh representing the extent of the volumetric data [S1].
*   **Input:** User-defined transfer function parameters (via color picker) [S1].
*   **Output:** A 2D composite image of the volumetric data blended onto the scene [S1].

**Data Structures:**
1.  **3D Floating-Point Texture:** Stores the volumetric data. The voxels are organized in a spherical coordinate system. The 3D space is divided into $n$ layers along the altitude direction. Each layer is divided into $m \times k$ grids along the longitude and latitude directions [S1].
2.  **Back Face Position Texture:** A 2D RGB texture used to store the termination points of rays entering the volume. This is rendered using a Frame Buffer Object (FBO) [S1].
3.  **Bounding Box Mesh:** A proxy mesh representing the volume bounds. Its vertex colors are assigned the 3D texture coordinates corresponding to the voxel positions [S1].

**Algorithm Steps:**

1.  **Pre-computation of Volumetric Data (Optional/Contextual):**
    The volumetric data (e.g., EME power density) is computed on the GPU using fragment shaders. The power density $P$ at a point is calculated based on radio device parameters. For a single radio, the radiated power density is:
    $$ P = P_t - L - L_s $$
    where $P_t$ is the radio's power, $L$ is the total propagation loss, and $L_s$ represents other losses (system loss, operation loss) [S1].
    The total propagation loss $L$ is the sum of free space propagation loss $L_{fs}$ and terrain/obstruction loss $A_{ref}$:
    $$ L = L_{fs} + A_{ref} $$
    The free space propagation loss is:
    $$ L_{fs} = 32.45 + 20 \lg f + 20 \lg d $$
    where $f$ is the frequency of the radio and $d$ is the distance [S1].
    If there are $N$ radio devices, the total power density is the summation:
    $$ P = \sum_{i=1}^{N} P_i $$
    where $P_i$ is the power density of the $i$-th radio [S1]. This data is stored in the 3D texture [S1].

2.  **First Pass: Render Back Faces:**
    *   Render the back faces of the volume bounding box to the **Back Face Position Texture** (2D RGB texture) using an FBO [S1].
    *   The vertex shader assigns the 3D texture coordinates (spherical coordinates mapped to the bounding box) to the vertex colors of the bounding box [S1].
    *   The fragment shader outputs these colors, effectively storing the entry/termination points of rays into the volume in Cartesian space (derived from the transformed vertex positions) [S1].

3.  **Second Pass: Ray Casting and Compositing:**
    *   Render the front faces of the volume bounding box [S1].
    *   **Vertex Shader:** Transform the vertex position to eye coordinates space and pass it to the fragment shader [S1].
    *   **Fragment Shader:**
        1.  Fetch the termination point (entry point) for the current fragment from the **Back Face Position Texture** using the transformed vertex position [S1].
        2.  Compute the projecting ray direction $\vec{d}$ as the difference between the current fragment's position (or vertex color representing the exit point) and the fetched termination point [S1]:
            $$ \vec{d} = \text{CurrentPosition} - \text{TerminationPoint} $$
        3.  Transform the ray direction and positions to align with the spherical coordinate system of the 3D texture if necessary, or sample the texture directly if the texture coordinates are mapped appropriately [S1]. *Note: The source states rays are generated in Cartesian coordinates to sample the spherical texture, implying a coordinate transformation step occurs during sampling or ray definition* [S1].
        4.  Sample the 3D texture of the volumetric data from front to back along the ray direction [S1].
        5.  Assign color to each sample using a transfer function based on user interactive input [S1].
        6.  Composite each sample color along the ray direction using front-to-back compositing to produce the final fragment color [S1].

4.  **Final Pass: Blending:**
    *   Blend the resulting 2D volume image to the scene [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pass 1: Render Back Faces to Position Texture
def render_back_faces():
    bind_fbo(back_face_position_texture)
    draw_mesh(bounding_box_mesh, face_culling=FRONT) # Render only back faces
    # Vertex shader outputs vertex color = 3D texture coordinates (spherical -> cartesian mapping handled by hardware/vertex data)
    unbind_fbo()

# Pass 2: Ray Casting
def render_front_faces():
    bind_fbo(scene_color_buffer)
    draw_mesh(bounding_box_mesh, face_culling=BACK) # Render only front faces
    
    # Vertex Shader
    # Input: Vertex position V, Vertex Color C (3D texture coords)
    # Output: FragPosEye (eye space position), FragColor (exit point in cartesian)
    
    # Fragment Shader
    # Input: FragPosEye, FragColor (exit point)
    # Uniform: back_face_position_texture (entry points), volume_texture (3D data)
    
    entry_point = texture(back_face_position_texture, FragColor).rgb
    exit_point = FragColor # Or derived from FragPosEye transformed back
    
    # Compute ray direction in Cartesian space
    ray_dir = exit_point - entry_point
    
    # Normalize and step along ray
    num_steps = 100 # Not given in sources, typical value
    step_size = length(ray_dir) / num_steps
    ray_dir = normalize(ray_dir)
    
    accumulated_color = vec4(0.0)
    accumulated_alpha = 0.0
    
    current_pos = entry_point
    
    for i in range(num_steps):
        # Sample 3D texture
        # Note: The source implies sampling the spherical texture. 
        # The ray is in Cartesian, but the texture is spherical.
        # A transformation from Cartesian ray position to Spherical texture coordinates is required.
        # This is marked as [G] as the exact transformation matrix is not provided in the text,
        # but the concept of "transforming bounding box positions to Cartesian... and sampling spherical texture" is explicit.
        spherical_coords = cartesian_to_spherical(current_pos)
        
        density = texture(volume_texture, spherical_coords).r
        
        # Transfer function
        color = transfer_function(density)
        alpha = opacity_function(density)
        
        # Front-to-back compositing
        if accumulated_alpha >= 1.0:
            break
            
        # Standard front-to-back blending
        # C_out = C_in + (1 - A_in) * C_sample * A_sample
        # A_out = A_in + (1 - A_in) * A_sample
        
        weight = (1.0 - accumulated_alpha) * alpha
        accumulated_color += weight * color
        accumulated_alpha += weight
        
        current_pos += ray_dir * step_size
        
    return accumulated_color

# Pass 3: Blend
def blend_to_scene():
    # Render a fullscreen quad with the result of Pass 2
    # Blend mode: Source Alpha, One
    draw_fullscreen_quad(result_texture_from_pass_2)
```

**Parameters:**
*   $P_t$: Radio's power [S1].
*   $L_s$: Other losses (system loss, operation loss) [S1].
*   $N$: Number of radio devices within the frequency bands [S1].
*   $P_i$: Power density of the $i$-th radio [S1].
*   $n$: Number of layers along the altitude direction for grid division [S1].
*   $m \times k$: Grid dimensions along longitude and latitude direction per layer [S1].

**Edge Cases and Pitfalls:**
*   **Irregular Coordinates:** Spherical voxel coordinates are irregular, so standard regular volume rendering approaches fail. Explicit coordinate transformation to Cartesian space is required for ray casting [S1].
*   **Precision:** Not given in the sources.
*   **Boundaries:** The method relies on the bounding box to define the ray entry and exit points. Degenerate bounding boxes (zero volume) would result in zero-length rays [G].

## Operating characteristics
*   **Time Complexity:** Designed for interactive frame rates using high dense parallel computation on GPU [S1].
*   **Memory Complexity:** Requires storage for the 3D volumetric texture ($m \times k \times n$ samples) and the 2D back-face position texture [S1].
*   **Scaling:** The computation of 3D EME power density is independent at each grid, making it suitable for parallel computing on GPU [S1].
*   **Strengths:** Allows dynamic visualization of 3D EME power density coverage by adjusting radio device parameters; leverages GPU hardware for high-performance rendering [S1].
*   **Limitations:** Requires complex coordinate transformations due to the spherical nature of the data; not directly applicable to regular Cartesian volumes without adaptation [S1].
*   **When to use:** When visualizing 3D electromagnetic environment (EME) data on a virtual globe where data is naturally structured in spherical coordinates [S1].
*   **When not:** For regular Cartesian volumetric data where simpler ray-marching techniques (like Bounding Box Raymarching) are sufficient [S1].

## Background
*   **Origin:** The method inherits from GPU-based Direct Volume Rendering (DVR) using 3D textures [9], extended to visualize 3D Electromagnetic Environment (EME) on a virtual globe with spherical coordinates [S1].
*   **History:** Proposed in 2010 by authors publishing in the *I.J. Image, Graphics and Signal Processing* [S1]. It builds upon earlier work on GPU-based DVR [9] and general-purpose GPU computation [17, 18].
*   **Usage:** Used for visualizing 3D electromagnetic environment (EME) on a virtual globe in the described system [S1].

## Sources
[S1] GPU-Based Volume Rendering for 3D Electromagnetic Environment on Virtual Globe, https://doi.org/10.5815/ijigsp.2010.01.07, fetched 2026-10-26
