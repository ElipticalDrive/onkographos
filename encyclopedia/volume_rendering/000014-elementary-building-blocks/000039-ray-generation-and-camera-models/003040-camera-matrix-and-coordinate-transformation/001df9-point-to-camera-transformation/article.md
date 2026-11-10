# Point to Camera Transformation

Also called: ptstocam

## Overview
The Point to Camera Transformation converts 3D points from world coordinates into the camera's local coordinate system [S1, unofficial]. It establishes the relative position of scene geometry with respect to the viewer, serving as a foundational step for ray generation and visibility checks [G]. Unlike [NDC Ray Transformation](905-ndc-ray-transformation.md), which maps pixels to rays, or [View Matrix Construction](1644-view-matrix-construction.md), which builds the full affine transform matrix, this method explicitly computes the rotated and translated position of individual points to facilitate downstream operations like depth sampling or projection [G].

## How it works
**Inputs:**
- $\mathbf{P}_{\text{world}}$: A 3D point in world coordinates ($\mathbb{R}^3$).
- $\mathbf{C}_{\text{c2w}}$: The camera-to-world transformation matrix (4x4) [G].

**Outputs:**
- $\mathbf{P}_{\text{cam}}$: The 3D point in camera coordinates ($\mathbb{R}^3$).

**Data Structures:**
- $\mathbf{C}_{\text{c2w}}$ is stored as a 4x4 matrix containing a 3x3 rotation submatrix $\mathbf{R}$ and a 3x1 translation vector $\mathbf{t}$ (the camera origin in world space) [G].
  $$
  \mathbf{C}_{\text{c2w}} = \begin{bmatrix} \mathbf{R} & \mathbf{t} \\ \mathbf{0}^T & 1 \end{bmatrix}
  $$
  The columns of $\mathbf{R}$ represent the camera's right, up, and forward basis vectors in world coordinates [G].

**Steps:**
1. **Extract Camera Origin:** Identify the translation component $\mathbf{t}$ from $\mathbf{C}_{\text{c2w}}$. This represents the camera's eye position $\mathbf{e}$ in world coordinates [G].
2. **Compute Relative Vector:** Calculate the vector from the camera origin to the world point:
   $$
   \mathbf{v} = \mathbf{P}_{\text{world}} - \mathbf{e}
   $$
   [S1, unofficial]
3. **Rotate to Camera Space:** Transform the relative vector $\mathbf{v}$ into the camera's local frame by multiplying by the transpose of the rotation part of $\mathbf{C}_{\text{c2w}}$:
   $$
   \mathbf{P}_{\text{cam}} = \mathbf{R}^T \mathbf{v}
   $$
   [S1, unofficial]
   Equivalently, using dot products with the basis vectors $\mathbf{r}$ (right), $\mathbf{u}$ (up), and $\mathbf{f}$ (forward) from $\mathbf{R}$:
   $$
   \begin{aligned}
   P_{\text{cam},x} &= \mathbf{r} \cdot \mathbf{v} \\
   P_{\text{cam},y} &= \mathbf{u} \cdot \mathbf{v} \\
   P_{\text{cam},z} &= \mathbf{f} \cdot \mathbf{v}
   \end{aligned}
   $$
   [G]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def point_to_camera(P_world: vec3, c2w_matrix: mat4x4) -> vec3:
    # Extract camera origin (translation part of c2w matrix)
    cam_origin = c2w_matrix[3]
    # Compute vector from camera to point
    relative_vec = P_world - cam_origin
    # Extract rotation matrix columns (top-left 3x3 of c2w)
    # Assuming column-major storage where c2w_matrix[i] is the i-th column
    R_col0 = c2w_matrix[0][:3]
    R_col1 = c2w_matrix[1][:3]
    R_col2 = c2w_matrix[2][:3]
    # Compute P_cam = R^T * relative_vec using dot products
    P_cam_x = dot(R_col0, relative_vec)
    P_cam_y = dot(R_col1, relative_vec)
    P_cam_z = dot(R_col2, relative_vec)
    return vec3(P_cam_x, P_cam_y, P_cam_z)
```
*Note: `vec3`, `mat4x4`, and `dot` are standard vector and math operations.*

**Parameters:**
- Not given in the sources.

**Edge Cases and Pitfalls:**
- **Non-orthonormal Basis:** If $\mathbf{C}_{\text{c2w}}$ contains scaling or shearing, $\mathbf{R}^T$ is not the exact inverse [G].
- **Precision:** Subtracting large world coordinates from the camera origin can lead to floating-point precision loss if the scene is very large [G].

## Subtopics

### [3D Point Projection to 2D](3148-3d-point-projection-to-2d.md)
Projects 3D points into multiple camera views to compute 2D pixel locations and visibility masks. It differs from orthographic projection by introducing depth-dependent scaling (perspective division), which is essential for realistic camera models and visibility computation in volume rendering pipelines.

### World to Clip Space Conversion
Converts a world position to camera clip space coordinates (2D), used for sampling textures or determining pixel locations in camera space [S2, unofficial].

**Steps:**
1. **Basis Construction:** Calculate camera basis vectors. Let $\mathbf{f}$ be the forward vector. Calculate $\mathbf{r} = \text{normalize}(\text{cross}(\mathbf{f}, \text{vec3}(0.0, 1.0, 0.0)))$ and $\mathbf{u} = \text{normalize}(\text{cross}(\mathbf{r}, \mathbf{f}))$ [S2, unofficial].
2. **View Space Position:** Compute the position relative to the camera: $\mathbf{v} = \mathbf{P}_{\text{world}} - \mathbf{e}$. Then compute view coordinates:
   $$
   \begin{aligned}
   \text{viewX} &= \mathbf{r} \cdot \mathbf{v} \\
   \text{viewY} &= \mathbf{u} \cdot \mathbf{v} \\
   \text{viewZ} &= \mathbf{f} \cdot \mathbf{v}
   \end{aligned}
   $$
   [S2, unofficial]
3. **Perspective Projection:** Apply perspective division factors. Let $\text{scale} = \tan(\text{radians}(\text{fov}) / 2.0)$ [S2, unofficial].
   $$
   \begin{aligned}
   \text{clipX} &= \frac{\text{viewX}}{\text{viewZ} \cdot \text{scale} \cdot \text{aspectRatio}} \\
   \text{clipY} &= \frac{\text{viewY}}{\text{viewZ} \cdot \text{scale}}
   \end{aligned}
   $$
   [S2, unofficial]
4. **Offset and Scale:** Adjust for pixel coordinates [S2, unofficial].
   $$
   \begin{aligned}
   \text{clipX} &= \text{clipX} \cdot \left(1 - \frac{1.0}{\text{screenWidth}}\right) \\
   \text{clipY} &= \text{clipY} \cdot \left(1 - \frac{1.0}{\text{screenHeight}}\right) \\
   \text{clipX} &= \frac{\text{clipX}}{2.0} + 0.5 \\
   \text{clipY} &= \frac{\text{clipY}}{2.0} + 0.5
   \end{aligned}
   $$
   [S2, unofficial]
5. **Pixel Coordinates:**
   $$
   \begin{aligned}
   \text{pixelX} &= \text{clipX} \cdot (\text{screenWidth} - 1) \\
   \text{pixelY} &= \text{clipY} \cdot (\text{screenHeight} - 1)
   \end{aligned}
   $$
   Return $\text{vec2}(\text{pixelX}, \text{pixelY})$ [S2, unofficial].

## Operating characteristics
- **Time Complexity:** $O(1)$ per point. Requires a few vector subtractions, dot products, and matrix multiplications [G].
- **Memory Complexity:** $O(1)$ auxiliary space [G].
- **Strengths:** Direct access to camera-relative coordinates is essential for depth-based sampling and occlusion queries [G].
- **Limitations:** Does not handle perspective projection or clipping directly; requires additional steps for screen-space mapping [G].
- **When to use:** When you need the exact 3D position relative to the camera for volumetric traversal or texture lookups [G].
- **When not to use:** When you need screen-space coordinates directly; use [World to Clip Space Conversion](3148-3d-point-projection-to-2d.md) or [NDC Ray Transformation](905-ndc-ray-transformation.md) instead [G].

## Background
Not given in the sources.

## Sources
[S1] bmild/nerf, https://github.com/bmild/nerf, fetched 2026-10-26 (unofficial)
[S2] jamestkiernan/webgpu-voxel-raymarching, https://github.com/jamestkiernan/webgpu-voxel-raymarching, fetched 2026-10-24 (unofficial)
