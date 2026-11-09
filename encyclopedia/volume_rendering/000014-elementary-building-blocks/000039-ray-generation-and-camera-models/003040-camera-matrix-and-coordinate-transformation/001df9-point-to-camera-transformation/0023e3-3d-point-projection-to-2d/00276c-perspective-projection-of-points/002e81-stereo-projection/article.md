# Stereo Projection
## Overview
Stereo projection applies a perspective projection twice with adjustments to simulate the viewpoint for an observer's left and right eye, creating a stereo effect [S1, unofficial]. It generates two correlated view frustums rather than one [S1, unofficial].
## How it works
**Inputs and Outputs:**
*   **Input:** 3D scene geometry, a base camera position, and the interpupillary distance (IPD).
*   **Output:** Two 2D image buffers (left eye and right eye).

**Data Structures:**
The method relies on two view frustums. Each frustum is defined by an eye position, a view direction, and projection parameters (field of view, aspect ratio, near/far clipping planes) [G].

**Steps:**
1.  **Calculate Eye Positions:** Offset the central camera position horizontally by half the IPD to determine the left and right eye origins [G].
2.  **Define Frustums:** Construct two perspective projection matrices corresponding to each eye's position and view direction [G].
3.  **Transform Geometry:** Transform all 3D scene vertices into the clip space of the left eye frustum.
4.  **Render Left Image:** Rasterize the transformed geometry to produce the left eye image.
5.  **Transform Geometry:** Transform all 3D scene vertices into the clip space of the right eye frustum.
6.  **Render Right Image:** Rasterize the transformed geometry to produce the right eye image.

The two resulting versions of the scene can be drawn in succession using shuttered glasses synced with the monitor, or drawn simultaneously using stereo viewers [S1, unofficial].

**Formulas:**
Not given in the sources. Standard perspective projection matrices are used, adjusted for each eye's specific position and orientation [G].

**Parameters:**
*   **IPD:** Typical human value is ~63–65 mm [G].
*   **Convergence Distance:** The distance at which the left and right view directions intersect [G].

**Edge Cases and Pitfalls:**
*   **Vertical Parallax:** Occurs if the projection planes of the left and right eyes do not coincide, causing visual discomfort [G].
*   **Clipping:** Geometry may be clipped differently by the two frustums, potentially causing popping artifacts if not handled carefully [G].
*   **Precision:** Floating-point precision errors in eye position calculation can lead to misalignment [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def perspective_matrix(fov_deg, aspect, near, far):
    fov_rad = math.radians(fov_deg)
    f = 1.0 / math.tan(fov_rad / 2.0)
    return [[f / aspect, 0, 0, 0],
            [0, f, 0, 0],
            [0, 0, (far + near) / (near - far), -1],
            [0, 0, (2 * far * near) / (near - far), 0]]

def transform_and_rasterize(vertices, proj_matrix):
    projected = []
    for v in vertices:
        v_h = v + [1.0]
        v_proj = [0, 0, 0, 0]
        for i in range(4):
            v_proj[i] = sum(proj_matrix[i][j] * v_h[j] for j in range(4))
        if v_proj[3] != 0:
            projected.append([c / v_proj[3] for c in v_proj[:3]])
    # Rasterization step: convert projected vertices to screen pixels
    # Implementation depends on the graphics pipeline [G]
    return projected

def render_stereo(scene_vertices, camera_center, ipd, fov, aspect, near, far):
    eye_offset = ipd / 2.0
    left_eye_pos = [camera_center[0] - eye_offset, camera_center[1], camera_center[2]]
    right_eye_pos = [camera_center[0] + eye_offset, camera_center[1], camera_center[2]]
    
    left_proj = perspective_matrix(fov, aspect, near, far)
    right_proj = perspective_matrix(fov, aspect, near, far)
    
    left_image = transform_and_rasterize(scene_vertices, left_proj)
    right_image = transform_and_rasterize(scene_vertices, right_proj)
    
    return left_image, right_image
```
## Subtopics

### [Off-axis Stereoscopic Rendering](6787-off-axis-stereoscopic-rendering.md)
Uses non-symmetric camera frustums to generate coincident projection planes for the left and right eyes, preventing vertical parallax. It aligns the central viewing directions of both cameras in parallel while focusing on different points, ensuring a natural and comfortable stereo effect. Choose this when vertical parallax must be avoided, such as in CAVE systems or when the screen is not centered between the eyes.

## Operating characteristics
*   **Time Complexity:** Double the rendering time of a single perspective view, as two full passes are required [G].
*   **Memory Complexity:** Requires storage for two image buffers (or one interleaved buffer) [G].
*   **Performance:** Dependent on the complexity of the scene and the efficiency of the rasterization pipeline [G].
*   **Strengths:** Provides true depth perception for stereoscopic displays [G].
*   **Limitations:** Requires specialized hardware (shutter glasses, stereo viewers) or post-processing. Vertical parallax can cause discomfort if not managed [G].
*   **When to Use:** When stereoscopic depth is required for visualization or interaction [G].
*   **When Not to Use:** When hardware support is unavailable or when performance is critical and depth is not necessary [G].
## Background
Stereo projection is a fundamental technique in computer graphics for creating stereoscopic 3D images. It is based on the principle of binocular vision, where each eye sees a slightly different perspective of the world [G]. The technique is widely used in virtual reality, 3D movies, and scientific visualization [G]. The specific implementation described here is referenced in the context of the Smokeview tool for visualizing fire dynamics simulation data [S1, unofficial].
## Sources
[S1] Smokeview (Version 6) A Tool for Visualizing Fire Dynamics Simulation Data Volume II : Technical Reference Guide, https://doi.org/10.6028/nist.sp.1017-2, fetched 2026-10-26 (unofficial)
