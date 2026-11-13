# General NDC Ray Conversion

Also called: General NDC, NDC Ray Conversion, NDC Rays, NDC projection, Normalized Device Coordinate rays

## Overview
General NDC Ray Conversion transforms camera-space rays into Normalized Device Coordinates (NDC) space, mapping the viewing frustum to a canonical cube (typically $[-1, 1]^3$ or $[0, 1]^3$) [S1, S4, S6, unofficial]. This technique fixes the near and far clipping planes to constant values (e.g., 0 and 1), removing the need for scene-dependent depth bounds and enabling uniform sampling strategies for forward-facing scenes [S2, S4, unofficial]. It differs from [Pinhole Camera Projection](3162-pinhole-camera-projection-pixel-to-camera.md) by producing a full ray for traversal rather than a single point, and from [Ray Generation from Pixel Coordinates](2514-ray-generation-from-pixel-coordinates.md) by specifically handling the normalization of depth bounds rather than just generating world-space rays [S4, S7, unofficial].

## How it works
**Inputs:**
- `rays_o`: Ray origins in camera space, shape `(batch_size, 3)` or `(H, W, 3)`.
- `rays_d`: Ray directions in camera space, shape `(batch_size, 3)` or `(H, W, 3)`.
- `focal`: Focal length of the pinhole camera (float).
- `W`: Image width in pixels (int).
- `H`: Image height in pixels (int).
- `near`: Near plane distance along the negative z-axis (float). Default is 1.0 [S1, S4, unofficial].

**Outputs:**
- `rays_o_ndc`: Transformed origins in NDC space, shape `(batch_size, 3)` or `(H, W, 3)`.
- `rays_d_ndc`: Transformed directions in NDC space, shape `(batch_size, 3)` or `(H, W, 3)`.

**Data Structures:**
The inputs and outputs are dense floating-point arrays. The z-component of the origin in NDC space maps the near plane to a constant value (typically 1 or 0 depending on convention) and infinity to another constant (typically 3 or 1). The direction vectors are not unit length [S1, unofficial].

**Steps:**
1.  **Shift Origins to Near Plane:**
    Calculate the shift factor $t$ to move the ray origin to the near plane ($z = -near$ in standard camera coordinates, though implementations often assume the camera looks down the negative z-axis and the near plane is at $z = -near$ or $z = near$ depending on the coordinate system handedness. The sources consistently use a shift that places the origin at $z = -near$ relative to the camera center if the camera looks along -z, or simply ensures the projected z-coordinate corresponds to the near plane).
    
    Using the convention where the camera looks along the negative z-axis and the near plane is at distance `near`:
    $$ t = -\frac{near + rays\_o[..., 2]}{rays\_d[..., 2]} $$
    Update origins:
    $$ rays\_o = rays\_o + t[..., \text{None}] \cdot rays\_d $$
    
    *Note: Some implementations (e.g., [S8, unofficial]) use $t = \frac{near - rays\_o[..., 2]}{rays\_d[..., 2]}$. This discrepancy arises from sign conventions in the camera coordinate system (OpenGL vs. DirectX/Blender). The sources [S1, S4, S6, unofficial] use the negative sign convention consistent with OpenGL-style cameras where the camera looks down -z.* [S1, S4, S6, S8, unofficial].

2.  **Project Origins to NDC:**
    Compute the NDC coordinates for the shifted origins. The x and y components are scaled by the focal length and image dimensions. The z component maps the depth to the NDC range.
    
    $$ o_0 = -\frac{1}{W / (2 \cdot focal)} \cdot \frac{rays\_o[..., 0]}{rays\_o[..., 2]} $$
    $$ o_1 = -\frac{1}{H / (2 \cdot focal)} \cdot \frac{rays\_o[..., 1]}{rays\_o[..., 2]} $$
    $$ o_2 = 1 + 2 \cdot \frac{near}{rays\_o[..., 2]} $$
    
    Here, $o_2$ maps the near plane ($rays\_o[..., 2] = -near$) to $1 + 2(-1) = -1$? No, if $rays\_o[..., 2] = -near$, then $near/rays\_o[..., 2] = -1$, so $o_2 = 1 - 2 = -1$. However, source [S4, unofficial] and [S6, unofficial] state $o_2 = 1 + 2 \cdot near / rays\_o[..., 2]$. If the near plane is at $z=-near$, then $o_2 = -1$. If the near plane is at $z=near$ (camera looks +z), then $o_2 = 3$. The sources [S1, S4, S6, unofficial] generally map the near plane to a specific bound. Source [S2, unofficial] mentions fixing near to 0 and far to 1. Source [S4, unofficial] formula results in $o_2 \in [-1, 1]$ if $rays\_o[..., 2]$ ranges from $-near$ to $-\infty$. Specifically, at $z=-near$, $o_2 = -1$. At $z \to -\infty$, $o_2 \to 1$. Thus the range is $[-1, 1]$.

3.  **Project Directions to NDC:**
    Compute the NDC direction components. This accounts for the perspective division difference between the origin and the point at infinity.
    
    $$ d_0 = -\frac{1}{W / (2 \cdot focal)} \cdot \left( \frac{rays\_d[..., 0]}{rays\_d[..., 2]} - \frac{rays\_o[..., 0]}{rays\_o[..., 2]} \right) $$
    $$ d_1 = -\frac{1}{H / (2 \cdot focal)} \cdot \left( \frac{rays\_d[..., 1]}{rays\_d[..., 2]} - \frac{rays\_o[..., 1]}{rays\_o[..., 2]} \right) $$
    $$ d_2 = -2 \cdot \frac{near}{rays\_o[..., 2]} $$
    
    The direction z-component $d_2$ ensures that the ray in NDC space points towards the correct depth range. At the near plane, $d_2 = 2$. As $z \to -\infty$, $d_2 \to 0$.

4.  **Stack Results:**
    Concatenate the components to form the final NDC rays.
    $$ rays\_o\_ndc = [o_0, o_1, o_2] $$
    $$ rays\_d\_ndc = [d_0, d_1, d_2] $$

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def convert_to_ndc(rays_o, rays_d, focal, W, H, near=1.0):
    """
    Convert camera-space rays to NDC space.
    
    Args:
        rays_o: (..., 3) ray origins in camera space.
        rays_d: (..., 3) ray directions in camera space.
        focal: float, focal length.
        W: int, image width.
        H: int, image height.
        near: float, near plane distance.
        
    Returns:
        rays_o_ndc: (..., 3) NDC origins.
        rays_d_ndc: (..., 3) NDC directions.
    """
    # Step 1: Shift origins to the near plane
    # t = -(near + rays_o_z) / rays_d_z
    # Note: This assumes camera looks along negative z-axis.
    # If rays_d_z is 0, this will result in inf/nan.
    t = -(near + rays_o[..., 2:3]) / rays_d[..., 2:3]
    rays_o_shifted = rays_o + t * rays_d
    
    # Step 2: Project origins to NDC
    # o0 = -1 / (W / (2*focal)) * (ox / oz)
    # o1 = -1 / (H / (2*focal)) * (oy / oz)
    # o2 = 1 + 2 * near / oz
    
    inv_hw = np.array([W / (2.0 * focal), H / (2.0 * focal)])
    
    o0 = -1.0 / inv_hw[0] * (rays_o_shifted[..., 0:1] / rays_o_shifted[..., 2:3])
    o1 = -1.0 / inv_hw[1] * (rays_o_shifted[..., 1:2] / rays_o_shifted[..., 2:3])
    o2 = 1.0 + 2.0 * near / rays_o_shifted[..., 2:3]
    
    rays_o_ndc = np.concatenate([o0, o1, o2], axis=-1)
    
    # Step 3: Project directions to NDC
    # d0 = -1 / (W / (2*focal)) * (dx/dz - ox/oz)
    # d1 = -1 / (H / (2*focal)) * (dy/dz - oy/oz)
    # d2 = -2 * near / oz
    
    # Precompute ox/oz and oy/oz from the shifted origin
    ox_oz = rays_o_shifted[..., 0:1] / rays_o_shifted[..., 2:3]
    oy_oz = rays_o_shifted[..., 1:2] / rays_o_shifted[..., 2:3]
    
    # Direction components in camera space
    dx_dz = rays_d[..., 0:1] / rays_d[..., 2:3]
    dy_dz = rays_d[..., 1:2] / rays_d[..., 2:3]
    
    d0 = -1.0 / inv_hw[0] * (dx_dz - ox_oz)
    d1 = -1.0 / inv_hw[1] * (dy_dz - oy_oz)
    d2 = -2.0 * near / rays_o_shifted[..., 2:3]
    
    rays_d_ndc = np.concatenate([d0, d1, d2], axis=-1)
    
    return rays_o_ndc, rays_d_ndc
```

**Edge Cases and Pitfalls:**
- **Zero Direction Z:** If `rays_d[..., 2]` is zero, the shift factor $t$ becomes infinite. This occurs for rays parallel to the near plane. In practice, this should be handled by clamping or checking for zero denominators [G].
- **Sign Conventions:** Different libraries (e.g., Blender vs. OpenGL) use different camera coordinate systems. Blender uses a right-handed system where the camera looks down +z, while OpenGL uses a left-handed system where the camera looks down -z. The formula for $t$ and the signs in the projection must be adjusted accordingly [S7, S8, unofficial].
- **Spherical Poses:** NDC assumes a forward-facing scene. For spherical or omnidirectional scenes, NDC can clip geometry or distort depth significantly. A `spheric_poses` flag is often used to bypass NDC conversion and use scene bounds instead [S2, S6, unofficial].

## Subtopics

### [Ray Generation from Pixel Coordinates](2514-ray-generation-from-pixel-coordinates.md)
Generates world-space rays from pixel indices. It typically precedes NDC conversion if the scene requires NDC sampling.

### [Pinhole Camera Projection (Pixel to Camera)](3162-pinhole-camera-projection-pixel-to-camera.md)
Projects pixels to camera-space rays. This is the immediate predecessor to NDC conversion.

## Operating characteristics
- **Time Complexity:** $O(N)$ where $N$ is the number of rays. The operations are element-wise arithmetic.
- **Memory Complexity:** $O(N)$ for storing the transformed rays.
- **Performance:** The transformation is computationally inexpensive compared to ray marching or volume rendering. It is typically performed once per ray batch.
- **Strengths:** Simplifies sampling strategies by fixing the depth range to $[0, 1]$ or $[-1, 1]$. Enables uniform sampling in NDC space to correspond to inverse depth sampling in world space, which is beneficial for forward-facing scenes with large depth variations [S2, S4, unofficial].
- **Limitations:** Not suitable for non-forward-facing or spherical scenes, as it distorts depth and can clip geometry [S2, S6, unofficial]. Requires accurate focal length and image dimensions.
- **When to Use:** Forward-facing scenes (e.g., LLFF dataset) where depth range is large and uniform sampling is desired [S4, S6, unofficial].
- **When Not to Use:** Spherical scenes, scenes with significant camera rotation around the object, or when exact world-space depth is required for other processing steps [S2, S6, unofficial].

## Background
General NDC Ray Conversion was introduced in the context of NeRF (Neural Radiance Fields) to handle forward-facing scenes efficiently. It was popularized by the original NeRF paper and subsequent implementations like those in the `nerfies` and `multinerf` repositories [S1, S4, unofficial]. The technique maps the perspective frustum to a canonical cube, allowing for simpler and more robust sampling strategies. It is particularly useful for datasets like LLFF (Look at Light From the Front) where the camera moves primarily in a forward-facing manner [S4, S6, unofficial].

## Sources
[S1] google-research/multinerf, https://github.com/google-research/multinerf, fetched 2026-10-26 (unofficial)
[S2] kwea123/nerf_pl, https://github.com/kwea123/nerf_pl, fetched 2026-10-26 (unofficial)
[S3] sjtuytc/UnboundedNeRFPytorch, https://github.com/sjtuytc/UnboundedNeRFPytorch, fetched 2026-10-26 (unofficial)
[S4] damn8daniel/nerf-from-scratch, https://github.com/damn8daniel/nerf-from-scratch, fetched 2026-10-26 (unofficial)
[S5] weiyithu/NerfingMVS, https://github.com/weiyithu/NerfingMVS, fetched 2026-10-27 (unofficial)
[S6] YoujiaZhang/NeMF, https://github.com/YoujiaZhang/NeMF, fetched 2026-10-27 (unofficial)
[S7] GitHub - apchenstu/TensoRF: [ECCV 2022] Tensorial Radiance Fields, a ..., https://github.com/apchenstu/TensoRF, fetched 2026-10-29 (unofficial)
[S8] https://github.com/ZhenyangLiu/Collaborative-Neural-Radiance-Fields-for-Novel-View-Synthesis, https://github.com/ZhenyangLiu/Collaborative-Neural-Radiance-Fields-for-Novel-View-Synthesis, fetched 2026-10-29 (unofficial)
[S9] https://github.com/CPREgroup/CaSE-NeRF, https://github.com/CPREgroup/CaSE-NeRF, fetched 2026-10-30 (unofficial)
