# Depth-based Clipping for Focus and Context Volume Rendering

Also called: Four-Pass Depth Clipping, Interactive Lens via Depth Clipping

## Overview
This method decomposes a volume into a high-quality 'focus' region and a lower-quality 'context' region using depth-based clipping and early-z rejection, implemented in four rendering passes to maximize fragment shader discard efficiency [S1, unofficial]. It is chosen over tessellating proxy geometry or tagging volume regions because it supports arbitrary convex clip objects and large bricked volumes without performance-complicating side effects [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A volume dataset (potentially bricked), a clip object (proxy geometry) defining the focus region, and a camera view.
*   **Output:** A rendered image where the focus region is rendered with full detail and the context region is rendered with lower quality or different transfer function settings, separated by the clip object's depth bounds.

**Data Structures:**
*   **Clip Object:** A proxy geometry (e.g., sphere, box) representing the focus region. It must have defined front-facing and back-facing polygons.
*   **Clip Texture:** A high-resolution depth component texture. It stores the depth values of the back-facing polygons of the clip object.
*   **Depth Buffer:** Standard framebuffer depth buffer.

**Parameters:**
*   **Clip Texture Resolution:** High resolution, matching the screen resolution or higher to ensure accurate depth sampling [S1, unofficial].
*   **Depth Test Modes:** `GL_LESS` and `GL_GREATER` [S1, unofficial].

**Steps:**
The rendering is performed in four passes to exploit early-z rejection. The volume slices are drawn in front-to-back order.

1.  **Pass 1: Initialize Depth Buffer with Front Faces**
    *   Set depth test to `GL_LESS`.
    *   Render the front-facing polygons of the clip object into the depth buffer.
    *   Draw all volume slices in front-to-back order. This establishes the initial depth occlusion for the front part of the volume relative to the clip object's front boundary.

2.  **Pass 2: Capture Back-Face Depth**
    *   Render the back-facing polygons of the clip object.
    *   Copy the resulting depth values from the depth buffer into the **Clip Texture**. This texture now holds the depth threshold for the back boundary of the focus region.

3.  **Pass 3: Render Front Half of Volume (Front to Center)**
    *   Render the front-facing polygons of the clip object into the depth buffer again (to reset/initialize depth for this pass).
    *   Change depth test to `GL_GREATER`.
    *   Draw volume slices from the front of the volume up to the center of the clip object.
    *   **Fragment Shader Logic:** At the beginning of the fragment shader, read the corresponding back-face depth value from the **Clip Texture**. If the fragment's depth is greater than the fetched clip texture value, terminate shader execution (discard fragment). This ensures that only fragments inside the focus region (behind the front face but in front of the back face depth) are processed.

4.  **Pass 4: Render Back Half of Volume (Center to Back)**
    *   Render the back-facing polygons of the clip object into the depth buffer.
    *   Change depth test to `GL_GREATER`.
    *   Draw volume slices from the center of the clip object to the back of the volume.
    *   **Fragment Shader Logic:** Similar to Pass 3, read the front-face depth value (now stored in the clip texture, as roles are reversed) and discard fragments if their depth is greater than the fetched value. This ensures only fragments inside the focus region are processed for the back half.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_focus_context(volume_slices, clip_object, clip_texture, depth_buffer):
    # Pass 1: Initialize depth with front faces
    set_depth_test(GL_LESS)
    render_polygons(clip_object.front_faces, depth_buffer)
    render_slices(volume_slices, order='front_to_back')

    # Pass 2: Capture back-face depth into clip texture
    render_polygons(clip_object.back_faces, depth_buffer)
    copy_depth_buffer_to_texture(depth_buffer, clip_texture)

    # Pass 3: Render front half of volume
    set_depth_test(GL_LESS) # Reset depth buffer
    render_polygons(clip_object.front_faces, depth_buffer)
    set_depth_test(GL_GREATER)
    render_slices(volume_slices, order='front_to_center')
    # Fragment shader for Pass 3:
    #   clip_depth = sample(clip_texture, fragment_uv)
    #   if fragment_depth > clip_depth: discard()

    # Pass 4: Render back half of volume
    set_depth_test(GL_LESS) # Reset depth buffer
    render_polygons(clip_object.back_faces, depth_buffer)
    set_depth_test(GL_GREATER)
    render_slices(volume_slices, order='center_to_back')
    # Fragment shader for Pass 4:
    #   clip_depth = sample(clip_texture, fragment_uv)
    #   if fragment_depth > clip_depth: discard()
```

**Edge Cases and Pitfalls:**
*   **Screen Space Lenses:** If the lens is screen-space based, the focus does not need division and a clip texture is not required [S1, unofficial]. However, opaque geometry and acceleration techniques (early ray termination/empty space skipping) require two additional depth textures: one for opaque object depths, one for the current pass's depth buffer [S1, unofficial].
*   **Transition Region:** Additional passes are needed for adding a transition region for blending the results [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Four passes over the volume slices. The fragment shader performs a texture lookup and comparison, which is relatively cheap but adds overhead per fragment.
*   **Memory Complexity:** Requires a high-resolution depth component texture (Clip Texture) and potentially additional depth textures for opaque geometry and acceleration techniques [S1, unofficial].
*   **Strengths:** Supports arbitrary convex clip objects and large bricked volumes without performance-complicating side effects [S1, unofficial]. Maximizes early-z rejection by discarding fragments early in the pipeline [S1, unofficial].
*   **Limitations:** Requires four rendering passes, which increases draw call overhead. Implementation is more complex than simple clipping. Screen-space lenses simplify the process but require additional depth management for acceleration techniques [S1, unofficial].

## Background
This method was developed as part of a focus and context interface for interactive volume rendering. It was chosen to allow varying sampling rates for different regions and to support arbitrary convex clip objects and large bricked volumes without complications that would otherwise significantly influence rendering performance [S1, unofficial].

## Sources
[S1] A Focus and Context Interface for Interactive Volume Rendering, https://www.sci.utah.edu/~ikits/publications/Ikits_Unpublished_04.pdf, fetched 2026-10-27 (unofficial)
