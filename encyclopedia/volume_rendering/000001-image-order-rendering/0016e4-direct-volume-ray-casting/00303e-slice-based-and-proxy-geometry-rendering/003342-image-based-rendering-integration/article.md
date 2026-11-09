# Image-based rendering integration

Also called: Image-Based Rendering with Impostors, Impostor rendering, Image-based volume rendering

## Overview
This method integrates volume rendering into a polygonal virtual environment by treating rendered volume images as image-based primitives (impostors) within the scene [S1, unofficial]. It solves the problem of coupling high-latency volume rendering with real-time interactive navigation by decoupling the rendering workload from the client's display loop. Unlike sibling methods that render volume geometry directly on the client, this approach distributes the heavy computation to a separate server and streams the resulting images to the client, where they are mapped onto simple geometric proxies [S1, unofficial].

## How it works
**Inputs and outputs:**
- **Input:** 3D volume dataset (stored on server), user viewing directions (left and right eye for stereoscopy), head-tracker position updates.
- **Output:** Stereo pair of images rendered on client-side quads, composited into the polygonal virtual environment.

**Data structures:**
- **Impostor Geometry:** Two quads per volume object, one for each eye. Each quad is defined by four vertices.
- **Texture Coordinates:** Standard 2D texture coordinates $(s, t)$ mapping the quad surface to the rendered image.
- **Network Protocol:** Asynchronous message passing for viewing direction requests and image data transfer.

**Steps:**
1. **Server Initialization:** The server loads the volume dataset into memory. It must have sufficient RAM to hold the volume data sets [S1, unofficial].
2. **Client Request:** The client (running in the virtual environment) detects a change in the user's head position via a 6DoF head-tracker [S1, unofficial]. It calculates the viewing direction for the left eye and the right eye. It sends these two viewing directions to the server via a LAN connection [S1, unofficial].
3. **Volume Rendering:** The server receives the two viewing directions. It renders the volume data for both directions, producing a stereo pair of images [S1, unofficial]. The rendering mode (e.g., surface shaded, maximum intensity projection, non-photorealistic) affects the computation time [S1, unofficial].
4. **Image Transmission:** The server sends the two rendered images back to the client over the LAN [S1, unofficial].
5. **Impostor Orientation:** On the client, the two impostor quads are rotated around their center so that their surface normal always faces the respective eye of the user [S1, unofficial]. This ensures the image appears correctly oriented in 3D space regardless of the user's head orientation.
6. **Texture Mapping:** The received images are used as textures on the respective quads. The size of the impostors does not need to be varied, as standard perspective rendering of the quads handles the scaling [S1, unofficial].
7. **Asynchronous Update:** The entire update process (steps 2-6) happens asynchronously in a background thread on the client. This prevents the virtual environment from freezing during the rendering time, which can vary from 20ms to 500ms depending on data complexity and rendering mode [S1, unofficial]. The environment continues to update according to user interactions, and graphical feedback (e.g., pen cursor) moves smoothly [S1, unofficial].

**Parameters:**
- **Network Latency:** A critical factor. High network delay results in delayed response when manipulating the volume data [S1, unofficial].
- **Rendering Mode:** Affects frame rate. Modes include surface shaded, maximum intensity projection, or non-photorealistic [S1, unofficial].

**Edge cases and pitfalls:**
- **Network Lag:** The main problem besides rendering rate is network delay, which causes a delayed response when manipulating the volume data [S1, unofficial].
- **Quick Turns:** Decoupling image update from impostor display becomes visible when turning the volume object quickly, as the image may not update fast enough to match the new orientation, causing a visual disconnect [S1, unofficial].
- **Freezing:** If the asynchronous update fails or blocks, the virtual environment may freeze. The background thread design mitigates this [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Server Side
class VolumeServer:
    def __init__(self, volume_data):
        self.volume_data = volume_data
        self.rendering_mode = "surface_shaded" # Example mode

    def render_stereo_pair(self, left_view_dir, right_view_dir):
        # Render volume for left eye
        left_image = render_volume(self.volume_data, left_view_dir, self.rendering_mode)
        # Render volume for right eye
        right_image = render_volume(self.volume_data, right_view_dir, self.rendering_mode)
        return left_image, right_image

# Client Side
class VirtualEnvironmentClient:
    def __init__(self, server_address):
        self.server = connect_to_server(server_address)
        self.head_tracker = HeadTracker()
        self.impostor_left = QuadImpostor()
        self.impostor_right = QuadImpostor()
        self.background_thread = Thread(target=self.update_loop)
        self.background_thread.start()

    def update_loop(self):
        while True:
            # Get current head position and orientation
            head_pos, head_orient = self.head_tracker.get_state()
            
            # Calculate viewing directions for left and right eye
            left_view_dir = calculate_view_direction(head_pos, head_orient, eye_offset_left)
            right_view_dir = calculate_view_direction(head_pos, head_orient, eye_offset_right)
            
            # Send request to server
            left_image, right_image = self.server.render_stereo_pair(left_view_dir, right_view_dir)
            
            # Update impostor orientation
            self.impostor_left.rotate_to_face_eye(head_pos, head_orient, eye_offset_left)
            self.impostor_right.rotate_to_face_eye(head_pos, head_orient, eye_offset_right)
            
            # Apply textures to impostors
            self.impostor_left.set_texture(left_image)
            self.impostor_right.set_texture(right_image)
            
            # Note: This loop runs asynchronously to avoid freezing the main render loop
            # The actual rendering of the quads happens in the main loop using the latest textures

    def render_frame(self):
        # Render polygonal environment
        render_polygonal_scene()
        # Render impostors with latest textures
        self.impostor_left.render()
        self.impostor_right.render()
```

## Subtopics

### [2D Visualization from 3D Grid (Slow)](1507-2d-visualization-from-3d-grid-slow.md)
Renders 2D slices directly from 3D grid data on the client. Slower for complex volumes but avoids network latency.

### [Hybrid Ray Tracing](2308-hybrid-ray-tracing.md)
Combines ray tracing with other techniques for higher quality but higher computational cost.

### [Surface Rendering Pipeline](2614-surface-rendering-pipeline.md)
Extracts surfaces (e.g., isosurfaces) and renders them as polygons. Good for sharp boundaries but loses internal structure.

### [Mixed Geometry Rendering](3442-mixed-geometry-rendering.md)
Combines different rendering techniques (e.g., ray casting and polygonal) in one scene.

### [Texture-Based Volume Rendering](4707-texture-based-volume-rendering.md)
Uses 3D textures and hardware rasterization on the client. No network dependency but limited by client GPU memory.

### [View-Aligned Slicing](6530-view-aligned-slicing.md)
Renders slices aligned with the view direction. Reduces artifacts but requires frequent geometry updates.

### [Vector Icon Volume Rendering](7693-vector-icon-volume-rendering.md)
Renders vector fields using icons. Specialized for vector data.

## Operating characteristics
- **Time Complexity:** The rendering time on the server depends on the volume data complexity and the rendering mode. Frame rates of up to 25fps for a resolution of $128^3$ and approximately 11fps for $256^3$ are achievable [S1, unofficial]. The client-side update is asynchronous and non-blocking, so the virtual environment maintains 60fps independently of the volume rendering rate [S1, unofficial].
- **Memory Complexity:** The server must hold the entire volume dataset in memory [S1, unofficial]. The client only needs to store the current stereo pair of images.
- **Measured Performance:** Interactive frame (stereo pair) rates of up to 25fps for $128^3$ resolution and ~11fps for $256^3$ resolution, depending on complexity and rendering modes [S1, unofficial]. The virtual environment itself delivers 60fps independently [S1, unofficial].
- **Scaling:** The system scales by distributing the rendering workload to a separate server. This adds flexibility and scalability to the system [S1, unofficial]. However, it introduces network latency, which can become a bottleneck.
- **Strengths:** Decouples volume rendering from the client's display loop, allowing the virtual environment to remain responsive. Allows the use of powerful servers for rendering.
- **Limitations:** Network latency causes delayed response when manipulating the volume data. Quick turns can cause visual disconnects as the image updates asynchronously. Requires a LAN connection and a separate server.
- **When to use:** When the volume data is too large for the client's GPU memory, or when the client hardware is insufficient for real-time volume rendering. Suitable for distributed virtual environments.
- **When not to use:** When low latency is critical, or when the network connection is unreliable or slow.

## Background
This method was introduced in the context of integrating volume visualization into virtual environments, specifically using the Studierstube virtual environment and the Real Time Volume Rendering Library (RTVR) [S1, unofficial]. It builds on earlier VR techniques applied in visualization settings [S1, unofficial]. The approach was developed to address the limitations of using texture-mapping capabilities of 3D rendering hardware (like SGI's volumizer) on PC platforms, by distributing the rendering to a separate server [S1, unofficial].

## Sources
[S1] Distributed Software-Based Volume Visualization in a Virtual Environment, https://doi.org/10.2312/egve/ipt_egve2005/129-140, fetched 2026-10-26 (unofficial)
