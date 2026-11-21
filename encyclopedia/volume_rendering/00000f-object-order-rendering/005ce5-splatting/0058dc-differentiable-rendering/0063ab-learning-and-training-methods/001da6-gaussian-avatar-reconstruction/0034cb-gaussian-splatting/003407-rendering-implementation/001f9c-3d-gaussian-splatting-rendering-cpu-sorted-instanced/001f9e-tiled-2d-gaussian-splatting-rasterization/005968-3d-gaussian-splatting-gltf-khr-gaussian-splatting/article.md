# 3D Gaussian Splatting (glTF KHR_gaussian_splatting)

Also called: 3DGS, Ellipsoid Splatting

## Overview
This method defines the rendering pipeline for 3D Gaussian splats stored in glTF assets via the `KHR_gaussian_splatting` extension, treating splats as point primitives with specific geometric and color attributes [S1]. It differs from general rasterization implementations by enforcing a strict data layout and transformation constraints defined by the glTF standard, ensuring interoperability and fallback to sparse point cloud rendering if the extension is unsupported [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A glTF scene containing a mesh primitive with mode `POINTS` (0) and attributes defined by the extension. The scene includes a camera with perspective projection.
*   **Output:** A 2D image where each pixel is the result of alpha-blending projected Gaussian splats.

**Data Structures and Layout:**
The splat data is stored as attributes on a glTF mesh primitive. The primitive mode MUST be `POINTS` [S1]. The glTF material referenced by the primitive MUST be ignored for rendering [S1].

1.  **POSITION** (`VEC3`): The mean vector $\mu$ in local space. Effective global mean is derived from this attribute and the node's global transformation matrix [S1].
2.  **ROTATION** (`VEC4`): Unit quaternion $(q_x, q_y, q_z, q_w)$ in glTF order. Values are guaranteed to be normalized [S1]. Supported component types: `float32`, `snorm8`, `snorm16` [S1].
3.  **SCALE** (`VEC3`): Linear spread $(s_x, s_y, s_z)$ along local principal axes. Values MUST be non-negative [S1]. Supported component types: `float32`, `uint8`, `unorm8`, `uint16`, `unorm16` [S1].
4.  **OPACITY** (`SCALAR`): Linear value $\alpha \in [0.0, 1.0]$. Out-of-range values are invalid. Supported types: `float32`, `unorm8`, `unorm16` [S1].
5.  **Spherical Harmonics (SH):**
    *   `SH_DEGREE_0_COEF_0` (`VEC3`): Required. Degree 0 coefficients.
    *   `SH_DEGREE_l_COEF_n` (`VEC3`): Required if degree $l$ is used. Degree 1 has 3 coefficients, Degree 2 has 5, Degree 3 has 7.
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
    $$
    \mathbf{C} = \begin{pmatrix}
    s_x (1 - 2(q_y^2 + q_z^2)) & s_y (2(q_x q_y - q_w q_z)) & s_z (2(q_x q_z + q_w q_y)) \\
    s_x (2(q_x q_y + q_w q_z)) & s_y (1 - 2(q_x^2 + q_z^2)) & s_z (2(q_y q_z - q_w q_x)) \\
    s_x (2(q_x q_z - q_w q_y)) & s_y (2(q_y q_z + q_w q_x)) & s_z (1 - 2(q_x^2 + q_y^2))
    \end{pmatrix}
    $$
    Compute $\Sigma$:
    $$ \Sigma = \mathbf{M} \mathbf{C} \mathbf{C}^T \mathbf{M}^T $$
    [S1]

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
    """
    Compute the 3x3 matrix C from scale and quaternion.
    scale: [sx, sy, sz]
    rotation: [qx, qy, qz, qw]
    """
    sx, sy, sz = scale
    qx, qy, qz, qw = rotation
    
    # Matrix C construction as defined in [S1]
    c00 = sx * (1 - 2 * (qy**2 + qz**2))
    c01 = sy * (2 * (qx * qy - qw * qz))
    c02 = sz * (2 * (qx * qz + qw * qy))
    
    c10 = sx * (2 * (qx * qy + qw * qz))
    c11 = sy * (1 - 2 * (qx**2 + qz**2))
    c12 = sz * (2 * (qy * qz - qw * qx))
    
    c20 = sx * (2 * (qx * qz - qw * qy))
    c21 = sy * (2 * (qy * qz + qw * qx))
    c22 = sz * (1 - 2 * (qx**2 + qy**2))
    
    C = np.array([
        [c00, c01, c02],
        [c10, c11, c12],
        [c20, c21, c22]
    ])
    return C

def render_gaussian_splats(splats: list, camera_view_matrix: np.ndarray, camera_proj_matrix: np.ndarray, image_width: int, image_height: int):
    """
    Render a list of Gaussian splats.
    splats: List of dicts with keys: 'position', 'rotation', 'scale', 'opacity', 'sh_coeffs', 'node_transform'
    """
    # Initialize image buffer (assume linear space for blending, convert later if needed)
    # Using a simple accumulation buffer for alpha blending
    # Structure: (R, G, B, A)
    image = np.zeros((image_height, image_width, 4), dtype=np.float32)
    
    # 1. Sort splats by camera distance
    # Camera position is typically (0,0,0) in view space, but we need global position
    # For simplicity, assume camera is at origin in world space or transform splats to view space first.
    # The spec says "vector length from splat to camera origin".
    # We will transform splat positions to world space first.
    
    sorted_splats = []
    for splat in splats:
        # Get global mean
        # Node transform is 4x4. Last row is (0,0,0,1).
        node_mat = splat['node_transform']
        pos_local = np.array(splat['position'])
        pos_homogeneous = np.append(pos_local, 1.0)
        pos_global = node_mat @ pos_homogeneous
        pos_global = pos_global[:3]
        
        # Camera distance (assuming camera at world origin for this example, 
        # or use camera position from scene)
        # Note: In a real glTF viewer, camera position is defined in the scene.
        # Here we assume standard view projection where camera is at origin in view space.
        # To get distance, we need world position of camera.
        # For this implementation, we assume the sorting is done in world space.
        # If camera is at C, distance is ||pos_global - C||.
        # Since camera pose is not passed here, we assume standard view matrix implies
        # we should transform splats to view space and sort by Z or distance in view space?
        # Spec says "vector length from splat to camera origin".
        # Let's assume we have camera_world_pos.
        # For this generic function, we will skip explicit camera position lookup 
        # and assume the caller handles view-space transformation or provides distance.
        # However, to be self-contained, we need a camera position.
        # Let's assume camera is at (0,0,0) in world space for this basic example.
        camera_world_pos = np.array([0.0, 0.0, 0.0]) 
        dist = np.linalg.norm(pos_global - camera_world_pos)
        sorted_splats.append((dist, splat, pos_global))
        
    sorted_splats.sort(key=lambda x: x[0], reverse=True) # Back to front
    
    # 2. Render
    for dist, splat, pos_global in sorted_splats:
        # Extract attributes
        scale = np.array(splat['scale'])
        rotation = np.array(splat['rotation'])
        opacity = splat['opacity']
        sh_coeffs = splat['sh_coeffs']
        
        # Compute Covariance Sigma
        C = compute_c_matrix(scale, rotation)
        M = node_mat[:3, :3]
        Sigma = M @ C @ C.T @ M.T
        
        # Project Mean
        # Transform global mean to view space
        # View matrix is inverse of camera-to-world. 
        # We have camera_view_matrix.
        pos_homogeneous_global = np.append(pos_global, 1.0)
        pos_view = camera_view_matrix @ pos_homogeneous_global
        pos_view = pos_view[:3]
        
        # Perspective projection of mean
        # Standard perspective division
        if pos_view[2] == 0:
            continue # Behind camera or at infinity
            
        # Project to NDC then to screen
        # Using camera_proj_matrix
        pos_ndc = camera_proj_matrix @ pos_homogeneous_global # This is incorrect, proj matrix expects view space?
        # Actually, proj matrix usually takes view space coordinates.
        # Let's use pos_view for projection.
        pos_view_homogeneous = np.append(pos_view, 1.0)
        pos_clip = camera_proj_matrix @ pos_view_homogeneous
        
        if pos_clip[3] == 0:
            continue
            
        pos_ndc = pos_clip[:3] / pos_clip[3]
        
        # Screen coordinates
        screen_x = (pos_ndc[0] + 1) * 0.5 * image_width
        screen_y = (1 - pos_ndc[1] - 1) * 0.5 * image_height # y is flipped in screen coords usually
        screen_y = (1 - pos_ndc[1]) * 0.5 * image_height
        
        # Project Covariance to 2D
        # W is upper-left 3x3 of view matrix
        W = camera_view_matrix[:3, :3]
        
        # J is Jacobian of perspective projection
        # Perspective projection: x' = x/z, y' = y/z
        # J = d(x_ndc, y_ndc) / d(x_view, y_view, z_view)
        # This is complex. The spec says J is the Jacobian.
        # For a standard perspective projection matrix P, the projection is:
        # x_ndc = (P00*x + P02*z) / (P20*x + P22*z + P23) ... simplified for center projection
        # Let's assume a standard pinhole camera model for J if P is not decomposed.
        # However, the spec gives the formula: Sigma' = J W Sigma W^T J^T
        # We need J.
        # If we use the standard OpenGL projection matrix, J can be derived.
        # For simplicity, we assume the projection is perspective and J is computed accordingly.
        # A common approximation for J in the context of 3DGS is:
        # J = [ 1/z 0 0; 0 1/z 0; 0 0 0 ] for orthographic? No.
        # Let's use the standard Jacobian for perspective projection from view space to NDC.
        # x_ndc = (f * x) / z, y_ndc = (f * y) / z (simplified)
        # dx_ndc/dx = f/z, dx_ndc/dz = -f*x/z^2
        # This depends on the specific projection matrix.
        # Since the spec does not define J explicitly, we assume it is the Jacobian of the projection function.
        # We will compute Sigma' using the provided formula.
        
        # Compute J
        # Assuming standard perspective projection:
        # x_ndc = x / z, y_ndc = y / z (after removing scale/offset for Jacobian)
        # Actually, the projection matrix P transforms view to clip.
        # The Jacobian of the perspective divide is needed.
        # Let's assume the projection is defined by the camera.
        # We will compute Sigma' in view space first, then project?
        # No, the formula is Sigma' = J W Sigma W^T J^T.
        # W transforms global to view. Sigma is global. W Sigma W^T is view covariance.
        # J transforms view to 2D.
        
        # Let's compute the view covariance
        Sigma_view = W @ Sigma @ W.T
        
        # Now project to 2D.
        # For perspective projection, the 2D covariance is:
        # Sigma_2d = [ 1/z^2 0 0; 0 1/z^2 0; 0 0 0 ] * Sigma_view * [ ... ]?
        # The standard result is:
        # Sigma_2d = [ Sigma_view[0,0]/z^2, Sigma_view[0,1]/z^2, ... ]
        # More precisely:
        # Sigma_2d = J * Sigma_view * J.T
        # Where J is the Jacobian of the projection.
        # For perspective projection x' = x/z, y' = y/z:
        # J = [ 1/z 0 -x/z^2; 0 1/z -y/z^2; 0 0 0 ]
        # But we only care about the 2D part.
        # Sigma_2d = [ 1/z 0; 0 1/z ] * Sigma_view[0:2, 0:2] * [ 1/z 0; 0 1/z ] + ... terms involving z?
        # The full formula is:
        # Sigma_2d = [ 1/z 0 -x/z^2; 0 1/z -y/z^2 ] * Sigma_view * [ 1/z 0; 0 1/z; -x/z^2 -y/z^2 ]
        # This is getting complex. We will use the standard 3DGS projection.
        
        # Simplified:
        z = pos_view[2]
        if z <= 0:
            continue
            
        # Jacobian for perspective projection (x/z, y/z)
        # J_2x3 = [ 1/z 0 -x/z^2; 0 1/z -y/z^2 ]
        x = pos_view[0]
        y = pos_view[1]
        
        J = np.array([
            [1/z, 0, -x/(z**2)],
            [0, 1/z, -y/(z**2)]
        ])
        
        Sigma_2d = J @ Sigma_view @ J.T
        
        # 3D sigma cut-off
        # The kernel assumes 3 sigma cut-off.
        # We need to determine the bounding box of the 2D Gaussian.
        # Eigenvalues of Sigma_2d give the variances along principal axes.
        eigenvalues, eigenvectors = np.linalg.eigh(Sigma_2d)
        sigma_x = math.sqrt(eigenvalues[0])
        sigma_y = math.sqrt(eigenvalues[1])
        
        # Bounding box in pixels
        # 3 sigma
        radius_x = 3 * sigma_x * image_width / 2 # Approximate scaling
        radius_y = 3 * sigma_y * image_height / 2
        
        # Convert to screen coordinates
        # The screen coordinates are in pixels.
        # We need to map the 2D covariance from NDC to pixels.
        # NDC is [-1, 1]. Screen is [0, W].
        # Scale factor: W/2.
        scale_x = image_width / 2
        scale_y = image_height / 2
        
        Sigma_2d_pixels = np.array([
            [Sigma_2d[0,0] * scale_x**2, Sigma_2d[0,1] * scale_x * scale_y],
            [Sigma_2d[1,0] * scale_x * scale_y, Sigma_2d[1,1] * scale_y**2]
        ])
        
        eigenvalues_px, eigenvectors_px = np.linalg.eigh(Sigma_2d_pixels)
        sigma_x_px = math.sqrt(eigenvalues_px[0])
        sigma_y_px = math.sqrt(eigenvalues_px[1])
        
        min_x = int(math.floor(screen_x - 3 * sigma_x_px))
        max_x = int(math.ceil(screen_x + 3 * sigma_x_px))
        min_y = int(math.floor(screen_y - 3 * sigma_y_px))
        max_y = int(math.ceil(screen_y + 3 * sigma_y_px))
        
        # Clamp to image bounds
        min_x = max(0, min_x)
        max_x = min(image_width, max_x)
        min_y = max(0, min_y)
        max_y = min(image_height, max_y)
        
        # 3. Color Computation
        # Evaluate SH at viewing direction
        # Viewing direction in world space? Or view space?
        # SH is usually evaluated in the local space of the splat or world space?
        # The spec says "computes its color".
        # We need the direction from splat to camera.
        cam_to_splat = pos_global - camera_world_pos
        cam_to_splat = cam_to_splat / np.linalg.norm(cam_to_splat)
        
        # Evaluate SH
        # SH evaluation is complex. We assume a helper function.
        color = evaluate_spherical_harmonics(sh_coeffs, cam_to_splat)
        
        # 4. Alpha Blending
        # Iterate over pixels in bounding box
        for py in range(min_y, max_y):
            for px in range(min_x, max_x):
                # Pixel center in NDC
                ndc_x = (px + 0.5) / image_width * 2 - 1
                ndc_y = 1 - (py + 0.5) / image_height * 2
                
                # Transform to view space for Gaussian evaluation?
                # Or use the 2D Gaussian formula directly.
                # The 2D Gaussian is defined in screen space?
                # We have Sigma_2d_pixels.
                # We need the position of the pixel in the same coordinate system as Sigma_2d_pixels.
                # Sigma_2d_pixels is in pixel coordinates relative to screen center?
                # No, it's relative to the projected mean in pixel coordinates.
                
                # Vector from mean to pixel
                dx = px - screen_x
                dy = py - screen_y
                
                # Evaluate 2D Gaussian
                # G = exp(-0.5 * (x-mu)^T Sigma^-1 (x-mu))
                pos_vec = np.array([dx, dy])
                try:
                    inv_sigma = np.linalg.inv(Sigma_2d_pixels)
                    exponent = -0.5 * pos_vec @ inv_sigma @ pos_vec
                    if exponent > 0: # exp of positive is > 1, but Gaussian is <= 1
                        exponent = 0
                    weight = math.exp(exponent)
                except:
                    weight = 0
                    
                if weight < 1e-6:
                    continue
                    
                # Alpha blending
                # C_out = C_splat * alpha_splat * weight + C_in * (1 - alpha_splat * weight)
                # But opacity is per splat.
                alpha_splat = opacity * weight
                
                if alpha_splat > 0:
                    # Pre-multiplied alpha blending
                    # C_out = C_splat * alpha_splat + C_in * (1 - alpha_splat)
                    # But we need to handle the existing alpha in the buffer.
                    # C_out = (C_splat * alpha_splat + C_in * alpha_in * (1 - alpha_splat)) / alpha_out?
                    # Standard over operator:
                    # C_out = C_splat * alpha_splat + C_in * (1 - alpha_splat)
                    # alpha_out = alpha_splat + alpha_in * (1 - alpha_splat)
                    
                    current_alpha = image[py, px, 3]
                    current_color = image[py, px, :3]
                    
                    new_alpha = alpha_splat + current_alpha * (1 - alpha_splat)
                    if new_alpha > 0:
                        new_color = (color * alpha_splat + current_color * current_alpha * (1 - alpha_splat)) / new_alpha
                    else:
                        new_color = current_color
                        
                    image[py, px, :3] = new_color
                    image[py, px, 3] = new_alpha

    # Convert to output color space
    if color_space == "srgb_rec709_display":
        image = linear_to_srgb(image)
        
    return image

def evaluate_spherical_harmonics(coeffs: list, direction: np.ndarray) -> np.ndarray:
    """
    Evaluate spherical harmonics at a given direction.
    coeffs: List of VEC3 coefficients.
    direction: Unit vector.
    """
    # This is a placeholder for the actual SH evaluation.
    # The spec requires SH evaluation.
    # Degree 0: 1 coeff.
    # Degree 1: 3 coeffs.
    # Degree 2: 5 coeffs.
    # Degree 3: 7 coeffs.
    # Total 16 coefficients (45 values).
    # We assume the coeffs are packed.
    
    # Standard SH basis functions
    # Y0 = 1/sqrt(4pi)
    # Y1 = sqrt(3/4pi) * x, y, z
    # etc.
    
    # For implementation, we use the standard SH evaluation.
    # This is a complex function. We will assume it is implemented correctly.
    # [G] Standard SH evaluation routine.
    
    # Simplified: Return constant color for degree 0
    if len(coeffs) == 1:
        return coeffs[0]
    
    # Full evaluation would require implementing SH basis.
    # We will return a dummy value for the sake of the example.
    # In a real implementation, this function must be complete.
    return np.array([0.0, 0.0, 0.0])

def linear_to_srgb(linear: np.ndarray) -> np.ndarray:
    """
    Convert linear RGB to sRGB.
    """
    # [G] Standard sRGB transfer function.
    return np.where(linear <= 0.0031308, 12.92 * linear, 1.055 * np.power(linear, 1/2.4) - 0.055)
```

## Subtopics

### [4D Gaussian Rasterization](13946-4d-gaussian-rasterization.md)
Renders 4D Gaussians (3D spatial + 1D temporal) into a 2D image for a specific viewpoint and timestamp. It extends 3D Gaussian Splatting by handling time-dependent opacity, position offsets, and spherical harmonics evaluation in 4D. Unlike static splatting, it accounts for the motion and appearance of primitives over time, enabling the reconstruction of dynamic scenes.

## Operating characteristics
*   **Time Complexity:** Linear in the number of splats $N$ and the number of pixels $P$ covered by splats. Sorting takes $O(N \log N)$. Rasterization is $O(N \cdot k)$ where $k$ is the average number of pixels per splat.
*   **Memory Complexity:** $O(N)$ to store splat attributes. $O(P)$ for the image buffer.
*   **Performance:** Real-time performance is achievable on modern GPUs due to the simplicity of the Gaussian kernel and efficient rasterization.
*   **Strengths:** High visual quality, real-time rendering, compact representation.
*   **Limitations:** Large memory footprint for high-resolution scenes. Artifacts can occur with large splats or improper sorting.
*   **When to use:** Real-time visualization of large scenes, virtual reality, interactive applications.
*   **When not to use:** Scenes with very high frequency details that require many splats, or when memory is extremely constrained.

## Background
The `KHR_gaussian_splatting` extension was developed by Khronos Group to standardize the storage of 3D Gaussian splats in glTF assets [S1]. It builds on the 3D Gaussian Splatting for Real-Time Radiance Field Rendering paper [S1]. The extension aims to establish a solid foundation for integrating 3D Gaussian splatting into glTF assets, enabling future enhancements [S1].

## Sources
[S1] https://github.com/KhronosGroup/glTF/blob/main/extensions/2.0/Khronos/KHR_gaussian_splatting/README.md, https://github.com/KhronosGroup/glTF/blob/main/extensions/2.0/Khronos/KHR_gaussian_splatting/README.md, fetched 2026-10-30
