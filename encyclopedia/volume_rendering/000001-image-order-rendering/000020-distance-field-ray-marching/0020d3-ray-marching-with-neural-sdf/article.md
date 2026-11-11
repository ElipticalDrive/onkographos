# Ray Marching with Neural SDF

Also called: DeepSDF Rendering, Neural SDF Ray Marching

## Overview
Ray Marching with Neural SDF is a rendering technique that approximates the Space Distance Field (SDF) using a deep neural network and renders images by marching rays along the estimated distance field [S1, unofficial]. It substitutes the analytical distance query $f(\mathbf{p})$ with a neural network inference $N(\mathbf{z}, \mathbf{p})$, where $\mathbf{z}$ is a latent code representing the shape and $\mathbf{p}$ is the 3D query point [S1, unofficial]. The method builds on [Distance-field ray marching](21-distance-field-ray-marching.md) by replacing explicit geometric primitives with a learned implicit representation [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Camera parameters (origin, direction vectors), a trained neural SDF model (weights and latent code), lighting configuration, and rendering parameters (`max_steps`, `epsilon`, `max_dist`) [S1, unofficial].
*   **Outputs:** A 2D image tensor of shape `(height, width, 3)` representing the rendered scene [S1, unofficial].

**Data Structures:**
*   **Camera:** Defined by `width` (int), `height` (int), `fov` (float, default $\pi/3$), `position` (3D vector), and `aspect_ratio` (float, `width/height`) [S1, unofficial].
*   **Ray Directions:** A tensor of shape `(width * height, 3)` containing normalized direction vectors for each pixel, generated via coordinate grid sampling over $[-1, 1]$ [S1, unofficial].
*   **Distances Tensor:** Initialized to zeros with shape `(width * height, 1)` on the GPU device [S1, unofficial].
*   **Neural SDF Model:** A neural network taking concatenated latent code and 3D coordinates as input, outputting a scalar distance value [S1, unofficial].

**Parameters:**
*   `max_steps`: Maximum number of ray marching iterations. Default value is 30 [S1, unofficial].
*   `epsilon`: Convergence threshold. Default value is $10^{-3}$ [S1, unofficial].
*   `max_dist`: Maximum ray travel distance. Default value is 10.0 [S1, unofficial].

**Algorithm Steps:**
1.  **Initialization:** Create a distances tensor $D$ of shape $(N, 1)$ filled with zeros, where $N = \text{width} \times \text{height}$ [S1, unofficial]. Compute ray origins $\mathbf{o}$ (camera position) and directions $\mathbf{d}$ (one per pixel) [S1, unofficial].
2.  **Position Calculation:** Calculate initial ray positions $\mathbf{p}_0 = \mathbf{o} + D \cdot \mathbf{d}$ [S1, unofficial].
3.  **Marching Loop:** Iterate up to `max_steps` times:
    *   Query the neural SDF for distances at current positions $\mathbf{p}$: $d = N(\mathbf{z}, \mathbf{p})$ [S1, unofficial].
    *   **Termination Check:** If all values in $d$ are less than `epsilon`, break the loop (convergence) [S1, unofficial].
    *   **Step Update:** Update distances $D \leftarrow D + d$ [S1, unofficial].
    *   **Recalculate Positions:** Update positions $\mathbf{p} \leftarrow \mathbf{o} + D \cdot \mathbf{d}$ [S1, unofficial].
    *   **Escape Check:** If all rays have exceeded `max_dist` (check if $D > \text{max\_dist}$), break the loop [S1, unofficial].
4.  **Normal Estimation:** At the final converged positions, estimate normals using central finite differences with step size $e = 10^{-3}$ [S1, unofficial].
    *   Compute offsets: $\mathbf{dx} = (e, 0, 0)$, $\mathbf{dy} = (0, e, 0)$, $\mathbf{dz} = (0, 0, e)$ [S1, unofficial].
    *   Calculate gradient components:
        $$ n_x = \frac{N(\mathbf{z}, \mathbf{p} + \mathbf{dx}) - N(\mathbf{z}, \mathbf{p} - \mathbf{dx})}{2e} $$
        $$ n_y = \frac{N(\mathbf{z}, \mathbf{p} + \mathbf{dy}) - N(\mathbf{z}, \mathbf{p} - \mathbf{dy})}{2e} $$
        $$ n_z = \frac{N(\mathbf{z}, \mathbf{p} + \mathbf{dz}) - N(\mathbf{z}, \mathbf{p} - \mathbf{dz})}{2e} $$ [G]
    *   Normalize: $\mathbf{n} = \frac{(n_x, n_y, n_z)}{\|(n_x, n_y, n_z)\|}$ [S1, unofficial].
5.  **Shading:** Calculate brightness using a single directional light $\mathbf{l}$:
    *   Compute dot product: $b = \max(0, \mathbf{n} \cdot \mathbf{l})$ [S1, unofficial].
    *   Set brightness to 0 for pixels where the final accumulated distance exceeds `max_dist` [S1, unofficial].
6.  **Post-processing:** Reshape brightness to `(height, width)`, convert to numpy, scale by 255, and cast to `uint8` for the final image [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def render_neural_sdf(camera, sdf_model, latent_code, light_dir, max_steps=30, epsilon=1e-3, max_dist=10.0):
    """
    Renders an image using ray marching with a neural SDF.
    
    Args:
        camera: Object with width, height, position (3,), and directions (N, 3).
        sdf_model: Callable N(latent_code, coords) -> distances (N, 1).
        latent_code: Tensor/array representing the shape.
        light_dir: Normalized 3D vector for the light source.
        max_steps: Max iterations.
        epsilon: Convergence threshold.
        max_dist: Max ray distance.
        
    Returns:
        numpy array: Rendered image (uint8, shape HxWx3).
    """
    device = camera.position.device
    dtype = camera.position.dtype
    num_rays = camera.width * camera.height
    
    # 1. Initialize distances tensor
    distances = np.zeros((num_rays, 1), dtype=dtype)
    
    # 2. Calculate initial positions
    # directions shape: (num_rays, 3)
    positions = camera.position + distances * camera.directions
    
    # 3. Marching Loop
    for step in range(max_steps):
        # Query neural SDF
        sdf_values = sdf_model(latent_code, positions)
        
        # Termination Check: Convergence
        if np.all(sdf_values < epsilon):
            break
            
        # Step Update
        distances = distances + sdf_values
        
        # Recalculate positions
        positions = camera.position + distances * camera.directions
        
        # Escape Check
        if np.all(distances > max_dist):
            break
            
    # 4. Normal Estimation
    e = 1e-3
    dx = np.array([e, 0, 0], dtype=dtype)
    dy = np.array([0, e, 0], dtype=dtype)
    dz = np.array([0, 0, e], dtype=dtype)
    
    # Calculate finite differences
    sdf_plus_dx = sdf_model(latent_code, positions + dx)
    sdf_minus_dx = sdf_model(latent_code, positions - dx)
    sdf_plus_dy = sdf_model(latent_code, positions + dy)
    sdf_minus_dy = sdf_model(latent_code, positions - dy)
    sdf_plus_dz = sdf_model(latent_code, positions + dz)
    sdf_minus_dz = sdf_model(latent_code, positions - dz)
    
    nx = (sdf_plus_dx - sdf_minus_dx) / (2 * e)
    ny = (sdf_plus_dy - sdf_minus_dy) / (2 * e)
    nz = (sdf_plus_dz - sdf_minus_dz) / (2 * e)
    
    # Normalize normals
    norms = np.sqrt(nx**2 + ny**2 + nz**2)
    norms = np.clip(norms, 1e-8, None)
    normals = np.stack([nx, ny, nz], axis=-1) / norms.reshape(-1, 1)
    
    # 5. Shading
    # Dot product with light direction
    dot_product = np.sum(normals * light_dir, axis=-1)
    brightness = np.clip(dot_product, 0.0, 1.0)
    
    # Set brightness to 0 for rays that exceeded max_dist
    escaped = distances.squeeze(-1) > max_dist
    brightness[escaped] = 0.0
    
    # 6. Post-processing
    # Reshape to (height, width)
    brightness_image = brightness.reshape(camera.height, camera.width)
    
    # Convert to numpy, scale, cast to uint8
    img_np = brightness_image * 255
    img_uint8 = img_np.astype(np.uint8)
    
    # Convert to RGB (single channel brightness -> grayscale RGB)
    img_rgb = np.stack([img_uint8] * 3, axis=-1)
    
    return img_rgb
```

**Edge Cases and Pitfalls:**
*   **Normal Estimation Bug:** The reference implementation contains a hardcoded index for the SDF object (`objects[0]`), which fails if multiple objects are passed [S1, unofficial].
*   **Screen Initialization:** The PyGame screen initialization is hardcoded to 1280x720, which may not match the requested render dimensions [S1, unofficial].
*   **Precision:** The choice of `epsilon` affects convergence; too large a value leads to jagged surfaces, while too small increases computation time [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N \cdot S \cdot C)$, where $N$ is the number of pixels, $S$ is the number of steps (`max_steps`), and $C$ is the cost of a neural network forward pass [S1, unofficial].
*   **Memory Complexity:** $O(N)$ for storing ray states and distances [S1, unofficial].
*   **Performance:** Performance depends heavily on the neural network architecture and hardware [S1, unofficial].
*   **Strengths:** Compact representation of complex geometry via latent codes; no need for explicit mesh or voxel data [S1, unofficial].
*   **Limitations:** Slow rendering speed compared to analytical methods; requires training data and training time; normal estimation via finite differences is noisy and computationally expensive [S1, unofficial].
*   **When to Use:** When geometry is complex and defined implicitly by a neural network, or when storage of explicit geometry is prohibitive [S1, unofficial].
*   **When Not to Use:** Not given in the sources.

## Background
The technique originates from the application of deep learning to implicit surface representations, specifically building on the DeepSDF framework [S1, unofficial]. The provided source is an unofficial project exploring the approximation of SDFs using deep machine learning for ray marching rendering [S1, unofficial]. It demonstrates the feasibility of using a neural network to replace the analytical distance function in the sphere tracing algorithm [S1, unofficial].

## Sources
[S1] Kulakov-Nikita/RayMarchingAI, https://github.com/Kulakov-Nikita/RayMarchingAI, fetched 2026-10-26 (unofficial)
