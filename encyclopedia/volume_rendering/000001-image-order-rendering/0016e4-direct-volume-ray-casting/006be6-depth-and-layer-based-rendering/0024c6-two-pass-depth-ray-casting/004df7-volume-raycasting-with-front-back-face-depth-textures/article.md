# Volume Raycasting with Front/Back Face Depth Textures

Also called: Dual-Texture Enter/Exit Vector Method, Enter/Exit 3D vectors method, Entry/Exit Depth Ray Casting, Front/Back face texture method, Ray Casting with Pre-computed Entry/Exit Textures, Volume Ray Casting with Pre-computed Entry/Exit Textures

## Overview
A volume rendering technique that visualizes 3D scalar fields by projecting a 2D screen onto a 3D volume's bounding box using pre-computed coordinate textures to map screen pixels to 3D entry and exit points, then marching along rays between these faces to accumulate color and alpha [S3, unofficial]. It differs from standard ray casting by avoiding per-ray intersection tests with the volume geometry; instead, it relies on two 2D textures storing the 3D coordinates of the front and back faces of the volume's bounding box for each screen pixel [S1, unofficial][S2, unofficial]. This method is a specific implementation of the two-pass depth ray casting approach, where the depth information is encoded as full 3D coordinates rather than just depth values [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D scalar field (volume data), a 3D bounding box defining the volume extent, a camera view matrix, and a projection matrix [S2, unofficial][S3, unofficial].
*   **Output:** A 2D image where each pixel contains the accumulated color and alpha of the ray passing through the volume [S3, unofficial].

**Data Structures:**
1.  **3D Texture:** Stores the scalar field values. The layout is a matrix of Z slices with dimensions $M \times N$ divisions in X and Y, and subdivisions in Z. Z increases from top to bottom ($Z=0$ at top) [S3, unofficial].
2.  **Front Face Coordinate Texture:** A 2D texture with resolution matching the screen (or target framebuffer). Each pixel's RGB value represents the 3D world coordinates $(x, y, z)$ of the point on the front face of the bounding box corresponding to that screen pixel [S2, unofficial][S3, unofficial].
3.  **Back Face Coordinate Texture:** A 2D texture with resolution matching the screen. Each pixel's RGB value represents the 3D world coordinates $(x, y, z)$ of the point on the back face of the bounding box corresponding to that screen pixel [S2, unofficial][S3, unofficial].

**Parameters:**
*   **Threshold:** A scalar value below which field data is disregarded. Example value: $0.5$ for a level set [S3, unofficial].
*   **Alpha Correction:** A multiplier for the local alpha contribution, typically a number smaller than 1. Example value: $0.1$ [S3, unofficial].
*   **Step Size:** The distance between samples along the ray. Not given in the sources. [G]

**Algorithm Steps:**
1.  **Generate Coordinate Textures:**
    *   Initialize a framebuffer object (FBO) and allocate two RGBA UNSIGNED_BYTE textures (`texEntr`, `texExit`) with LINEAR mag/min filtering [S2, unofficial].
    *   Bind the FBO, set the viewport to the target resolution (e.g., 1024×1024), and enable face culling [S2, unofficial].
    *   Set cull mode to BACK, bind `texExit` to COLOR_ATTACHMENT0, clear the color and depth buffers, and draw the bounding box (e.g., a cube from $(0,0,0)$ to $(1,1,1)$) [S2, unofficial][S4, unofficial].
    *   Set cull mode to FRONT, bind `texEntr` to COLOR_ATTACHMENT0, clear the buffers, and draw the bounding box again [S2, unofficial][S4, unofficial].
    *   The fragment shader for these passes outputs the 3D world coordinates of the fragment [S2, unofficial][S4, unofficial].
2.  **Ray Casting Pass:**
    *   Bind `texEntr` to texture unit 0, `texExit` to unit 1, the volume texture to unit 2, and the transfer function texture to unit 3 [S2, unofficial].
    *   Draw a full-screen quad (TRIANGLE_FAN) [S2, unofficial].
    *   For each pixel: retrieve the 3D front point $P_{front}$ from `texEntr` and the 3D back point $P_{back}$ from `texExit` [S2, unofficial][S3, unofficial].
    *   Construct the ray vector $\vec{R} = P_{back} - P_{front}$ [S3, unofficial].
    *   Initialize accumulated color $C_{acc} = (0, 0, 0)$ and accumulated alpha $A_{acc} = 0$ [S3, unofficial].
    *   Iterate along the ray from $P_{front}$ towards $P_{back}$ with a small step size $\Delta s$ [G].
    *   Sample the 3D field value $v$ at the current position using trilinear interpolation from the 3D Texture [S3, unofficial].
    *   If $v$ is above the Threshold: calculate local color $c$ and local alpha $\alpha$ based on $v$ (using a transfer function), update accumulated color $C_{acc} = C_{acc} + c \times \alpha_{correction}$, and update accumulated alpha $A_{acc} = A_{acc} + \alpha$ [S3, unofficial].
    *   If $A_{acc} \ge 1$ or the back face is reached, break the loop [S3, unofficial].
    *   The final color for the pixel is $C_{acc}$ [S3, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_coordinate_textures(view_matrix, proj_matrix, bounding_box, texture_back, texture_front):
    """
    Pass 1: Render back face coordinates.
    Pass 2: Render front face coordinates.
    """
    # Sources do not provide the exact shader logic for coordinate generation. [G]
    pass

def volume_ray_casting(front_coord_texture, back_coord_texture, volume_texture, transfer_function, threshold, alpha_correction, step_size):
    """
    Main raycasting pass.
    """
    output_image = empty_2d_texture()
    
    for u, v in output_image.dimensions():
        # 1. Retrieve entry and exit points
        P_front = front_coord_texture.sample(u, v)
        P_back = back_coord_texture.sample(u, v)
        
        # 2. Construct ray
        ray_vector = P_back - P_front
        ray_length = length(ray_vector)
        ray_direction = normalize(ray_vector)
        
        # 3. Initialize accumulators
        C_acc = (0.0, 0.0, 0.0)
        A_acc = 0.0
        
        t = 0.0
        while t < ray_length:
            P_curr = P_front + ray_direction * t
            v = volume_texture.sample_trilinear(P_curr)
            
            # Check threshold
            if v > threshold:
                # Calculate local color and alpha based on v
                c, alpha = transfer_function(v)
                
                # Update accumulators
                C_acc = tuple(C_acc[i] + c[i] * alpha_correction for i in range(3))
                A_acc = A_acc + alpha
                
                # Check for termination
                if A_acc >= 1.0:
                    break
            
            t += step_size
            
        # 5. Store result
        output_image.set_pixel(u, v, C_acc)
        
    return output_image
```

**Edge Cases and Pitfalls:**
*   **Premature Termination:** The loop must break if accumulated alpha reaches 1 (full opacity) to avoid unnecessary computation, or if the back face is reached [S3, unofficial].
*   **Degenerate Input:** If the camera is inside the volume, the front and back face coordinates might be swapped or invalid. Not given in the sources. [G]

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Not given in the sources.
*   **Strengths:** The use of pre-computed coordinates avoids expensive ray-box intersection tests per pixel, but requires two additional render passes for coordinate generation [S2, unofficial][S4, unofficial].
*   **Limitations:** Precision of the coordinate textures can affect accuracy. Not given in the sources. [G]
*   **When to use:** Static volumes with simple bounding boxes. Not given in the sources. [G]
*   **When not to use:** Dynamic volumes where the bounding box changes frequently, as the coordinate textures must be regenerated. Not given in the sources. [G]

## Background
The method is described in the context of WebGL for Numerical PDEs, specifically Session 11 on Volume Ray Casting [S3, unofficial]. It is presented as a practical approach for visualizing 3D scalar fields, such as voltage fields, using pre-computed coordinate textures [S3, unofficial].

## Sources
[S1] mversiotech/volume-rendering-demo, https://github.com/mversiotech/volume-rendering-demo, fetched 2026-10-26 (unofficial)
[S2] chrisyeshi/VolCast, https://github.com/chrisyeshi/VolCast, fetched 2026-10-26 (unofficial)
[S3] WebGL for Numerical PDEs (Session 11): Volume Ray Casting, https://www.youtube.com/watch?v=X095f9QzSBo, fetched 2026-10-27 (unofficial)
[S4] Display voxels using shaders in openGL, https://stackoverflow.com/questions/24394580/display-voxels-using-shaders-in-opengl, fetched 2026-10-27 (unofficial)
