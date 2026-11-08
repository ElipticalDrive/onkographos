# Compute Shader Ray Generation

Also called: Basic GPU Ray Generation via Compute Shader, CUDA Ray Generation, Kamera-Strahl-Erzeugung im Compute-Shader, Ray Generation Program

## Overview
This method generates initial camera rays for a frame by launching a compute shader with one thread per pixel [S2]. It converts thread indices into normalized UV coordinates and computes ray directions using a pinhole camera model defined by a focal length [S1, unofficial][S2]. It differs from standard perspective projection implementations by explicitly detailing the normalization to $[-1, 1]$ and the specific focal length derivation from the field of view, while deferring camera orientation (rotation) to subsequent transformation steps [S1, unofficial][S2].

## How it works
**Inputs and Outputs:**
*   Window width $W$ and height $H$ (integers) [S2].
*   Field of view angle $\alpha$ (radians) [S1, unofficial].
*   Camera orientation angles $\phi$ (rotation around x-axis) and $\theta$ (rotation around y-axis) [S2].
*   Outputs: Ray origin $\vec{o}$ (world space, camera position) [S2] and ray direction $\vec{rd}$ (world space, normalized) [S2].

**Data Structures:**
*   No persistent data structures are required for ray generation itself [G]. The compute shader state relies on the hardware-provided `DispatchThreadID` (global thread ID) and `GroupThreadID`/`GroupID` for coordinate mapping [S1, unofficial][S2].

**Numbered Steps:**
1.  Determine the pixel coordinates $(x, y)$ from the thread ID within the block and the block ID within the grid [S2].
2.  Scale the integer pixel coordinates to the range $[0, 1]$ by dividing by window dimensions [S1, unofficial].
3.  Center the coordinates so the screen center is $(0,0)$, mapping the top-left pixel to $[-1, 1]$ and the bottom-right to $[1, -1]$ [S1, unofficial][S2].
4.  Calculate the focal length $f$ from the field of view $\alpha$ [S1, unofficial].
5.  Compute the initial unoriented ray direction $\vec{rd}^0$ in a local coordinate system where the camera looks along the positive Z-axis [S2].
6.  Set the ray origin. In the local coordinate system, it is $\vec{o}_{origin} = [0, 0, -f]^T$ [S1, unofficial]. In world space, it corresponds to the camera position $\vec{ro}$ [S2].
7.  Transform $\vec{rd}^0$ and $\vec{o}_{origin}$ into world space using rotation matrices defined by camera angles $\phi$ and $\theta$ [S2].

**Formulas:**
*   Pixel scaling: $\vec{v}_{scaledPixelCoord} = (x/W, y/H)$ [S1, unofficial].
*   UV centering: $\vec{uv} = \vec{v}_{scaledPixelCoord} \cdot 2 - \vec{1}$, where $u = \vec{uv}_x$ and $v = \vec{uv}_y$ [S1, unofficial][S2].
*   Focal length: $f = 1 / \tan(\alpha / 2)$ [S1, unofficial].
*   Local direction: $\vec{rd}^0 = [u, v, f]^T / \sqrt{u^2 + v^2 + f^2}$ [S2]. Source [S1, unofficial] describes an equivalent vector subtraction method where the origin is $\vec{o}_{origin} = [0, 0, -f]^T$ and the direction is $\vec{p}_{direction} = [u, v, 0]^T - [0, 0, -f]^T = [u, v, f]^T$, which yields the same direction vector before normalization.
*   World orientation: $\vec{rd} = R_y(\theta) \cdot R_x(\phi) \cdot \vec{rd}^0$ and $\vec{o} = R_y(\theta) \cdot R_x(\phi) \cdot \vec{o}_{origin}$ [S2].
*   Rotation matrices: $R_y(\theta) = \begin{bmatrix} \cos\theta & 0 & \sin\theta \\ 0 & 1 & 0 \\ -\sin\theta & 0 & \cos\theta \end{bmatrix}$ and $R_x(\phi) = \begin{bmatrix} 1 & 0 & 0 \\ 0 & \cos\phi & -\sin\phi \\ 0 & \sin\phi & \cos\phi \end{bmatrix}$ [S2].

**Parameters:**
*   Thread block size: $8 \times 8$ threads (64 threads per group) [S1, unofficial][S2].
*   Focal length $f$: Derived from $\alpha$ [S1, unofficial].
*   Near plane width $v_{nearPlaneWidth}$: Set to $1$ [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Division by Zero:** If $W$ or $H$ is zero, normalization fails [G]. Ensure $W, H > 0$.
*   **Precision:** Using integer arithmetic for thread ID calculation is safe. Floating point precision for $u, v$ depends on the window size. For very large resolutions, $u^2 + v^2 + f^2$ may suffer from precision loss if $f$ is very small or very large [G].
*   **Boundary Pixels:** The mapping ensures all pixels from $(0,0)$ to $(W-1, H-1)$ are covered. The window is divided into $8 \times 8$ squares, and the dispatch arguments are the number of squares in the X and Y directions [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def generate_ray(group_id_x, group_id_y, group_thread_id_x, group_thread_id_y, W, H, alpha, phi, theta):
    # 1. Determine pixel coordinates
    x = group_id_x * 8 + group_thread_id_x
    y = group_id_y * 8 + group_thread_id_y

    # 2. Normalize to [0, 1]
    u_scaled = x / W
    v_scaled = y / H

    # 3. Center to [-1, 1]
    u = u_scaled * 2.0 - 1.0
    v = v_scaled * 2.0 - 1.0

    # 4. Calculate focal length
    f = 1.0 / math.tan(alpha / 2.0)

    # 5. Calculate local direction vector [u, v, f] and normalize
    rd_local_x = u
    rd_local_y = v
    rd_local_z = f
    rd_local_len = math.sqrt(rd_local_x**2 + rd_local_y**2 + rd_local_z**2)
    rd_local_x /= rd_local_len
    rd_local_y /= rd_local_len
    rd_local_z /= rd_local_len

    # 6. Set local origin
    origin_local_x = 0.0
    origin_local_y = 0.0
    origin_local_z = -f

    # 7. Apply rotation matrices for camera orientation
    cos_theta = math.cos(theta)
    sin_theta = math.sin(theta)
    cos_phi = math.cos(phi)
    sin_phi = math.sin(phi)

    # Apply R_x(phi)
    rd_x_temp = rd_local_x
    rd_y_temp = rd_local_y * cos_phi - rd_local_z * sin_phi
    rd_z_temp = rd_local_y * sin_phi + rd_local_z * cos_phi

    origin_x_temp = origin_local_x
    origin_y_temp = origin_local_y * cos_phi - origin_local_z * sin_phi
    origin_z_temp = origin_local_y * sin_phi + origin_local_z * cos_phi

    # Apply R_y(theta)
    rd_world_x = rd_x_temp * cos_theta + rd_z_temp * sin_theta
    rd_world_y = rd_y_temp
    rd_world_z = -rd_x_temp * sin_theta + rd_z_temp * cos_theta

    origin_world_x = origin_x_temp * cos_theta + origin_z_temp * sin_theta
    origin_world_y = origin_y_temp
    origin_world_z = -origin_x_temp * sin_theta + origin_z_temp * cos_theta

    return (origin_world_x, origin_world_y, origin_world_z), (rd_world_x, rd_world_y, rd_world_z)
```

## Subtopics

### [Compute Shader Voxel Raycasting with Corner Ray Interpolation](4215-compute-shader-voxel-raycasting-with-corner-ray-interpolatio.md)
Generates view rays for each framebuffer pixel by bilinearly interpolating four pre-computed world-space corner rays of the camera's view frustum, then traces them through a 3D voxel or Sparse Voxel Octree (SVO) texture to produce a color image. It differs from standard perspective projection by avoiding per-pixel matrix-vector multiplication for ray direction generation, instead relying on spatial interpolation of boundary vectors. Choose this method when minimizing arithmetic intensity per pixel is critical and the camera is static or moves slowly.

## Operating characteristics
*   **Time Complexity:** $O(1)$ per pixel. The computation involves a constant number of arithmetic operations (normalization, trigonometry, normalization) [G].
*   **Memory Complexity:** $O(1)$ per thread. No additional memory is allocated for ray generation [G].
*   **Performance:** Highly parallelizable. The $8 \times 8$ thread block size aligns with typical GPU warp/wavefront sizes (32 or 64 threads), ensuring efficient SIMD execution [S2].
*   **Strengths:** Simple, deterministic, and low overhead. Suitable for real-time applications where ray generation is a small fraction of the total rendering cost [G].
*   **Limitations:** Assumes a pinhole camera model. Does not account for lens effects (bokeh, distortion) without extension. Requires explicit rotation handling for non-axis-aligned cameras [S2].

## Background
This method originates from implementations of GPU-based raycasting and raymarching, specifically those utilizing compute shaders for general-purpose rendering tasks. It is detailed in works on CUDA-based distance field raymarching [S2] and voxel raycasting implementations [S1, unofficial][S2]. The approach of mapping thread IDs to pixel coordinates and generating rays via a pinhole model is a standard technique in GPU ray tracing pipelines [G].

## Sources
[S1] Raycasting-basiertes Voxelrendering, https://serwiss.bib.hs-hannover.de/frontdoor/index/index/docId/2250, fetched 2026-10-25 (unofficial)
[S2] Raymarching Distance Fields with CUDA, https://doi.org/10.3390/electronics10222730, fetched 2026-10-26
