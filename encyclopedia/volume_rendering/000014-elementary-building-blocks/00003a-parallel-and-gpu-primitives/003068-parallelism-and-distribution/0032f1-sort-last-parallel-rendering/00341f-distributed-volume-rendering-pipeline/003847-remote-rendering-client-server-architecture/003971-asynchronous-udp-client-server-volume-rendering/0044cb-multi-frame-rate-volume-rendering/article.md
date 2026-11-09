# Multi-Frame Rate Volume Rendering

Also called: Asynchronous Multi-GPU Volume Rendering, Fast/Slow GPU Volume Rendering

## Overview
Multi-frame rate volume rendering is an asynchronous parallel rendering technique that distributes workload across multiple GPUs to balance high frame rates with high visual quality [S1]. A fast display GPU handles user input and low-latency feedback, while one or more backend GPUs asynchronously compute high-quality views [S1]. It distinguishes itself from earlier multi-frame rate systems by using layered image warping to preserve motion parallax and enable view-dependent lighting for transparent volumetric structures, rather than reconstructing the scene as a single surface [S1].

## How it works
The method operates on a client-server architecture where the "fast node" (client) manages display and interaction, and the "slow node" (server) performs heavy raycasting [S1].

**Inputs and Outputs**
*   **Input:** Volumetric dataset, camera parameters, user input (mouse/keyboard).
*   **Output:** Final composited image displayed on the fast node's monitor.

**Data Structures**
*   **Ring Buffer:** Process-local heap memory on the host, used as an intermediate storage for images transferred from the slow node to the fast node [S1].
*   **G-Buffers:** Stored on the fast node GPU. Each G-Buffer stores three channels: Color, Normal, and Depth [S1]. Multiple G-Buffers may exist to hold reference frames or intermediate warping results [S1].

**Pipeline Steps**
The rendering pipeline executes five main steps in order [S1]:

1.  **Raycasting (Slow Node):** The slow node GPU performs volume raycasting. It produces three buffers: Color, Depth, and Normal [S1]. The raycaster supports complex translucent objects and CSG intersections [S1].
2.  **Transfer:** The image buffers are downloaded from the slow node's GPU memory to the host memory (ring buffer) and then uploaded to the fast node's GPU memory [S1].
3.  **Latency Compensation (Warping):** The fast node warps the reference frames (stored in G-Buffers) to compensate for camera movement since the last raycast [S1]. This involves projecting the depth and normal information to estimate new pixel positions and colors [S1].
4.  **Shading:** View-dependent lighting is computed on the warped data [S1]. This step is critical for transparent objects to correctly light inner structures revealed by motion parallax [S1].
5.  **Compositing:** Fragments are sorted per pixel, blended front-to-back, and written to the frame buffer [S1].

**Pseudocode**
The following pseudocode outlines the asynchronous loop on the fast node and the raycasting loop on the slow node.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Fast Node (Client) State
g_buffers = [] # List of G-Buffer objects (Color, Normal, Depth textures)
ring_buffer = RingBuffer() # Host memory buffer for incoming images
current_camera = Camera()
last_warped_g_buffer = None

def fast_node_loop():
    while True:
        # 1. Process User Input
        user_input = get_user_input()
        if user_input:
            current_camera.update(user_input)
        
        # 2. Check for new frames from slow node
        if ring_buffer.has_new_frame():
            # Download from host ring buffer to fast node GPU memory
            new_color, new_depth, new_normal = ring_buffer.read()
            
            # Store as a new G-Buffer reference frame
            ref_g_buffer = create_g_buffer(new_color, new_depth, new_normal)
            g_buffers.append(ref_g_buffer)
            
            # Limit buffer size to prevent memory overflow
            if len(g_buffers) > MAX_BUFFERS:
                g_buffers.pop(0)

        # 3. Latency Compensation (Warping)
        # Warp the most recent reference frame(s) to current camera view
        if g_buffers:
            # Select best reference frame (e.g., most recent)
            ref = g_buffers[-1]
            
            # Warp reference G-Buffer to current camera
            # Uses depth and normal to estimate occlusion and displacement
            warped_g_buffer = warp_g_buffer(ref, current_camera)
            
            # 4. Shading
            # Compute lighting based on warped normals and view direction
            shaded_g_buffer = shade(warped_g_buffer, current_camera)
            
            # 5. Compositing
            # If multiple layers exist, composite them
            final_image = composite(shaded_g_buffer)
            
            # Display
            display(final_image)
        else:
            # Fallback: display previous frame or black
            if last_warped_g_buffer:
                display(last_warped_g_buffer)

def warp_g_buffer(g_buffer, camera):
    """
    Warps a G-Buffer (Color, Normal, Depth) from a reference camera 
    to the current camera using depth and normal information.
    Returns a new G-Buffer.
    """
    new_color = empty_texture()
    new_normal = empty_texture()
    new_depth = empty_texture()
    
    width, height = g_buffer.width, g_buffer.height
    
    for y in range(height):
        for x in range(width):
            # Get reference pixel data
            ref_color = g_buffer.color[x, y]
            ref_normal = g_buffer.normal[x, y]
            ref_depth = g_buffer.depth[x, y]
            
            # Project reference point to current camera
            # P_ref = unproject(ref_depth, ref_normal, ref_camera)
            # P_proj = project(P_ref, camera)
            
            # Handle occlusion: if the projected point is occluded by 
            # closer geometry in the current view, mark as invalid
            if is_occluded(P_proj, camera):
                continue # Skip this pixel
            
            # Sample or interpolate color and normal at projected location
            new_color[P_proj.x, P_proj.y] = ref_color
            new_normal[P_proj.x, P_proj.y] = ref_normal
            new_depth[P_proj.x, P_proj.y] = P_proj.z
            
    return create_g_buffer(new_color, new_normal, new_depth)

def shade(g_buffer, camera):
    """
    Applies view-dependent lighting to the G-Buffer.
    """
    shaded_color = empty_texture()
    for y in range(g_buffer.height):
        for x in range(g_buffer.width):
            normal = g_buffer.normal[x, y]
            view_dir = get_view_direction(x, y, camera)
            # Simple diffuse/specular lighting model
            light_intensity = compute_lighting(normal, view_dir)
            shaded_color[x, y] = g_buffer.color[x, y] * light_intensity
    return create_g_buffer(shaded_color, g_buffer.normal, g_buffer.depth)

# Slow Node (Server) State
volume_data = LoadVolume("dataset.vti")

def slow_node_loop():
    while True:
        # 1. Raycasting
        # Render volume to color, depth, and normal buffers
        color, depth, normal = raycast(volume_data, current_camera)
        
        # 2. Transfer to Host Memory (Ring Buffer)
        ring_buffer.write(color, depth, normal)
        
        # Optional: Sleep or wait for signal to control frame rate
```

**Parameters**
*   **Ring Buffer Size:** Not given in the sources. Must be large enough to hold at least one full frame of color, depth, and normal data [S1].
*   **Layer Separation:** The system uses feature peeling to separate layers for transparent objects, adapting to dataset features rather than using regular intervals [S1].

**Edge Cases and Pitfalls**
*   **Data Transfer Bottleneck:** Transfer through host memory is a major performance limitation, especially when the slow node's frame rate approaches the fast node's frame rate [S1].
*   **Economic Viability:** Due to transfer costs, the system is only economic for large rendering problems until fast inter-GPU links are available [S1].
*   **Transparent Objects:** Previous systems failed to preserve motion parallax for transparent objects by using a single surface representation. This method avoids that by warping layered G-Buffers [S1].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Scales by adding backend GPUs for higher quality or resolution, while the fast node maintains display latency [S1].
*   **Strengths:** High frame rates and low latency for user input; preserves motion parallax and view-dependent lighting for transparent volumes [S1].
*   **Limitations:** Performance limited by host memory bandwidth for inter-GPU transfer; requires large rendering problems to be economic [S1].
*   **When to use:** Medical visualization or scientific visualization of transparent volumetric data where user interaction latency is critical but high-quality rendering is also required [S1].
*   **When not to use:** Small datasets where single-GPU rendering is sufficient, or when fast inter-GPU links are not available and transfer overhead dominates [S1].

## Background
Multi-frame rate volume rendering was introduced by Stefan Hauswiesner, Denis Kalkofen, and Dieter Schmalstieg from Graz University of Technology [S1]. It was presented in the context of the Eurographics Symposium on Parallel Graphics and Visualization (EGPGV) in 2010 [S1]. The work addresses limitations in previous multi-frame rate systems that used single-surface reconstruction, which destroyed motion parallax for transparent objects [S1].

## Sources
[S1] Multi-Frame Rate Volume Rendering, https://doi.org/10.2312/egpgv.egpgv10.019-026, fetched 2026-10-29
