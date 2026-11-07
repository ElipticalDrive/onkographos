# Near Plane Clipping Correction

Also called: Camera Inside Volume Handling

## Overview
Near Plane Clipping Correction handles the artifact where the camera is located inside a fluid volume and the camera's near clipping plane intersects the front faces of the volume geometry. Without correction, the front faces are clipped, leaving no valid entry point for the ray-marching algorithm, which results in incorrect ray depths and missing volume data. It sets itself apart from standard bounding-box raycasting by explicitly detecting clipped regions via back-face rendering markers and correcting the ray origin and depth using the near plane intersection [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A RayData texture (initialized to zero), the volume geometry (front and back faces), the camera near plane distance, and the camera-to-near-plane intersection positions.
*   **Output:** A corrected RayData texture where pixels corresponding to clipped front faces contain valid entry positions and marching distances derived from the near plane.

**Data Structures:**
*   **RayData Texture:** A 2D texture storing ray parameters.
    *   **Red Channel:** Stores a large negative value to mark empty space or invalid rays [S1, unofficial].
    *   **Green Channel:** Used as a marker. A negative value indicates that the back face was rendered but the front face was clipped [S1, unofficial].
    *   **Other Channels:** Store the ray entry position and marching distance (as per the parent method [3077-volume-ray-casting-with-raydata-texture.md]).

**Steps:**
1.  **Clear RayData Texture:** Initialize the RayData texture to zero for all pixels [S1, unofficial].
2.  **Render Back Faces:** Render the back faces of the fluid volume. When writing to the RayData texture, write a negative color value into the green channel. This marks pixels where the back face is visible [S1, unofficial].
3.  **Render Front Faces:** Render the front faces of the fluid volume. Do not modify the RGB values of the destination color during this pass (using alpha blending as defined in Equation 1 of the source) [S1, unofficial].
4.  **Identify Clipped Regions:** After both passes, inspect the RayData texture. Pixels with a negative green channel indicate that the back face was rendered (step 2) but the front face was not rendered (step 3) due to near-plane clipping [S1, unofficial].
5.  **Correct Ray Origin in Shader:** In the ray-casting shader, for any pixel identified as clipped (negative green channel):
    *   Explicitly initialize the ray entry position to the texture-space position of the corresponding point on the near plane [S1, unofficial].
6.  **Correct Ray Depth:**
    *   Read the depth value from the RayData texture.
    *   Subtract the distance from the eye (camera) to the corresponding point on the near plane from this depth value [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode for the ray-casting shader fragment
def ray_cast_shader(uv):
    # Read RayData texture
    ray_data = read_ray_data(uv)
    
    # Check if this pixel was marked as clipped
    # Negative green channel indicates back face rendered, front face clipped
    if ray_data.green < 0:
        # Get the intersection point with the near plane
        # This requires computing the ray direction from the camera to uv
        # and intersecting with the near plane at distance z_near
        near_plane_pos = compute_near_plane_intersection(uv, camera_position, camera_direction, z_near)
        
        # Correct the ray entry position
        ray_entry_position = near_plane_pos
        
        # Correct the depth
        # Distance from eye to near plane point
        dist_to_near = length(near_plane_pos - camera_position)
        
        # Original depth from RayData (which is incorrect/invalid for clipped regions)
        original_depth = ray_data.depth
        
        # Subtract the distance to the near plane to get the correct remaining march distance
        # Note: The source says "subtract from it the distance... from the depth value"
        # This implies the stored depth might be absolute or relative to a different origin.
        # Assuming the RayData texture stores the total distance to the back face or similar,
        # we adjust it.
        corrected_depth = original_depth - dist_to_near
        
        return ray_entry_position, corrected_depth
    else:
        # Standard case: use data from RayData texture
        return ray_data.entry_position, ray_data.depth
```

**Parameters:**
*   **Negative Marker Value:** Any negative value in the green channel serves as the marker [S1, unofficial].
*   **Large Negative Value:** A large negative value in the red channel is used to mark empty space or rays that should not sample the volume [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** Ensure the negative marker in the green channel is distinguishable from valid positive depth/position data.
*   **Boundary Conditions:** The correction only applies to pixels where the front face is clipped. If the front face is fully visible, the standard RayData values are used.
*   **Near Plane Intersection:** Accurately computing the texture-space position of the near plane intersection is critical for correct ray origin initialization [S1, unofficial].

## Subtopics

### Viewpoint Inside Volume Ray Setup
A technique to allow the viewpoint to be inside the volume (e.g., virtual colonoscopy) by intersecting the ray with polygonal geometry instead of just a bounding box. It involves computing a starting position image, a ray length image, rendering polygonal geometry to modify the ray length image (filling holes caused by clipping), performing raycasting into a compositing buffer, and blending to the final image. A key pitfall is that the near clipping plane may clip into the front faces of the volume, creating holes that must be filled using the near clipping plane or depth buffer [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Adds two additional rendering passes (back faces and front faces) to populate the RayData texture, plus a conditional check in the ray-casting shader.
*   **Memory Complexity:** Requires a RayData texture with specific channel usage (green for marking, red for empty space markers).
*   **Strengths:** Correctly handles the complex case of a camera inside a volume, which is common in medical visualization (e.g., virtual colonoscopy) [S2, unofficial].
*   **Limitations:** Requires careful management of the RayData texture channels and additional rendering passes. The correction relies on the assumption that the back face is visible when the front face is clipped.
*   **When to Use:** When the camera can be inside the volume and the near plane may intersect the volume geometry.
*   **When Not to Use:** When the camera is always outside the volume and the near plane does not intersect the volume front faces.

## Background
The method was described in the context of real-time simulation and rendering of 3D fluids, specifically addressing artifacts caused by near-plane clipping when the camera is inside the fluid volume [S1, unofficial]. It is also discussed in the context of GPU-based volume ray-casting with advanced illumination, where it is part of a broader strategy for handling viewpoints inside the volume, such as in virtual colonoscopy [S2, unofficial].

## Sources
[S1] Real-Time Simulation and Rendering of 3D Fluids, https://wo1fsea.github.io/files/About_NavMesh/P5_GPU_Gems_3_Real-Time_Simulation_and_Rendering_of_3D_Fluids.pdf, fetched 2026-10-27 (unofficial)
[S2] GPU-Based Volume Ray-Casting with Advanced Illumination, https://doi.org/10.2312/egt.20091069, fetched 2026-10-26 (unofficial)
