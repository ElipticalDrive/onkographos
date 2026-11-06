# Hardware-accelerated surface voxelization via 2D textures

Also called: Image-space voxelization

## Overview
This method transfers intersection computation from geometric space to image space by rasterizing model surfaces into 2D textures, enabling collision detection and volumetric rendering at real-time frame rates [S1, unofficial]. It differs from slicing-based voxelization by rasterizing the model approximately once rather than $N$ times (where $N$ is the volume resolution), making performance insensitive to scene complexity [S1, unofficial]. Unlike Layered Depth Image (LDI) extraction, it is view-independent and avoids CPU-side sorting of depth lists [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Polygonal mesh (vertices, normals, triangles) or implicit surface definition; volume resolution $D$; bounding box parameters ($p_{min}, p_{max}$).
*   **Output:** 2D texture(s) representing the voxelized volume data in image space.

**Data Structures:**
The volume is stored in 2D textures (worksheets). The bit-depth per voxel depends on the resolution and memory constraints [S1, unofficial]:
*   **512^3 resolution:** 1 bit per voxel. Stored in one worksheet.
*   **256^3 resolution:** 8 bits per voxel.
*   **128^3 resolution:** 16 bits per voxel (used to encode vertex normals).
*   **64^3 resolution:** 8 bits per voxel.

**Steps:**
1.  **Rasterization:** Rasterize the model surfaces into 2D textures. This step traverses the model approximately once. The performance depends mainly on the number of triangles [S1, unofficial].
2.  **Texelization and Synthesis:** Convert rasterized fragments into voxel data. This is an image-space operation dominated by volume resolution and bit-depth [S1, unofficial].
3.  **Collision Query:** Perform collision detection by querying the generated 2D textures in image space [S1, unofficial].
4.  **Rendering:** Render the volume data and voxelized slices orderly in 3D space to display complex deformable geometric models semi-transparently [S1, unofficial].

**Formulas:**
The mapping from world position $p$ to voxel index $i$ follows the standard formula defined in the parent article [G]:
$$
i = D \times \frac{p - p_{min}}{p_{max} - p_{min}}
$$
For implicit surfaces, the surface is defined by $f(x,y,z) = 0$. The example given is:
$$
x^4 - 5x^2 + y^4 - 5y^2 + z^4 - 5z^2 + 11.8 = 0
$$
Both surface and solid voxelization for this implicit surface take 25ms at 256^3 resolution [S1, unofficial].

**Parameters:**
*   **Volume Resolution:** 256^3 is identified as a good choice under current hardware conditions (ATI Radeon 9800 Pro) [S1, unofficial].
*   **Bit-depth:** 8 bits per voxel for the standard 256^3 configuration [S1, unofficial].
*   **Hardware:** Implemented using Vertex/Pixel Shader 2.0 of Direct3D 9.0b [S1, unofficial].

**Pitfalls:**
*   **Aliasing:** Aliasing effects exist in solid voxelization for closed polygonal models, appearing as prolonged lines where some voxels at the rasterization stage are missing [S1, unofficial].
*   **Bit-operations Cost:** Bit-operations are much more expensive than byte-operations. Storing a 512^3 volume in one worksheet using 1-bit depth results in significantly higher processing times compared to 8-bit depth at lower resolutions [S1, unofficial].
*   **Memory Constraints:** If multivalued voxelization is required for high resolution, multiple worksheets are needed given sufficient video memory. If floating-point texture blending is supported, fewer worksheets are needed [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Hardware-accelerated surface voxelization via 2D textures
# Note: This is a high-level representation of the GPU pipeline described in [S1, unofficial].
# Actual implementation relies on Direct3D 9.0b Vertex/Pixel Shaders.

def hardware_accelerated_voxelization(mesh, volume_resolution, bounding_box):
    """
    Voxelize a mesh into 2D textures using hardware rasterization.
    
    Args:
        mesh: List of triangles (vertices, normals).
        volume_resolution: Integer D (e.g., 256).
        bounding_box: Tuple (p_min, p_max).
        
    Returns:
        2D textures representing the voxelized volume.
    """
    # 1. Setup 2D texture(s) based on resolution and bit-depth
    # For 256^3, use 8-bit depth. For 512^3, use 1-bit depth.
    bit_depth = get_bit_depth(volume_resolution)
    worksheets = allocate_worksheets(volume_resolution, bit_depth)
    
    # 2. Rasterize model surfaces into 2D textures
    # This step traverses the model approximately once.
    # Performance depends on triangle count.
    for triangle in mesh:
        # Project triangle vertices to screen space
        screen_vertices = project_to_screen(triangle.vertices, camera_matrix)
        
        # Rasterize triangle to coverage mask on 2D texture
        # The GPU handles the rasterization via Pixel Shader 2.0
        rasterize_triangle_to_texture(screen_vertices, worksheets)
        
    # 3. Texelization and Synthesis
    # Convert rasterized fragments to voxel data.
    # This is an image-space operation dominated by volume resolution.
    voxel_data = synthesize_voxel_data(worksheets, volume_resolution)
    
    return voxel_data

def get_bit_depth(resolution):
    """
    Determine bit-depth based on resolution and hardware constraints.
    Based on [S1, unofficial]:
    - 512^3: 1 bit
    - 256^3: 8 bits
    - 128^3: 16 bits (for normal encoding)
    - 64^3: 8 bits
    """
    if resolution == 512:
        return 1
    elif resolution == 256:
        return 8
    elif resolution == 128:
        return 16
    elif resolution == 64:
        return 8
    else:
        # Not given in the sources.
        return 8

def project_to_screen(vertices, camera_matrix):
    """
    Project 3D vertices to 2D screen coordinates.
    Standard graphics operation.
    """
    # Implementation depends on specific graphics API (Direct3D 9.0b)
    # [G] Standard perspective projection and viewport transformation.
    pass

def rasterize_triangle_to_texture(screen_vertices, worksheets):
    """
    Rasterize a triangle into the 2D texture worksheets.
    The GPU handles the actual rasterization.
    """
    # [G] This step is handled by the graphics hardware's rasterizer.
    # The fragment shader writes to the texture based on coverage.
    pass

def synthesize_voxel_data(worksheets, volume_resolution):
    """
    Synthesize final voxel data from worksheets.
    """
    # [G] Combine worksheets if necessary.
    # For 256^3 with 8-bit depth, the data is directly usable.
    pass
```

## Operating characteristics
*   **Time Complexity:** The algorithm traverses the model approximately once, making performance insensitive to scene complexity [S1, unofficial]. Rasterization timings depend mainly on the number of triangles [S1, unofficial]. Texelization and synthesis are image-space operations dominated by volume resolution and bit-depth [S1, unofficial].
*   **Memory Complexity:** Depends on volume resolution and bit-depth. For 512^3 with 1-bit depth, memory is 16MB [S1, unofficial]. For 256^3 with 8-bit depth, memory is 16MB [S1, unofficial]. For 128^3 with 16-bit depth, memory is 4MB [S1, unofficial]. For 64^3 with 8-bit depth, memory is 256KB [S1, unofficial].
*   **Measured Performance:**
    *   **Hardware:** PC with 2.4 GHz Pentium IV CPU, 2GB RAM, ATI Radeon 9800 Pro (256MB RAM) [S1, unofficial].
    *   **Blade Model (1,765,388 triangles):** 75ms for rasterization at 256^3 resolution [S1, unofficial]. Slicing-based voxelization for the same model at 256^3 costs about 5 seconds (20.8ms x 256) [S1, unofficial].
    *   **Wagner Model (60,246 triangles):**
        *   512^3, 1-bit: 500ms [S1, unofficial].
        *   256^3, 8-bit: 30ms [S1, unofficial].
        *   128^3, 16-bit: 24ms [S1, unofficial].
        *   64^3, 8-bit: 21ms [S1, unofficial].
    *   **Implicit Surface:** 25ms at 256^3 resolution [S1, unofficial].
    *   **Visualization:** Costs about 8ms on average, depending mainly on volume resolution [S1, unofficial].
    *   **Hybrid Rendering:** Transparent cerebra model (36,758 triangles) with MRI head volume (128^3) achieves about 10 fps [S1, unofficial].
*   **Strengths:** Real-time frame rates for collision detection and volumetric rendering; insensitive to scene complexity; simple, robust, and easy-to-implement [S1, unofficial].
*   **Limitations:** Aliasing effects in solid voxelization for closed polygonal models; bit-operations are expensive; view-independent but requires sufficient video memory for high-resolution multivalued voxelization [S1, unofficial].
*   **When to Use:** Surgical planning, intra-operation navigation, and radiation therapy [S1, unofficial]. When real-time collision detection and volumetric rendering are required for complex models [S1, unofficial].
*   **When Not to Use:** When high-resolution multivalued voxelization is required and video memory is insufficient [S1, unofficial]. When aliasing artifacts are unacceptable without post-processing [S1, unofficial].

## Background
This technique was presented in "Real-time Voxelization for Complex Models" [S1, unofficial]. It was implemented using Vertex/Pixel Shader 2.0 of Direct3D 9.0b [S1, unofficial]. The method addresses the bottleneck of traditional slicing-based voxelization, which rasterizes the model $N$ times, and LDI extraction, which is view-dependent and requires CPU-side sorting [S1, unofficial]. It is particularly useful in fields requiring fast or real-time voxelization for interactive graphics applications [S1, unofficial].

## Sources
[S1] Real-time Voxelization for Complex Models, https://www.researchgate.net/publication/252533918_Real-time_Voxelization_for_Complex_Models, fetched 2026-11-03 (unofficial)
