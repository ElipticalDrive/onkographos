# Volume Raycasting with Front/Back Face Depth Textures

Also called: 3D Texture Raycasting, Entry/Exit Depth Ray Casting, Krüger Ray-Casting, VRC, Volume Ray Casting with Pre-computed Coordinate Textures, Volume Raycaster

## Overview
A volume rendering technique that visualizes 3D scalar fields by projecting a 2D screen onto a 3D volume's bounding box using pre-computed coordinate textures to map screen pixels to 3D entry and exit points, then marching along rays between these faces to accumulate color and alpha [S2, unofficial]. It differs from standard ray casting by avoiding per-ray intersection tests with the volume geometry; instead, it relies on two 2D textures storing the 3D coordinates of the front and back faces of the volume's bounding box for each screen pixel [S1, S2, unofficial]. This method is a specific implementation of the two-pass depth ray casting approach, where the depth information is encoded as full 3D coordinates rather than just depth values [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D scalar field (volume data), a 3D bounding box defining the volume extent, a camera view matrix, and a projection matrix.
*   **Output:** A 2D image where each pixel contains the accumulated color and alpha of the ray passing through the volume.

**Data Structures:**
1.  **3D Texture:** Stores the scalar field values. The layout is a matrix of Z slices with dimensions $M \times N$ divisions in X and Y, and subdivisions in Z. Z increases from top to bottom ($Z=0$ at top) [S2, unofficial].
2.  **Front Face Coordinate Texture:** A 2D texture with resolution matching the screen (or target framebuffer). Each pixel's RGB value represents the 3D world coordinates $(x, y, z)$ of the point on the front face of the bounding box corresponding to that screen pixel [S2, unofficial].
3.  **Back Face Coordinate Texture:** A 2D texture with resolution matching the screen. Each pixel's RGB value represents the 3D world coordinates $(x, y, z)$ of the point on the back face of the bounding box corresponding to that screen pixel [S2, unofficial].

**Parameters:**
*   **Threshold:** A scalar value below which field data is disregarded. Example value: $0.5$ for a level set [S2, unofficial].
*   **Alpha Correction:** A multiplier for the local alpha contribution, typically a number smaller than 1. Example value: $0.1$ [S2, unofficial].
*   **Step Size:** The distance between samples along the ray. Not explicitly defined in sources, but implied by "small steps" [S2, unofficial].

**Algorithm Steps:**
1.  **Generate Coordinate Textures:**
    *   Render the back face of the volume's bounding box to a texture. The fragment shader outputs the 3D world coordinates of the fragment. This creates the Back Face Coordinate Texture.
    *   Render the front face of the volume's bounding box to a separate texture. The fragment shader outputs the 3D world coordinates of the fragment. This creates the Front Face Coordinate Texture.
    *   *Note:* This step effectively pre-computes the ray origins and destinations for every screen pixel [S2, unofficial].

2.  **Ray Casting Pass:**
    For each pixel in the output image:
    1.  Retrieve the 3D front point $P_{front}$ from the Front Face Coordinate Texture.
    2.  Retrieve the 3D back point $P_{back}$ from the Back Face Coordinate Texture.
    3.  Construct the ray vector $\vec{R} = P_{back} - P_{front}$.
    4.  Initialize accumulated color $C_{acc} = (0, 0, 0)$ and accumulated alpha $A_{acc} = 0$.
    5.  Iterate along the ray from $P_{front}$ towards $P_{back}$ with a small step size $\Delta s$. Let the current position be $P_{curr}$.
        a.  Sample the 3D field value $v$ at $P_{curr}$ using trilinear interpolation from the 3D Texture.
        b.  If $v$ is above the Threshold:
            i.  Calculate local color $c$ and local alpha $\alpha$ based on $v$ (using a transfer function).
            ii. Update accumulated color: $C_{acc} = C_{acc} + c \times \alpha_{correction}$.
            iii. Update accumulated alpha: $A_{acc} = A_{acc} + \alpha$.
        c.  If $A_{acc} \ge 1$ or $P_{curr}$ has reached or passed $P_{back}$, break the loop [S2, unofficial].
    6.  The final color for the pixel is $C_{acc}$ [S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pre-computation Step (Render Passes)
# Pass 1: Back Face Coordinates
def render_back_face_coordinates(view_matrix, proj_matrix, bounding_box):
    """
    Renders the back face of the bounding box.
    Output: 2D texture where each pixel stores the 3D world coordinates of the back face.
    """
    # For each pixel in the screen space:
    #   Unproject to get a ray.
    #   Intersect ray with the back face of the bounding box.
    #   Store the intersection point's world coordinates (x, y, z) in the texture.
    pass

# Pass 2: Front Face Coordinates
def render_front_face_coordinates(view_matrix, proj_matrix, bounding_box):
    """
    Renders the front face of the bounding box.
    Output: 2D texture where each pixel stores the 3D world coordinates of the front face.
    """
    # For each pixel in the screen space:
    #   Unproject to get a ray.
    #   Intersect ray with the front face of the bounding box.
    #   Store the intersection point's world coordinates (x, y, z) in the texture.
    pass

# Main Raycasting Pass
def volume_ray_casting(front_coord_texture, back_coord_texture, volume_texture, threshold, alpha_correction):
    """
    Renders the final volume image.
    """
    output_image = empty_2d_texture()
    
    for each pixel (u, v) in output_image:
        # 1. Retrieve entry and exit points
        P_front = front_coord_texture.sample(u, v) # Returns vec3(x, y, z)
        P_back = back_coord_texture.sample(u, v)   # Returns vec3(x, y, z)
        
        # 2. Construct ray
        ray_vector = P_back - P_front
        ray_length = length(ray_vector)
        ray_direction = normalize(ray_vector)
        
        # 3. Initialize accumulators
        C_acc = vec3(0.0)
        A_acc = 0.0
        
        # 4. March along the ray
        # Step size is not explicitly defined in sources. 
        # A common implementation detail [G] is to use a fixed step size or 
        # a fraction of the ray length. We assume a small fixed step size 'step_size'.
        step_size = 0.01 # Placeholder for implementation detail [G]
        
        t = 0.0
        while t < ray_length:
            P_curr = P_front + ray_direction * t
            
            # Sample volume
            v = volume_texture.sample_trilinear(P_curr)
            
            # Check threshold
            if v > threshold:
                # Calculate local color and alpha based on v
                # Transfer function details are not provided in sources [G]
                c = transfer_function_color(v)
                alpha = transfer_function_alpha(v)
                
                # Update accumulators
                C_acc = C_acc + c * alpha_correction
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
*   **Premature Termination:** The loop must break if accumulated alpha reaches 1 (full opacity) to avoid unnecessary computation, or if the back face is reached [S2, unofficial].
*   **Precision:** The coordinate textures store 3D coordinates. Precision loss in texture sampling or coordinate representation can lead to artifacts, especially if the bounding box is large or the texture resolution is low [S2, unofficial].
*   **Degenerate Input:** If the camera is inside the volume, the front and back face coordinates might be swapped or invalid. The method assumes the camera is outside the volume [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N \cdot M \cdot S)$, where $N$ and $M$ are the screen dimensions, and $S$ is the number of steps along the ray. The number of steps depends on the ray length and step size.
*   **Memory Complexity:** $O(V + N \cdot M)$, where $V$ is the size of the 3D volume texture, and $N \cdot M$ is the size of the two coordinate textures.
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Performance scales linearly with screen resolution and ray length. The use of pre-computed coordinates avoids expensive ray-box intersection tests per pixel, but requires two additional render passes for coordinate generation.
*   **Strengths:** Simple implementation, avoids complex ray-triangle intersection logic for the volume boundary.
*   **Limitations:** Requires two additional render passes. Precision of the coordinate textures can affect accuracy. Not suitable for dynamic volumes where the bounding box changes frequently, as the coordinate textures must be regenerated.
*   **When to use:** Static volumes with simple bounding boxes.
*   **When not to use:** Dynamic volumes, complex geometries, or when high precision is required without high-resolution coordinate textures.

## Background
The method is described in the context of WebGL for Numerical PDEs, specifically Session 11 on Volume Ray Casting [S2, unofficial]. It is presented as a practical approach for visualizing 3D scalar fields, such as voltage fields, using pre-computed coordinate textures. The technique is also referred to as Krüger Ray-Casting in some contexts [S1, S2, unofficial].

## Sources
[S1] mversiotech/volume-rendering-demo, https://github.com/mversiotech/volume-rendering-demo, fetched 2026-10-26 (unofficial)
[S2] WebGL for Numerical PDEs (Session 11): Volume Ray Casting, https://www.youtube.com/watch?v=X095f9QzSBo, fetched 2026-10-27 (unofficial)
