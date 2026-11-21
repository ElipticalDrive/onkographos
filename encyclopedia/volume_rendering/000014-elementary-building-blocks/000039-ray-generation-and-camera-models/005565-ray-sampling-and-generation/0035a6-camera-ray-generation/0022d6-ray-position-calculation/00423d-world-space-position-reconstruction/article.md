# World Space Position Reconstruction

Also called: Coordinate system transformations, Screen-to-World Ray Reconstruction, World position reconstruction

## Overview
World Space Position Reconstruction is a method to reconstruct 3D world-space positions from 2D screen-space UV coordinates and depth values, enabling raymarching in post-processing by converting screen space to world space [S1, S2, unofficial]. It differs from direct ray generation by first establishing the precise 3D location of the current fragment in the world, which serves as the origin for subsequent raymarching steps [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:**
    *   `uv`: UV coordinate of the current fragment in the post-processing pass [S1, unofficial].
    *   `depth`: Depth value of the underlying scene sampled at `uv` [S1, unofficial].
    *   `projectionMatrixInverse`: Inverse of the camera projection matrix [S1, unofficial].
    *   `viewMatrixInverse`: Inverse of the view matrix (equivalent to `camera.matrixWorld`) [S1, unofficial].
    *   `CameraPosition`: The 3D position of the camera [S2, unofficial].
    *   `ViewDir`: The view direction vector for the current pixel [S2, unofficial].
*   **Outputs:**
    *   `worldSpace`: The reconstructed 3D position in world coordinates [S1, unofficial].

**Data Structures:**
*   **Clip Space Vector:** A 4D vector `{xNDC, yNDC, zNDC, 1.0}` [S1, unofficial].
*   **View Space Vector:** A 4D vector resulting from matrix multiplication [S1, unofficial].
*   **World Space Vector:** A 4D vector resulting from matrix multiplication, requiring perspective division [S1, unofficial].
*   **Depth Texture Index:** An integer 3-vector (`int3`) constructed from the pixel's screen-space `x` and `y` coordinates and a `z` of 0, used for direct texture loading [S2, unofficial].

**Steps:**
1.  **Convert UV to NDC:** Calculate Normalized Device Coordinates (NDC) for x and y.
    $$x_{NDC} = uv.x \times 2.0 - 1.0$$
    $$y_{NDC} = uv.y \times 2.0 - 1.0$$
    [S1, unofficial]
2.  **Convert Depth to NDC Z:** Transform the linear or depth-buffer value to NDC Z.
    $$z_{NDC} = depth \times 2.0 - 1.0$$
    [S1, unofficial]
3.  **Construct Clip Space Vector:** Assemble the 4D clip-space vector.
    $$clipSpace = \{x_{NDC}, y_{NDC}, z_{NDC}, 1.0\}$$
    [S1, unofficial]
4.  **Transform to View Space:** Multiply the clip-space vector by the inverse projection matrix.
    $$view = projectionMatrixInverse \times clipSpace$$
    [S1, unofficial]
5.  **Transform to World Space:** Multiply the view-space vector by the inverse view matrix.
    $$world = viewMatrixInverse \times view$$
    [S1, unofficial]
6.  **Perspective Divide:** Divide the x, y, and z components by the w component to obtain the final world position.
    $$worldSpace = \frac{world.xyz}{world.w}$$
    [S1, unofficial]

**Alternative Linear Depth Method:**
If linear depth is available, the position can be calculated directly without matrix inversion for the Z component, using the view direction and camera position [S2, unofficial]:
$$Position_{WS} = Depth_{linear} \times ViewDir + CameraPosition$$
[S2, unofficial]

**Parameters:**
*   `cameraFar`: The depth limit beyond which nothing is visible [S1, unofficial].
*   `lightDirection`: A normalized vector representing the direction the light points toward [S1, unofficial].
*   `lightPosition`: A vector representing the position of the light [S1, unofficial].
*   `coneAngle`: Represents the width of the spotlight aperture [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Texture Filtering:** Hardware texture filtering must not be used for depth sampling in this context because it interpolates across edges of different blocks, leading to incorrect depth values. Point filtering (e.g., `Texture.Load`) must be used instead [S2, unofficial].
*   **Precision:** Standard floating-point precision is assumed for matrix operations. Degenerate inputs (e.g., `world.w` being zero) should be handled, though not explicitly detailed in sources [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def get_world_position(uv: vec2, depth: float, projection_matrix_inverse: mat4, view_matrix_inverse: mat4) -> vec3:
    # Step 1 & 2: Convert to NDC
    clip_z = depth * 2.0 - 1.0
    ndc = uv * 2.0 - 1.0
    
    # Step 3: Construct clip space vector
    clip = vec4(ndc.x, ndc.y, clip_z, 1.0)
    
    # Step 4: Transform to view space
    view = projection_matrix_inverse * clip
    
    # Step 5: Transform to world space
    world = view_matrix_inverse * view
    
    # Step 6: Perspective divide
    return world.xyz / world.w
```

## Operating characteristics
*   **Time Complexity:** O(1) per pixel, involving a fixed number of arithmetic operations and two matrix-vector multiplications.
*   **Memory Complexity:** O(1) auxiliary memory per pixel, requiring storage for intermediate vectors.
*   **Strengths:** Enables post-processing volumetric effects by accurately placing ray origins in world space without requiring geometry traversal.
*   **Limitations:** Requires access to the inverse projection and view matrices, as well as a valid depth buffer. Accuracy is limited by the precision of the depth buffer and the inverse matrix calculations.
*   **When to use:** When implementing post-processing volumetric lighting or raymarching effects that need to interact with scene geometry depth.
*   **When not to use:** When forward rendering geometry directly or when depth information is unavailable.

## Background
This method is fundamental to real-time volumetric lighting techniques that rely on post-processing. It allows the conversion of screen-space coordinates into world-space coordinates, facilitating the calculation of light rays originating from the camera or light sources [S1, unofficial]. The technique is commonly used in shaders for volumetric fog, god rays, and other atmospheric effects.

## Sources
[S1] On Shaping Light: Real-Time Volumetric Lighting with Post-Processing ..., https://blog.maximeheckel.com/posts/shaping-light-volumetric-lighting-with-post-processing-and-raymarching/, fetched 2026-10-28 (unofficial)
[S2] Signed Distance Field Rendering Journey pt.1 – kosmonaut's blog, https://kosmonautblog.wordpress.com/2017/05/01/signed-distance-field-rendering-journey-pt-1/, fetched 2026-10-29 (unofficial)
