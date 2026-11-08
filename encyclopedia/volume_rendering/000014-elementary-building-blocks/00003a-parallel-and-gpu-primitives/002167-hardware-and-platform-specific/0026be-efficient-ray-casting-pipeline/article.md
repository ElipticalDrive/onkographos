# Efficient Ray Casting Pipeline

Also called: EfficientRayCast, Raycasting Rendering Pipeline

## Overview
This article describes two distinct implementations sharing the name "Efficient Ray Casting Pipeline," which differ fundamentally in their rendering targets and hardware acceleration models. The first is a CPU-based 2D raycasting engine for pseudo-3D first-person views, optimized for minimal per-pixel arithmetic and direct framebuffer manipulation [S1, unofficial]. The second is a multi-pass Direct3D 11 pipeline for 3D volume raycasting, utilizing geometry shaders for active brick detection and ray generation [S2, unofficial]. Both methods prioritize predictable, low-latency operations over complex mathematical approximations to achieve real-time performance on their respective platforms.

## How it works

### CPU-Based 2D Raycasting (Wolfenstein-style)
This method renders a 2D map as a pseudo-3D view by casting one ray per screen column.

**Inputs and Outputs**
*   **Input:** A 2D grid map (walls/empty), camera position, camera direction, and viewport dimensions.
*   **Output:** A raw pixel buffer (e.g., `Uint32Array`) representing the final image.

**Data Structures**
*   **Map:** A 2D array or grid representing wall cells.
*   **Framebuffer:** A linear array of pixel values, mapped directly to the canvas's raw pixel buffer.

**Steps**
1.  **Initialize Framebuffer:** Clear the raw pixel buffer.
2.  **Column Loop:** Iterate through each column $x$ of the screen.
3.  **Ray Casting:** Cast a single ray from the camera origin through the current column's angle.
4.  **Voxel Traversal:** Use a fast voxel traversal algorithm (e.g., Amanatides & Woo) to find the first wall intersection [S1, S3].
5.  **Distance Calculation:** Calculate the perpendicular distance to the wall to correct fisheye distortion.
6.  **Vertical Strip Rendering:** Determine the vertical span of the wall strip based on distance.
7.  **Fog and Color:** For each pixel in the strip, perform a texture fetch and fog multiply. The hot loop consists of three integer multiplies and shifts per pixel [S1, unofficial].
8.  **Direct Write:** Write the final color directly into the `Uint32Array` view of the canvas buffer.

**Parameters**
*   **Resolution:** Internal framebuffer resolution (e.g., 480x300) [S1, unofficial].
*   **Upscaling:** Integer scale factor applied to the internal framebuffer to fill the viewport, using pixelated filtering (`image-rendering: pixelated`) [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_frame(map_grid, camera, viewport_width, viewport_height):
    # Internal low-res framebuffer
    fb_width = 480
    fb_height = 300
    framebuffer = [0] * (fb_width * fb_height)
    
    # Scale factor for upscaling
    scale_x = viewport_width / fb_width
    scale_y = viewport_height / fb_height
    
    for col in range(fb_width):
        # Calculate ray angle for this column
        ray_angle = camera.angle + atan((col - fb_width / 2) / (fb_width / 2))
        
        # Cast ray to find wall distance
        dist = cast_ray(map_grid, camera.pos, ray_angle)
        
        # Correct fisheye
        corrected_dist = dist * cos(ray_angle - camera.angle)
        
        # Calculate wall height on screen
        wall_height = fb_height / corrected_dist
        
        # Determine vertical start and end
        start_y = (fb_height - wall_height) / 2
        end_y = start_y + wall_height
        
        # Render vertical strip
        for row in range(fb_height):
            if start_y <= row < end_y:
                # Texture coordinate for fog/color
                tex_y = (row - start_y) / wall_height
                
                # Fog multiply: three integer multiplies and shifts [S1, unofficial]
                color = fog_multiply(tex_y)
                
                # Write to framebuffer
                framebuffer[row * fb_width + col] = color

    # Upscale to viewport
    for y in range(fb_height):
        for x in range(fb_width):
            pixel = framebuffer[y * fb_width + x]
            # Write to raw pixel buffer with integer scaling
            for dy in range(scale_y):
                for dx in range(scale_x):
                    target_y = y * scale_y + dy
                    target_x = x * scale_x + dx
                    if target_y < viewport_height and target_x < viewport_width:
                        write_pixel(target_x, target_y, pixel)
```

### GPU-Based 3D Volume Raycasting (D3D11)
This method uses a multi-pass pipeline to render volumetric data using geometry shaders.

**Inputs and Outputs**
*   **Input:** Volumetric data (bricked or unstructured), view/projection matrices, camera position, density range constants.
*   **Output:** Final output texture (`m_pOutputTex`).

**Data Structures**
*   **Constant Buffer (`CB_EfficientRayCast_Frame`):**
    *   `mInvView`: Inverse View Matrix (XMMATRIX)
    *   `mView`: View Matrix (XMMATRIX)
    *   `mViewProj`: View-Projection Matrix (XMMATRIX)
    *   `f4ViewPos`: Camera position (XMFLOAT4)
    *   `f2HalfWinSize`: Half window size (XMFLOAT2)
    *   `f2WinReso`: Window resolution (XMFLOAT2)
    *   `f2MinMaxDensity`: Density range constants (`MIN_DENSITY`, `MAX_DENSITY`) (XMFLOAT2)
    *   `NIU`: Normalization factor (float)
    *   `NIU1`: Normalization factor + 1 (float)
*   **Textures:**
    *   `m_pFarNearTex`: Stores near and far ray positions.
    *   `m_pOutputTex`: Final rendered output.
    *   `m_pBrickVolTex`: Bricked volume texture.

**Steps**
1.  **Reset Volume (Compute Shader):** Clear the data volume and optionally the flag volume using compute shaders (`m_pResetVolumeCS`, `m_pRestFlagVolumeCS`) [S2, unofficial].
2.  **Update Volume (Compute Shader):** Update the volume data based on time and animated parameters (e.g., ball positions) using `m_pVolumeUpdatewithFlagUpdateCS` [S2, unofficial].
3.  **Active Brick Detection (Geometry Shader):** Use `m_pActiveCellAndSOGS` to detect active bricks and generate cube meshes. This step filters out empty space [S2, unofficial].
4.  **Ray Generation and Marching (Geometry Shader):** Use `m_pRaymarchGS` to cast rays through the active bricks. This shader handles the ray marching logic and compositing [S2, unofficial].
5.  **Output:** Write the result to `m_pOutputTex`.

**Parameters**
*   `MIN_DENSITY`, `MAX_DENSITY`: Constants defining the density range, initialized in the constructor [S2, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Note: This is a high-level description of the D3D11 pipeline steps.
# Actual implementation requires HLSL shaders and D3D11 API calls.

def render_volume(device_context, volume_data, camera_info):
    # 1. Reset volumes
    device_context.CSSetShader(m_pResetVolumeCS, None, 0)
    uavs = [m_pVolUAV, m_pBrickVolUAV]
    device_context.CSSetUnorderedAccessViews(0, 2, uavs, [0, 0])
    device_context.Dispatch(VOXEL_NUM_X // THREAD_X, VOXEL_NUM_Y // THREAD_Y, VOXEL_NUM_Z // THREAD_Z)
    
    # 2. Update volume data (if animated)
    device_context.UpdateSubresource(m_pCB_VolumeInfo, 0, None, camera_info, 0, 0)
    device_context.CSSetConstantBuffers(0, 1, [m_pCB_VolumeInfo])
    device_context.CSSetShader(m_pVolumeUpdatewithFlagUpdateCS, None, 0)
    device_context.Dispatch(...) # Dispatch based on brick ratio
    
    # 3. Active Brick Detection and Ray Casting
    # Set up geometry shaders for brick detection and ray marching
    device_context.GSSetShader(m_pActiveCellAndSOGS, None, 0)
    # Generate cube meshes for active bricks
    
    device_context.GSSetShader(m_pRaymarchGS, None, 0)
    # Cast rays and composite into m_pFarNearTex and m_pOutputTex
    
    # 4. Present Output
    device_context.Present(0, 0)
```

## Operating characteristics

### CPU-Based 2D Raycasting
*   **Time Complexity:** $O(C \times H + S)$, where $C$ is the number of screen columns, $H$ is the screen height, and $S$ is the number of visible sprites [S1, unofficial]. The rendering time depends on screen resolution, not map size.
*   **Memory Complexity:** $O(W \times H)$ for the framebuffer, where $W$ and $H$ are the internal framebuffer dimensions (e.g., 480x300) [S1, unofficial].
*   **Strengths:** Extremely fast per-pixel operations (three integer multiplies and shifts); no floating-point heavy lifting in the hot loop; simple to implement in JavaScript or C [S1, unofficial].
*   **Limitations:** Limited to 2D maps with vertical walls; no true 3D perspective for non-vertical surfaces; low internal resolution requires upscaling [S1, unofficial].
*   **When to Use:** Retro-style FPS games, browser-based demos, or educational purposes where simplicity and speed are paramount [S1, unofficial].

### GPU-Based 3D Volume Raycasting
*   **Time Complexity:** Depends on the number of active bricks and the ray marching steps per brick. The compute shader steps are $O(V)$ for volume update, where $V$ is the number of voxels [S2, unofficial].
*   **Memory Complexity:** $O(V)$ for the volume data and $O(B)$ for the bricked data, where $B$ is the number of bricks [S2, unofficial].
*   **Strengths:** Leverages GPU parallelism for volume updates and ray marching; geometry shaders allow for efficient culling of empty space [S2, unofficial].
*   **Limitations:** Requires Direct3D 11 support; complex shader code; memory bandwidth intensive for large volumes [S2, unofficial].
*   **When to Use:** Scientific visualization, medical imaging, or real-time 3D volume rendering where hardware acceleration is available [S2, unofficial].

## Background
The CPU-based raycasting method is inspired by early FPS games like Wolfenstein 3D, which used 2.5D raycasting to render pseudo-3D environments [S1, S5]. The GPU-based volume raycasting method is part of a broader family of GPU-accelerated volume rendering techniques, utilizing modern graphics APIs like Direct3D 11 to handle complex volumetric data [S2, unofficial].

## Sources
[S1] A minimal Wolfenstein 3D raycasting engine, https://4rknova.com/blog/2026/07/06/wolfenstein-3d-raycasting, fetched 2026-10-24 (unofficial)
[S2] pengliu916/DynamicVolumeRayCasting, https://github.com/pengliu916/DynamicVolumeRayCasting, fetched 2026-10-26 (unofficial)
