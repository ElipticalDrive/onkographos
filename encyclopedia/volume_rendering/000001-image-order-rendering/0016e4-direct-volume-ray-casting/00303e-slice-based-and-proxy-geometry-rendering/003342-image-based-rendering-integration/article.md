# Image-based rendering integration

Also called: Image-Based Rendering with Impostors, Impostor rendering, Image-based volume rendering

## Overview
This method integrates volume rendering into a polygonal virtual environment by treating rendered volume images as image-based primitives (impostors) within the scene [S1, unofficial]. It solves the problem of maintaining interactive frame rates for complex volume data in distributed or latency-sensitive VR applications by decoupling the heavy volume rendering computation from the client-side polygonal rendering loop [S1, unofficial]. Unlike [Texture-Based Volume Rendering](4707-texture-based-volume-rendering.md) which renders the volume geometry directly on the client, this approach sends pre-rendered 2D images to the client, allowing the virtual environment to remain responsive even when volume updates are delayed [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Volumetric scalar data (stored on server), user viewing directions (left and right eye), head-tracker position/orientation data.
*   **Output:** Two 2D images (stereo pair) rendered on the server, integrated into the client's polygonal scene as textured quads.

**Data Structures:**
*   **Impostor Geometry:** Two quads (one per eye) representing the volume object. The quads are centered at the volume's center.
*   **Texture Coordinates:** Standard 2D texture coordinates $(u, v) \in [0, 1]^2$ mapped to the quad surface.
*   **Network Protocol:** Asynchronous request/response for image pairs.

**Steps:**
1.  **Server Initialization:** The server (a fast PC with sufficient memory) loads the volume dataset. It initializes the Real Time Volume Rendering Library (RTVR) or equivalent software renderer [S1, unofficial].
2.  **Client Request:** The client (Studierstube application) detects user movement via a 6DoF head-tracker. It sends the current viewing directions for the left and right eyes to the server [S1, unofficial].
3.  **Volume Rendering (Server):** The server renders the volume data for the two requested viewing directions. The rendering mode (e.g., surface shaded, maximum intensity projection, non-photorealistic) is determined by the application state [S1, unofficial].
4.  **Image Transmission:** The server sends the resulting two images back to the client over the LAN [S1, unofficial].
5.  **Impostor Orientation (Client):** Upon receiving the images, the client assigns them as textures to two quad impostors. The quads are rotated around their center so that their surface normal always faces the respective eye of the user [S1, unofficial].
6.  **Scene Integration:** The textured quads are rendered as part of the standard polygonal OpenGL rendering pipeline of the virtual environment [S1, unofficial]. The size of the impostors is not varied; normal perspective projection handles the scaling [S1, unofficial].
7.  **Asynchronous Update:** The image update process runs in a background thread on the client to prevent freezing the virtual environment during the rendering and network delay (20ms to 500ms) [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Server-side rendering loop (runs on separate server)
def render_volume_server(volume_data, left_eye_dir, right_eye_dir):
    # Render volume for left eye
    left_image = rtvr.render(volume_data, left_eye_dir)
    # Render volume for right eye
    right_image = rtvr.render(volume_data, right_eye_dir)
    return left_image, right_image

# Client-side impostor management
class ImpostorManager:
    def __init__(self, volume_center):
        self.volume_center = volume_center
        self.left_quad = Quad(center=volume_center)
        self.right_quad = Quad(center=volume_center)
        self.left_texture = None
        self.right_texture = None
        self.pending_request = False

    def update(self, head_tracker_data):
        left_eye_dir = head_tracker_data.get_left_eye_direction()
        right_eye_dir = head_tracker_data.get_right_eye_direction()
        
        # Send request to server if not already pending
        if not self.pending_request:
            send_request_to_server(left_eye_dir, right_eye_dir)
            self.pending_request = True

    def on_image_received(self, left_img, right_img):
        self.left_texture = Texture(left_img)
        self.right_texture = Texture(right_img)
        self.pending_request = False

    def render(self, camera_position):
        # Rotate quads so normals face respective eyes
        # Note: In a real stereo setup, the camera position for each eye
        # is used to determine the rotation of the specific impostor.
        
        # Left eye impostor
        self.left_quad.set_texture(self.left_texture)
        self.left_quad.rotate_to_face(camera_position.left_eye)
        self.left_quad.render()
        
        # Right eye impostor
        self.right_quad.set_texture(self.right_texture)
        self.right_quad.rotate_to_face(camera_position.right_eye)
        self.right_quad.render()
```

**Parameters:**
*   **Volume Resolution:** 128^3 or 256^3 [S1, unofficial].
*   **Update Frequency:** Asynchronous, dependent on rendering complexity (20ms to 500ms per update) [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Network Latency:** The primary disadvantage is network delay (lag), which causes a delayed response when manipulating volume data [S1, unofficial].
*   **Rapid Rotation:** Turning the volume object quickly reveals the decoupling between image update and impostor display, as the static images may not match the new viewpoint immediately [S1, unofficial].
*   **Freezing:** If the asynchronous update mechanism fails, the virtual environment may freeze during the 20ms–500ms rendering time [S1, unofficial].

## Operating characteristics
*   **Performance:** Interactive frame rates for the volume rendering component are up to 25fps for a 128^3 resolution and approximately 11fps for a 256^3 resolution, depending on complexity and rendering modes (surface shaded, MIP, etc.) [S1, unofficial].
*   **Virtual Environment Performance:** The polygonal virtual environment itself delivers independently at 60fps, unaffected by the volume rendering load [S1, unofficial].
*   **Memory:** Requires sufficient memory on the server to hold the volume datasets [S1, unofficial].
*   **Strengths:** Decouples volume rendering from the client, allowing for complex volume visualization in VR without compromising the responsiveness of the polygonal scene [S1, unofficial]. Scalable via distributed architecture [S1, unofficial].
*   **Limitations:** Susceptible to network latency and lag, especially during rapid user movement or volume manipulation [S1, unofficial]. Not suitable for real-time interaction where immediate visual feedback is critical for every frame [S1, unofficial].

## Background
This method was developed by A.L. Fuhrmann et al. and presented at Eurographics 2005 [S1, unofficial]. It builds on previous VR techniques employed in visualization settings [FLS97, FG98] and utilizes the Real Time Volume Rendering Library (RTVR) and the Studierstube virtual environment [SFH*02]. The approach addresses the limitations of hardware-dependent volume rendering (like SGI’s Volumizer) by providing a software-based, distributed solution for PC platforms [S1, unofficial].

## Sources
[S1] Distributed Software-Based Volume Visualization in a Virtual Environment, https://doi.org/10.2312/egve/ipt_egve2005/129-140, fetched 2026-10-26 (unofficial)
