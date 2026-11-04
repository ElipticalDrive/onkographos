# Back-face rasterized volume rendering

Also called: Back-Face Entry Point Method, Backface Rendering for Ray Casting, Backface Culling for Ray Casting Entry Points

## Overview
Back-face rasterized volume rendering is a GPU-based technique that determines ray entry and exit points for volume raycasting by rasterizing the bounding box geometry of the volume, rather than computing intersections analytically in the fragment shader [S1, S2, S3, S4, S5, unofficial]. It solves the problem of efficiently restricting fragment shader execution to only those pixels that intersect the volume, avoiding unnecessary processing for empty screen regions, while correctly handling camera zooms where front faces might project behind the camera and be clipped [S2, unofficial]. It differs from [Direct volume ray casting](19-direct-volume-ray-casting.md) by offloading the ray-boundary computation to the rasterization pipeline via a two-pass (or single-pass with geometry shaders) approach, and from [Bounding Box Raymarching](2071-bounding-box-raymarching.md) by explicitly storing the back-face coordinates in a texture to be sampled during the front-face pass, rather than computing endpoints directly in the fragment shader [S2, S3, unofficial].

## How it works
The method uses the parent's shared principles of ray casting, discretization, and front-to-back compositing [S2, S3, S8, S9, S11, S14, S16, S20, S23, S26, unofficial]. It replaces the standard ray-box intersection calculation with a geometry-driven approach.

**Inputs and Outputs:**
*   **Inputs:** 3D volume texture ($V$), 1D or 2D transfer function texture ($TF$), camera parameters (eye position $O$, projection/view matrices), bounding box geometry (unit cube $[0,1]^3$ scaled by volume dimensions), viewport resolution $(W, H)$.
*   **Outputs:** Final color image.

**Data Structures:**
1.  **Bounding Box Geometry:** A cube defined by 8 vertices in object coordinates $[0,1]^3$ and 36 indices forming 12 triangles [S3, S4, unofficial].
2.  **Frame Buffer Object (FBO):** Attached to a 2D texture (`g_bfTexObj` or `u_ExitPoints`) with internal format `GL_RGBA16F` (or equivalent high-precision float format) and dimensions matching the viewport $(W, H)$ [S1, S4, unofficial].
3.  **Back-face Texture:** Stores the 3D entry points (or exit points, depending on culling convention) as RGB color values [S1, S4, unofficial].

**Numbered Steps:**

1.  **Pass 1: Compute Entry/Exit Points.**
    *   Bind the FBO attached to `g_bfTexObj`.
    *   Enable face culling and set `glCullFace` to `GL_FRONT` (to render back faces) [S1, S3, S4, unofficial].
    *   Render the bounding box geometry using a vertex shader that outputs the vertex position (in object/texture space) as a varying color, and a fragment shader that writes this position to the FBO texture [S1, S3, S4, unofficial].
    *   Unbind the FBO. The texture now contains the ray entry points for every pixel covered by the back face of the bounding box [S1, S4, unofficial].

2.  **Pass 2: Ray Marching and Compositing.**
    *   Bind the default framebuffer.
    *   Enable face culling and set `glCullFace` to `GL_BACK` (to render front faces) [S1, S3, S4, unofficial].
    *   Render the bounding box geometry using the raycasting shaders.
    *   **Vertex Shader:** Outputs the vertex position (front face) as `EntryPoint` [S4, unofficial].
    *   **Fragment Shader:**
        1.  Sample the back-face texture (`g_bfTexObj`) using the current fragment's normalized coordinates (`gl_FragCoord.xy / viewport_resolution`) to retrieve the corresponding back-face position (`ExitPoint`) [S1, S4, unofficial].
        2.  Calculate the ray direction vector: $\vec{d} = \text{ExitPoint} - \text{EntryPoint}$ [S1, S4, unofficial].
        3.  Calculate the ray length: $L = \|\vec{d}\|$ [S4, unofficial].
        4.  If $L \approx 0$ (entry equals exit, e.g., tangent ray), discard the fragment to avoid division by zero [S4, unofficial].
        5.  Compute the step vector: $\vec{s} = \frac{\vec{d}}{N}$, where $N$ is the number of sampling steps (or compute step size based on voxel dimensions) [S1, S2, S5, unofficial].
        6.  Initialize accumulated color $C' = 0$ and opacity $\alpha' = 0$ [S2, S3, S8, S9, S11, S14, S16, S20, S23, S26, unofficial].
        7.  Initialize current position $P = \text{EntryPoint}$ [S1, S4, unofficial].
        8.  Loop for $i = 0$ to $N-1$:
            *   Sample volume value $v = V(P)$ [S1, S2, S3, S4, S5, unofficial].
            *   Apply transfer function: $(C_i, \alpha_i) = TF(v)$ [S2, S3, S4, S5, unofficial].
            *   Update accumulated color: $C' = C' + (1 - \alpha') \alpha_i C_i$ [S2, S3, S4, S5, unofficial].
            *   Update accumulated opacity: $\alpha' = \alpha' + (1 - \alpha') \alpha_i$ [S2, S3, S4, S5, unofficial].
            *   Advance position: $P = P + \vec{s}$ [S1, S2, S3, S4, S5, unofficial].
            *   **Early Termination:** If $\alpha' \ge 1.0$ (or a threshold like 0.95/0.99), break the loop [S1, S2, S3, S4, unofficial].
            *   **Boundary Check:** If $P$ goes outside $[0,1]^3$, break [S1, unofficial].
        9.  Output final color $C'$ [S2, S3, S4, S5, unofficial].

**Formulas:**
*   Ray Direction: $\vec{d} = P_{back} - P_{front}$ [S1, S4, unofficial].
*   Step Vector: $\vec{s} = \frac{\vec{d}}{N}$ [S1, S4, unofficial].
*   Compositing: $C'_i = C'_{i-1} + (1 - \alpha'_{i-1}) \alpha_i C_i$ [S2, S3, S8, S9, S11, S14, S16, S20, S23, S26, unofficial].
*   Opacity: $\alpha'_i = \alpha'_{i-1} + (1 - \alpha'_{i-1}) \alpha_i$ [S2, S3, S8, S9, S11, S14, S16, S20, S23, S26, unofficial].

**Parameters:**
*   `uSteps` / `N`: Number of sampling steps. Typical value: 256 [S1, S3, unofficial].
*   `uOpacityVal`: Opacity scaling factor. Typical value: 4.0 [S1, unofficial].
*   `uMinGrayVal` / `uMaxGrayVal`: Volume value thresholds. Typical: 0.001 to 1.0 [S1, unofficial].
*   `DELTA`: Constant step size for gradient/ray marching. Value: 0.005 [S3, unofficial].
*   `SHADING_THRES`: Threshold for shading. Value: 0.10 [S3, unofficial].

**Edge Cases and Pitfalls:**
*   **Zooming:** Rendering only front faces causes clipping when the camera zooms in, as front faces project behind the camera. Rendering back faces avoids this [S2, unofficial].
*   **Tangent Rays:** If the entry point equals the exit point, the ray length is zero. The shader must discard the fragment to prevent division by zero or infinite loops [S4, unofficial].
*   **Loop Termination:** GLSL loop limits may require a large constant (e.g., 4095) with an explicit break condition based on `uSteps` [S1, unofficial].
*   **Mida Compositing:** Some implementations note that 'mida' compositing mode may not work correctly [S3, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def vec3(x, y, z):
    return [x, y, z]

def vec4(x, y, z, w):
    return [x, y, z, w]

def length(v):
    return math.sqrt(v[0]**2 + v[1]**2 + v[2]**2)

def normalize(v):
    l = length(v)
    if l == 0:
        return vec3(0, 0, 0)
    return vec3(v[0]/l, v[1]/l, v[2]/l)

def trilinear_interpolate(volume, pos, dims):
    # Standard trilinear interpolation [G]
    vx = pos[0] * (dims[0] - 1)
    vy = pos[1] * (dims[1] - 1)
    vz = pos[2] * (dims[2] - 1)
    x0, y0, z0 = int(math.floor(vx)), int(math.floor(vy)), int(math.floor(vz))
    x1, y1, z1 = x0 + 1, y0 + 1, z0 + 1
    x0 = max(0, min(x0, dims[0]-1))
    x1 = max(0, min(x1, dims[0]-1))
    y0 = max(0, min(y0, dims[1]-1))
    y1 = max(0, min(y1, dims[1]-1))
    z0 = max(0, min(z0, dims[2]-1))
    z1 = max(0, min(z1, dims[2]-1))
    wx = vx - x0
    wy = vy - y0
    wz = vz - z0
    c00 = volume[x0][y0][z0] * (1-wx) + volume[x1][y0][z0] * wx
    c01 = volume[x0][y0][z1] * (1-wx) + volume[x1][y0][z1] * wx
    c10 = volume[x0][y1][z0] * (1-wx) + volume[x1][y1][z0] * wx
    c11 = volume[x0][y1][z1] * (1-wx) + volume[x1][y1][z1] * wx
    r0 = c00 * (1-wy) + c10 * wy
    r1 = c01 * (1-wy) + c11 * wy
    return r0 * (1-wz) + r1 * wz

def backface_rasterized_volume_rendering(volume_data, volume_dims, transfer_function, 
                                        back_face_texture, viewport_width, viewport_height, 
                                        num_steps=256, threshold=0.99):
    """
    Renders a volume using back-face rasterized raycasting.
    Assumes back_face_texture contains the entry points for each pixel.
    """
    h, w = viewport_height, viewport_width
    image = [[vec3(0.0, 0.0, 0.0) for _ in range(w)] for _ in range(h)]
    
    for y in range(h):
        for x in range(w):
            # Retrieve entry point from back-face texture
            # In a real implementation, this is done in the fragment shader
            # Here we simulate the retrieval for a specific pixel
            # Note: back_face_texture is indexed by normalized coordinates
            tex_x = x / viewport_width
            tex_y = y / viewport_height
            
            # Simulate sampling the back-face texture to get entry point
            # The texture stores the 3D position of the back face at this pixel
            entry_point = back_face_texture.sample(tex_x, tex_y)
            
            # In the actual method, the front face position is the current fragment position
            # For the purpose of this pseudocode, we assume the front face position
            # is also available or computed. However, the standard method renders
            # the front face geometry, so the vertex shader outputs the front position.
            # Since we are iterating pixels, we need the front position corresponding
            # to this pixel. In the two-pass method, the second pass renders the front face.
            # The fragment shader receives the interpolated front position.
            # For this pseudocode, we assume we have the front position for this pixel.
            # This is a simplification; in reality, this loop is the fragment shader.
            
            # Let's assume we have the front position for this pixel from the geometry
            # In the actual GPU implementation, this is `EntryPoint` varying.
            # We will simulate it by assuming the ray goes from entry to some exit.
            # But the method specifically uses the back face texture for the EXIT point
            # (or entry, depending on culling). Let's stick to the source:
            # Pass 1: Render Back Face -> Store in Texture.
            # Pass 2: Render Front Face -> Sample Texture to get Back Face Pos.
            # So, for a given pixel, we have Front Pos (from geometry) and Back Pos (from texture).
            
            # For this pseudocode, we need the Front Position.
            # Since we are simulating the fragment shader, we assume `front_point` is available.
            # In a CPU simulation, we would need to know which triangle covers this pixel.
            # This is complex. The pseudocode below represents the FRAGMENT SHADER logic.
            
            # To make it implementable from text, we define the fragment shader logic.
            pass # See below for fragment shader logic

def fragment_shader_backface_raymarch(front_point, back_point_texture, volume_data, volume_dims, 
                                      transfer_function, num_steps=256, threshold=0.99):
    """
    Simulates the fragment shader for the second pass.
    front_point: vec3, the position of the front face vertex interpolated to this fragment.
    back_point_texture: sampler2D, the texture storing back face positions.
    """
    # 1. Sample back-face texture to get exit/entry point
    # In GLSL: vec3 exitPoint = texture(u_ExitPoints, gl_FragCoord.st/u_Resolution).xyz;
    # We assume `front_point` is the current fragment's position in texture space [0,1]^3
    # The texture lookup uses normalized fragment coordinates.
    # For this simulation, we assume `back_point` is retrieved.
    # In reality, `back_point` comes from the texture sampled at (x/W, y/H).
    
    # Let's assume we have `back_point` from the texture lookup
    # back_point = back_point_texture.sample(front_point.xy) # Simplified
    
    # 2. Calculate ray direction and length
    dir_vec = vec3(back_point[0] - front_point[0], 
                   back_point[1] - front_point[1], 
                   back_point[2] - front_point[2])
    ray_length = length(dir_vec)
    
    # 3. Handle degenerate rays
    if ray_length < 1e-6:
        return vec4(0.0, 0.0, 0.0, 0.0) # Discard
    
    # 4. Compute step vector
    step_vec = vec3(dir_vec[0]/num_steps, dir_vec[1]/num_steps, dir_vec[2]/num_steps)
    
    # 5. Initialize accumulation
    accum_color = vec3(0.0, 0.0, 0.0)
    accum_alpha = 0.0
    
    # 6. Ray march
    current_pos = front_point
    
    for i in range(num_steps):
        # Sample volume
        scalar = trilinear_interpolate(volume_data, current_pos, volume_dims)
        
        # Apply transfer function
        color, opacity = transfer_function(scalar)
        
        # Compositing
        if opacity > 0.0:
            accum_color[0] += (1.0 - accum_alpha) * opacity * color[0]
            accum_color[1] += (1.0 - accum_alpha) * opacity * color[1]
            accum_color[2] += (1.0 - accum_alpha) * opacity * color[2]
            accum_alpha += (1.0 - accum_alpha) * opacity
        
        # Advance
        current_pos[0] += step_vec[0]
        current_pos[1] += step_vec[1]
        current_pos[2] += step_vec[2]
        
        # Early termination
        if accum_alpha >= threshold:
            break
            
        # Boundary check
        if not (0.0 <= current_pos[0] <= 1.0 and 
                0.0 <= current_pos[1] <= 1.0 and 
                0.0 <= current_pos[2] <= 1.0):
            break
            
    return vec4(accum_color[0], accum_color[1], accum_color[2], accum_alpha)
```

## Subtopics

### [Bounding Box Raymarching](2071-bounding-box-raymarching.md)
A GPU-based technique that renders a 3D voxel grid by casting rays through a bounding cube using a two-pass approach. It differs from Back-face rasterized volume rendering by computing ray endpoints directly from the rasterized front and back faces of the bounding box geometry in the fragment shaders, rather than storing pre-computed ray data in a dedicated texture. Choose this when you want to avoid the extra texture sampling overhead of the back-face texture.

## Operating characteristics
*   **Time Complexity:** $O(W \cdot H \cdot N)$, where $W, H$ are viewport dimensions and $N$ is the number of steps. The two-pass approach adds a constant overhead for the first pass, but reduces the number of pixels processed in the second pass to only those intersecting the volume [S2, unofficial].
*   **Memory Complexity:** $O(W \cdot H)$ for the back-face texture storage [S4, unofficial].
*   **Performance:** Efficient for large viewports with small volumes, as it avoids processing empty pixels [S2, unofficial]. Performance degrades if the volume fills the entire screen, as the overhead of the second pass is incurred for all pixels.
*   **Strengths:** Correctly handles zooming (front faces don't clip behind camera), avoids explicit ray-box intersection math in the fragment shader, restricts processing to relevant pixels [S2, S3, S4, unofficial].
*   **Limitations:** Requires two render passes, extra memory for the back-face texture, and careful handling of degenerate rays (tangent/miss) [S4, unofficial].
*   **When to use:** When rendering volumes that do not fill the entire screen, or when zooming is required.
*   **When not to use:** When memory is constrained (due to the extra texture), or when the volume always fills the screen (single-pass analytical intersection may be faster).

## Background
The method originated as a GPU optimization technique for WebGL and OpenGL volume rendering, allowing efficient raymarching by leveraging the rasterization pipeline to determine ray bounds [S2, S3, S4, unofficial]. It was developed to address the inefficiency of processing pixels outside the volume and the clipping issues associated with rendering front faces during zoom [S2, unofficial]. It is used in various open-source volume rendering engines and educational examples [S1, S3, S4, unofficial].

## Sources
[S1] levl-dev/VolumeRayCasting_0, https://github.com/levl-dev/VolumeRayCasting_0, fetched 2026-10-26 (unofficial)
[S2] Volume Rendering with WebGL, https://www.willusher.io/webgl/2019/01/13/volume-rendering-with-webgl, fetched 2026-10-26 (unofficial)
[S3] fargiolas/qvrc, https://github.com/fargiolas/qvrc, fetched 2026-10-26 (unofficial)
[S4] thePZH/VolumeRendering-OpenGL, https://github.com/thePZH/VolumeRendering-OpenGL, fetched 2026-10-26 (unofficial)
[S5] GPU-accelerated single-pass volumetric raycasting in Qt ..., https://martinopilia.com/posts/2018/09/17/volume-raycasting.html, fetched 2026-10-27 (unofficial)
