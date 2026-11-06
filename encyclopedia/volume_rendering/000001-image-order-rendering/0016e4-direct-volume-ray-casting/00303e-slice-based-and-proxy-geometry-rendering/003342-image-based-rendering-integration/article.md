# Image-based rendering integration

Also called: Image-Based Rendering with Impostors, Impostor rendering, Image-based volume rendering

## Overview
This method integrates volume rendering into a polygonal virtual environment by treating rendered volume images as image-based primitives (impostors) within the scene [S1, unofficial]. It decouples the computationally expensive volume rendering process from the real-time polygonal rendering loop, distributing the workload to a separate server [S1, unofficial]. Unlike direct ray casting which computes pixels per frame on the client, this approach renders static or slowly changing volume views as textures mapped onto quads that track the viewer's orientation [S1, unofficial].

## How it works
The method operates as a distributed client-server system where the server handles volume rendering and the client handles polygonal scene composition [S1, unofficial].

### Inputs and Outputs
*   **Inputs:** Volume dataset (scalar field), camera parameters (view direction for left and right eyes), head-tracker data (position/orientation updates).
*   **Outputs:** Two rendered images (stereo pair) sent to the client, which are displayed as textures on impostor quads.

### Data Structures
1.  **Impostor Quad:** A geometric primitive (quad) in the client's polygonal scene.
    *   **Fields:** Vertices defining the quad geometry; Texture coordinates mapping the received image; Orientation matrix/rotation.
    *   **Layout:** The quad is centered at the volume object's center. Its surface normal is rotated to always face the respective eye [S1, unofficial].
2.  **Volume Image:** The rendered output from the server.
    *   **Layout:** Standard 2D image buffer (e.g., RGBA), resolution dependent on server capability and network bandwidth.

### Steps
1.  **Server Initialization:** The server loads the volume dataset into memory. It must have sufficient memory to hold the dataset and perform rendering [S1, unofficial].
2.  **Client Request:** The client (Studierstube application) detects a change in the user's position via a head-tracker [S1, unofficial]. It sends a request to the server containing the two viewing directions (one for each eye) [S1, unofficial].
3.  **Volume Rendering:** The server renders the volume data for the specified directions. This involves the standard volume rendering pipeline (ray generation, sampling, classification, compositing) described in the parent article [S1, S4, S34].
4.  **Image Transmission:** The server sends the resulting images (stereo pair) to the client via a LAN [S1, unofficial].
5.  **Impostor Update:** The client receives the images and maps them as textures onto the impostor quads [S1, unofficial].
6.  **Orientation Adjustment:** The impostor quads are rotated around their center so their surface normal always faces the respective eye of the user [S1, unofficial].
7.  **Polygonal Rendering:** The client renders the polygonal virtual environment, including the impostor quads with the volume textures, at the standard frame rate (e.g., 60fps) [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Server Side (Running on fast PC)
def server_render_loop():
    volume_data = load_volume_dataset()
    while True:
        # Wait for request from client
        request = receive_request_from_client()
        left_view_dir = request.left_eye_direction
        right_view_dir = request.right_eye_direction
        
        # Render volume for both eyes
        # This step uses the standard volume rendering pipeline
        left_image = render_volume(volume_data, left_view_dir)
        right_image = render_volume(volume_data, right_view_dir)
        
        # Send images back to client
        send_images_to_client(left_image, right_image)

# Client Side (Running on main application)
def client_render_loop():
    impostor_left = create_impostor_quad()
    impostor_right = create_impostor_quad()
    
    while True:
        # Get current head position from tracker
        head_pos = get_head_tracker_data()
        
        # Update impostor orientation to face the eyes
        impostor_left.orientation = look_at(head_pos.left_eye, impostor_left.center)
        impostor_right.orientation = look_at(head_pos.right_eye, impostor_right.center)
        
        # If new images are available, update textures
        if images_available():
            left_img, right_img = receive_images()
            impostor_left.texture = left_img
            impostor_right.texture = right_img
            
        # Render the polygonal scene including impostors
        render_polygonal_scene()

# Asynchronous Update Thread
def update_thread():
    while True:
        # Check for head movement
        if head_moved():
            # Send request to server
            send_render_request()
            # Wait for response asynchronously
            wait_for_images()
            # Mark images as available for main loop
            mark_images_ready()
```

### Parameters and Typical Values
*   **Rendering Modes:** Surface shaded, maximum intensity projection (MIP), or non-photorealistic [S1, unofficial].
*   **Performance:** Interactive frame rates for the volume rendering server are up to 25fps for a $128^3$ resolution stereo pair, or approximately 11fps for a $256^3$ resolution stereo pair [S1, unofficial].
*   **Update Latency:** The update process (network delay + rendering) varies from 20ms to 500ms depending on complexity and rendering mode [S1, unofficial].
*   **Polygonal Frame Rate:** The virtual environment itself delivers 60fps independently of the volume rendering update rate [S1, unofficial].

### Edge Cases and Pitfalls
*   **Network Delay:** The primary problem besides rendering rate is network delay (lag), which results in a delayed response when manipulating the volume data [S1, unofficial].
*   **Quick Movements:** Decoupling image update from impostor display causes visible artifacts when the user turns the volume object quickly, as the texture may not match the new viewpoint immediately [S1, unofficial].
*   **Freezing:** The update process must happen asynchronously in a background thread to prevent the virtual environment from freezing during the rendering time [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** The volume rendering time depends on the dataset size ($N^3$) and rendering mode, typically 20ms to 500ms per stereo pair [S1, unofficial]. The polygonal rendering remains constant at high frame rates (e.g., 60fps) [S1, unofficial].
*   **Memory Complexity:** The server requires enough memory to hold the entire volume dataset and intermediate rendering buffers [S1, unofficial].
*   **Strengths:**
    *   Decouples volume rendering from the main rendering loop, maintaining responsiveness of the virtual environment [S1, unofficial].
    *   Allows the use of high-quality volume rendering algorithms on a dedicated server [S1, unofficial].
    *   Seamless integration of volume visualization into polygon-based scenes [S1, unofficial].
*   **Limitations:**
    *   Latency due to network communication and asynchronous updates [S1, unofficial].
    *   Visual artifacts during rapid head movements due to stale textures [S1, unofficial].
    *   Requires a separate server with sufficient memory and processing power [S1, unofficial].
*   **When to Use:** When integrating volume data into a larger polygonal virtual environment where real-time interaction with the polygonal scene is critical, and the volume data does not change frequently.
*   **When Not to Use:** When low-latency interaction with the volume data itself is required, or when network bandwidth is severely constrained.

## Background
This method was developed for the Studierstube virtual environment and the Real Time Volume Rendering Library (RTVR) [S1, unofficial]. It was presented at the Eurographics Symposium on Virtual Environments (EGVE) in 2005 [S1, unofficial]. The approach builds on earlier VR techniques employed in visualization settings [S1, unofficial]. It addresses the challenge of mixing volume rendering and polygonal rendering seamlessly within a virtual environment [S1, unofficial].

## Sources
[S1] Distributed Software-Based Volume Visualization in a Virtual Environment, https://doi.org/10.2312/egve/ipt_egve2005/129-140, fetched 2026-10-26 (unofficial)
