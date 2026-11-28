# 3D Gaussian Splatting (glTF KHR_gaussian_splatting)

Also called: 3D Gaussian Splatting in glTF, 3DGS, Ellipsoid Splatting, Gaussian Splatting Extension

## Overview
This method defines the rendering pipeline for 3D Gaussian splats stored in glTF assets via the `KHR_gaussian_splatting` extension, treating splats as point primitives with specific geometric and color attributes [S1]. It differs from general rasterization implementations by enforcing a strict data layout and transformation constraints defined by the glTF standard, ensuring interoperability and fallback to sparse point cloud rendering if the extension is unsupported [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A glTF scene containing a mesh primitive with mode `POINTS` (0) and attributes defined by the extension [S1]. The scene includes a camera with perspective projection [S1].
*   **Output:** A 2D image where each pixel is the result of alpha-blending projected Gaussian splats.

**Data Structures and Layout:**
The splat data is stored as attributes on a glTF mesh primitive [S1]. The primitive mode MUST be `POINTS` [S1]. The glTF material referenced by the primitive MUST be ignored for rendering [S1].

1.  **POSITION** (`VEC3`): The mean vector $\mu$ in local space. Effective global mean is derived from this attribute and the node's global transformation matrix [S1].
2.  **ROTATION** (`VEC4`): Unit quaternion $(q_x, q_y, q_z, q_w)$ in glTF order. Values are guaranteed to be normalized [S1]. Supported component types: `float32`, `snorm8`, `snorm16` [S1].
3.  **SCALE** (`VEC3`): Linear spread $(s_x, s_y, s_z)$ along local principal axes. Values MUST be non-negative [S1]. Supported component types: `float32`, `uint8`, `unorm8`, `uint16`, `unorm16` [S1].
4.  **OPACITY** (`SCALAR`): Linear value $\alpha \in [0.0, 1.0]$. Out-of-range values are invalid. Supported types: `float32`, `unorm8`, `unorm16` [S1].
5.  **Spherical Harmonics (SH):**
    *   `SH_DEGREE_0_COEF_0` (`VEC3`): Required. Degree 0 coefficients.
    *   `SH_DEGREE_l_COEF_n` (`VEC3`): Required if degree $l$ is used. Degree 1 has 3 coefficients, Degree 2 has 5, Degree 3 has 7 [S1].
    *   Degrees must be fully defined; partial definitions are invalid. Coefficients are packed from lowest order $m$ to highest [S1].
    *   Total 45 coefficients for all 3 degrees [S1].

**Node Transformation Constraints:**
The global transformation matrix of the node containing the splat primitive MUST satisfy:
*   Last row is $(0, 0, 0, 1)$ [S1].
*   Lengths of the first three columns are finite positive numbers [S1].
*   The determinant of the upper-left 3x3 matrix (formed by dividing each element by its column's length) is close to +1 [S1].
*   Non-decomposable matrices or negative scales are undefined [S1].

**Rendering Steps:**

1.  **Splat Reconstruction (Covariance Construction):**
    Compute the 3D covariance matrix $\Sigma$ in global space.
    Let $\mathbf{M}$ be the upper-left 3x3 submatrix of the node's global transformation matrix [S1].
    Construct matrix $\mathbf{C}$ from scale $s_x, s_y, s_z$ and quaternion $q_x, q_y, q_z, q_w$:
    $$ \mathbf{C} = \begin{pmatrix} s_x (1 - 2(q_y^2 + q_z^2)) & s_y (2(q_x q_y - q_w q_z)) & s_z (2(q_x q_z + q_w q_y)) \\ s_x (2(q_x q_y + q_w q_z)) & s_y (1 - 2(q_x^2 + q_z^2)) & s_z (2(q_y q_z - q_w q_x)) \\ s_x (2(q_x q_z - q_w q_y)) & s_y (2(q_y q_z + q_w q_x)) & s_z (1 - 2(q_x^2 + q_y^2)) \end{pmatrix} $$
    Compute $\Sigma$:
    $$ \Sigma = \mathbf{M} \mathbf{C} \mathbf{C}^T \mathbf{M}^T $$ [S1]

2.  **Splat Projection:**
    *   **Mean Vector:** Project the global mean vector $\mu$ onto the 2D image plane using standard perspective projection [S1].
    *   **2D Covariance:** Transform $\Sigma$ to 2D covariance $\Sigma'$ using affine approximation:
        $$ \Sigma' = \mathbf{J} \mathbf{W} \Sigma \mathbf{W}^T \mathbf{J}^T $$
        Where $\mathbf{W}$ is the 3x3 view-dependent transformation matrix from global to camera space (typically the upper-left 3x3 of the camera view matrix) [S1].
        Where $\mathbf{J}$ is the Jacobian of the perspective projection [S1].

3.  **Color Computation:**
    Evaluate the spherical harmonics at the viewing direction to determine the RGB color. The color space is specified by the `colorSpace` property (`srgb_rec709_display` or `lin_rec709_display`) [S1].

4.  **Sorting:**
    Sort splats by distance to the camera origin. The default method is `cameraDistance` (vector length from splat to camera) [S1].

5.  **Rasterization and Blending:**
    Project each splat onto the 2D plane. Assume a 3$\sigma$ cut-off (Mahalanobis distance of 3 units) for the kernel shape [S1]. Alpha-blend the splats in sorted order. Intermediate rendering steps (including alpha blending) MUST be performed before any color gamut or transfer function conversions [S1].

**Parameters:**
*   `kernel`: String, required. Currently only `"ellipse"` is defined [S1].
*   `colorSpace`: String, required. `"srgb_rec709_display"` or `"lin_rec709_display"` [S1].
*   `projection`: String, optional, default `"perspective"`. Specifies projection method [S1].
*   `sortingMethod`: String, optional, default `"cameraDistance"` [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import numpy as np

def compute_c_matrix(scale: np.ndarray, rotation: np.ndarray) -> np.ndarray:
    sx, sy, sz = scale
    qx, qy, qz, qw = rotation
    return np.array([
        [sx * (1 - 2 * (qy**2 + qz**2)), sy * (2 * (qx * qy - qw * qz)), sz * (2 * (qx * qz + qw * qy))],
        [sx * (2 * (qx * qy + qw * qz)), sy * (1 - 2 * (qx**2 + qz**2)), sz * (2 * (qy * qz - qw * qx))],
        [sx * (2 * (qx * qz - qw * qy)), sy * (2 * (qy * qz + qw * qx)), sz * (1 - 2 * (qx**2 + qy**2))]
    ])

def evaluate_spherical_harmonics(coeffs: list, direction: np.ndarray) -> np.ndarray:
    # The sources do not provide the SH basis functions or evaluation algorithm.
    # This implements degree 0 (constant) for demonstration.
    if len(coeffs) == 0:
        return np.array([0.0, 0.0, 0.0])
    return coeffs[0]

def linear_to_srgb(linear: np.ndarray) -> np.ndarray:
    # [G] Standard sRGB transfer function.
    return np.where(linear <= 0.0031308, 12.92 * linear, 1.055 * np.power(linear, 1/2.4) - 0.055)

def render_gaussian_splats(splats: list, camera_view_matrix: np.ndarray, camera_proj_matrix: np.ndarray, 
                           camera_world_pos: np.ndarray, image_width: int, image_height: int, 
                           color_space: str) -> np.ndarray:
    image = np.zeros((image_height, image_width, 4), dtype=np.float32)
    
    # 1. Sort splats by camera distance
    sorted_splats = []
    for splat in splats:
        node_mat = splat['node_transform']
        pos_local = np.array(splat['position'])
        pos_homogeneous = np.append(pos_local, 1.0)
        pos_global = (node_mat @ pos_homogeneous)[:3]
        dist = np.linalg.norm(pos_global - camera_world_pos)
        sorted_splats.append((dist, splat, pos_global))
    sorted_splats.sort(key=lambda x: x[0], reverse=True)
    
    # 2. Render
    for dist, splat, pos_global in sorted_splats:
        scale = np.array(splat['scale'])
        rotation = np.array(splat['rotation'])
        opacity = splat['opacity']
        sh_coeffs = splat['sh_coeffs']
        
        # Compute Covariance Sigma
        C = compute_c_matrix(scale, rotation)
        M = splat['node_transform'][:3, :3]
        Sigma = M @ C @ C.T @ M.T
        
        # Project Mean
        pos_homogeneous_global = np.append(pos_global, 1.0)
        pos_view = (camera_view_matrix @ pos_homogeneous_global)[:3]
        z = pos_view[2]
        if z <= 0:
            continue
            
        pos_view_homogeneous = np.append(pos_view, 1.0)
        pos_clip = camera_proj_matrix @ pos_view_homogeneous
        if pos_clip[3] == 0:
            continue
        pos_ndc = pos_clip[:3] / pos_clip[3]
        
        screen_x = (pos_ndc[0] + 1) * 0.5 * image_width
        screen_y = (1 - pos_ndc[1]) * 0.5 * image_height
        
        # Project Covariance to 2D
        W = camera_view_matrix[:3, :3]
        Sigma_view = W @ Sigma @ W.T
        
        # Jacobian for perspective projection (x/z, y/z)
        x, y = pos_view[0], pos_view[1]
        J = np.array([
            [1/z, 0, -x/(z**2)],
            [0, 1/z, -y/(z**2)]
        ])
        Sigma_2d = J @ Sigma_view @ J.T
        
        # Scale to pixel coordinates
        scale_x, scale_y = image_width / 2, image_height / 2
        Sigma_2d_pixels = np.array([
            [Sigma_2d[0,0] * scale_x**2, Sigma_2d[0,1] * scale_x * scale_y],
            [Sigma_2d[1,0] * scale_x * scale_y, Sigma_2d[1,1] * scale_y**2]
        ])
        
        eigenvalues_px, _ = np.linalg.eigh(Sigma_2d_pixels)
        sigma_x_px = math.sqrt(eigenvalues_px[0])
        sigma_y_px = math.sqrt(eigenvalues_px[1])
        
        min_x = max(0, int(math.floor(screen_x - 3 * sigma_x_px)))
        max_x = min(image_width, int(math.ceil(screen_x + 3 * sigma_x_px)))
        min_y = max(0, int(math.floor(screen_y - 3 * sigma_y_px)))
        max_y = min(image_height, int(math.ceil(screen_y + 3 * sigma_y_px)))
        
        # Color Computation
        cam_to_splat = pos_global - camera_world_pos
        cam_to_splat = cam_to_splat / np.linalg.norm(cam_to_splat)
        color = evaluate_spherical_harmonics(sh_coeffs, cam_to_splat)
        
        # Alpha Blending
        inv_sigma = np.linalg.inv(Sigma_2d_pixels)
        for py in range(min_y, max_y):
            for px in range(min_x, max_x):
                dx = px - screen_x
                dy = py - screen_y
                pos_vec = np.array([dx, dy])
                exponent = -0.5 * pos_vec @ inv_sigma @ pos_vec
                weight = math.exp(max(exponent, -10))
                alpha_splat = opacity * weight
                
                if alpha_splat > 1e-6:
                    current_alpha = image[py, px, 3]
                    current_color = image[py, px, :3]
                    new_alpha = alpha_splat + current_alpha * (1 - alpha_splat)
                    if new_alpha > 0:
                        new_color = (color * alpha_splat + current_color * current_alpha * (1 - alpha_splat)) / new_alpha
                    else:
                        new_color = current_color
                    image[py, px, :3] = new_color
                    image[py, px, 3] = new_alpha

    if color_space == "srgb_rec709_display":
        image = linear_to_srgb(image)
    return image
```

## Subtopics

### [4D Gaussian Rasterization](13946-4d-gaussian-rasterization.md)
Renders 4D Gaussians (3D spatial + 1D temporal) into a 2D image for a specific viewpoint and timestamp. It extends static 3D Gaussian Splatting by handling time-dependent opacity, position offsets, and spherical harmonics evaluation in 4D. Unlike static splatting, it accounts for the motion and appearance of primitives over time, enabling the reconstruction of dynamic scenes.

## Operating characteristics
Not given in the sources.

## Background
The `KHR_gaussian_splatting` extension was developed by Khronos Group to standardize the storage of 3D Gaussian splats in glTF assets [S1]. It builds on the 3D Gaussian Splatting for Real-Time Radiance Field Rendering paper [S1]. The extension aims to establish a solid foundation for integrating 3D Gaussian splatting into glTF assets, enabling future enhancements [S1].

## Sources
[S1] https://github.com/KhronosGroup/glTF/blob/main/extensions/2.0/Khronos/KHR_gaussian_splatting/README.md, https://github.com/KhronosGroup/glTF/blob/main/extensions/2.0/Khronos/KHR_gaussian_splatting/README.md, fetched 2026-10-30
[S2] KHR_gaussian_splatting, https://github.com/KhronosGroup/glTF/tree/main/extensions/2.0/Khronos/KHR_gaussian_splatting, fetched 2026-10-30
