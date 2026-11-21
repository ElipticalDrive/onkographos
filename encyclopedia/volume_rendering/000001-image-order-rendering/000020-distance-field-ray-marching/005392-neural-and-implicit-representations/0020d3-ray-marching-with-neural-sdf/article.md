# Ray Marching with Neural SDF
Also called: DeepSDF Rendering, Neural SDF Ray Marching

## Overview
Ray Marching with Neural SDF is a rendering technique that approximates the Signed Distance Field (SDF) using a deep neural network and renders images by marching rays along the estimated distance field [S1, unofficial]. It differs from traditional SDF rendering by replacing the analytical distance function with a learned implicit representation, allowing for the reconstruction of complex geometries from data rather than explicit modeling.

## How it works
**Inputs and Outputs:**
*   **Inputs:** A trained neural network model (typically a Multi-Layer Perceptron) representing the SDF, camera parameters (position, orientation, field of view, resolution), and lighting configuration.
*   **Outputs:** A 2D image (height $\times$ width) representing the rendered scene.

**Data Structures:**
*   **Camera:** Defined by `width` (int), `height` (int), `fov` (float, default $\pi/3$), `position` (3D vector), and `aspect_ratio` (float, `width/height`) [S1, unofficial].
*   **Ray Buffer:** A tensor of shape `[width * height, 1]` storing the current distance along each ray, initialized to zeros [S1, unofficial].
*   **Position Buffer:** A tensor of shape `[width * height, 3]` storing the 3D coordinates of the current ray sample points [S1, unofficial].

**Parameters:**
*   `max_steps`: Maximum number of ray marching iterations. Default is 30 [S1, unofficial].
*   `epsilon`: Convergence threshold. If the remaining distance is less than `epsilon`, the ray is considered to have hit the surface. Default is $10^{-3}$ [S1, unofficial].
*   `max_dist`: Maximum distance a ray can travel before being clipped. Default is 10.0 [S1, unofficial].

**Algorithm Steps:**
1.  **Initialization:** Create a distance tensor `distances` of shape `[width * height, 1]` initialized to zeros on the GPU [S1, unofficial].
2.  **Ray Setup:** Calculate ray directions for each pixel based on the camera's `fov`, `aspect_ratio`, and position. Normalize directions. Compute initial positions as $\mathbf{p}_0 = \mathbf{o} + \text{distances} \cdot \mathbf{d}$, which effectively sets initial positions to camera origins [S1, unofficial].
3.  **Marching Loop:** Iterate up to `max_steps` times:
    a.  **Query SDF:** Evaluate the neural network $f(\mathbf{p})$ at all current positions $\mathbf{p}$ to get signed distances [S1, unofficial].
    b.  **Convergence Check:** If all distances are less than `epsilon`, break the loop [S1, unofficial].
    c.  **Step Update:** Update distances by adding the sampled SDF value: $\text{distances} \leftarrow \text{distances} + f(\mathbf{p})$. Recalculate positions: $\mathbf{p} \leftarrow \mathbf{o} + \text{distances} \cdot \mathbf{d}$ [S1, unofficial].
    d.  **Clipping Check:** If all rays have either converged (distance < `epsilon`) or exceeded `max_dist`, break the loop [S1, unofficial].
4.  **Normal Estimation:** At the final positions, estimate normals using central finite differences with step size $e = 10^{-3}$ [S1, unofficial].
    *   Compute offsets: $\mathbf{dx} = (e, 0, 0)$, $\mathbf{dy} = (0, e, 0)$, $\mathbf{dz} = (0, 0, e)$ [S1, unofficial].
    *   Calculate normal components:
        $$ n_x = f(\mathbf{p} + \mathbf{dx}) - f(\mathbf{p} - \mathbf{dx}) $$
        $$ n_y = f(\mathbf{p} + \mathbf{dy}) - f(\mathbf{p} - \mathbf{dy}) $$
        $$ n_z = f(\mathbf{p} + \mathbf{dz}) - f(\mathbf{p} - \mathbf{dz}) $$
    *   Normalize the resulting vector $\mathbf{n} = (n_x, n_y, n_z)$ [S1, unofficial].
5.  **Shading:** Calculate brightness using a single directional light. Clamp the dot product of the normal $\mathbf{n}$ and light direction $\mathbf{l}$ to $[0, 1]$: $\text{brightness} = \max(0, \mathbf{n} \cdot \mathbf{l})$ [S1, unofficial].
6.  **Clipping:** Set brightness to 0 for pixels where the final distance exceeds `max_dist` [S1, unofficial].
7.  **Output:** Reshape brightness to height $\times$ width, convert to numpy, scale by 255, and cast to `uint8` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import numpy as np

def render_neural_sdf(camera, sdf_model, lights, max_steps=30, epsilon=1e-3, max_dist=10.0):
    """
    Renders an image using ray marching with a neural SDF.
    
    Args:
        camera: Camera object with width, height, fov, position, aspect_ratio.
        sdf_model: Trained neural network model (nn.Module) taking (latent_code, coords).
        lights: List of light sources. Assumes single directional light for simplicity.
        max_steps: Max iterations.
        epsilon: Convergence threshold.
        max_dist: Max ray distance.
        
    Returns:
        np.ndarray: Rendered image (H, W, 3) uint8.
    """
    device = next(sdf_model.parameters()).device
    dtype = next(sdf_model.parameters()).dtype
    
    # 1. Initialize distances
    num_pixels = camera.width * camera.height
    distances = torch.zeros((num_pixels, 1), device=device, dtype=dtype)
    
    # 2. Setup rays
    # Generate pixel coordinates
    y, x = torch.meshgrid(
        torch.linspace(-1, 1, camera.height, device=device, dtype=dtype),
        torch.linspace(-1, 1, camera.width, device=device, dtype=dtype),
        indexing='ij'
    )
    # Flatten and create direction vectors
    # Assuming camera looks down -Z or similar standard setup, adjust based on Camera implementation
    # From source: directions are normalized. 
    # Reconstructing direction logic from source snippet:
    # x, y are in [-1, 1]. z is 1.0.
    directions = torch.stack([x, y, torch.full_like(x, 1.0)], dim=-1)
    directions = torch.nn.functional.normalize(directions, dim=-1)
    directions = directions.reshape((num_pixels, 3))
    
    # Initial positions: origin + distances * direction (distances=0 initially)
    positions = camera.position.unsqueeze(0).expand(num_pixels, -1)
    
    # 3. Marching Loop
    for step in range(max_steps):
        # Query SDF
        # Note: Source uses a latent code. For rendering a fixed shape, 
        # the latent code is fixed. Here we assume latent_code is available or fixed.
        # In the source example, latent_code is generated during training.
        # For rendering, we need the specific latent code for the object.
        # Assuming 'latent_code' is passed or retrieved.
        # Placeholder for latent_code retrieval if not passed directly.
        # In the source 'render_to_file.py', the model and latent code are loaded.
        # We assume 'latent_code' is a tensor of shape (1, latent_dim) or (num_pixels, latent_dim)
        # The source code expands it: latent_code[:1].expand(points_torch.shape[0], -1)
        # Let's assume we have a single latent code for the scene.
        # This part depends on how the scene is defined. 
        # For a single object, latent_code is constant.
        
        # To make this self-contained, we assume latent_code is passed or fixed.
        # Since the prompt asks for the method, and the source code has 'FIX IT' comments
        # about multiple objects, we stick to the single object logic shown in the loop.
        
        # Evaluate SDF
        # Note: The source code snippet for rendering isn't fully provided, 
        # but the steps are described. 
        # We need to call sdf_model(latent_code, positions)
        # However, the source code for 'render_to_file.py' is not fully visible.
        # We infer the call from 'deepSDF.py' forward pass.
        
        # Let's assume 'latent_code' is a tensor of shape (1, 16)
        # And we expand it to match positions.
        # This is a gap filled by [G] as the exact latent code handling in render loop is not fully explicit in the provided snippets.
        # But the source says: "Query the neural SDF for distances at current positions."
        
        # We need to define 'latent_code' here. 
        # In the source, 'fit()' returns 'model, latent_code'.
        # So the renderer must receive this latent_code.
        # We will assume 'latent_code' is an argument or global.
        # To keep pseudocode self-contained, we add it as an argument.
        
        # Re-defining function signature for self-containment
        pass # Placeholder for actual implementation details not fully in source snippets
        
    # Since the full render loop code is not in the source snippets, 
    # we rely on the "step" notes.
    
    # Let's write the loop based on the notes strictly.
    
    # We need 'latent_code'. 
    # The source code 'deepSDF.py' generates it. 
    # The renderer uses it.
    # We will assume 'latent_code' is passed to the render function.
    
    # Re-writing the function to include latent_code
    pass

def render_neural_sdf_full(camera, sdf_model, latent_code, lights, max_steps=30, epsilon=1e-3, max_dist=10.0):
    device = next(sdf_model.parameters()).device
    dtype = next(sdf_model.parameters()).dtype
    
    num_pixels = camera.width * camera.height
    distances = torch.zeros((num_pixels, 1), device=device, dtype=dtype)
    
    y, x = torch.meshgrid(
        torch.linspace(-1, 1, camera.height, device=device, dtype=dtype),
        torch.linspace(-1, 1, camera.width, device=device, dtype=dtype),
        indexing='ij'
    )
    directions = torch.stack([x, y, torch.full_like(x, 1.0)], dim=-1)
    directions = torch.nn.functional.normalize(directions, dim=-1)
    directions = directions.reshape((num_pixels, 3))
    
    positions = camera.position.unsqueeze(0).expand(num_pixels, -1)
    
    # Expand latent_code to match number of pixels
    # latent_code shape: (1, latent_dim)
    latent_code_expanded = latent_code.expand(num_pixels, -1)
    
    for step in range(max_steps):
        # Query SDF
        # sdf_model forward: cat(latent_code, coords) -> model -> output
        # Note: Source code shows: x = torch.cat([latent_code, coords], dim=-1)
        # So we pass latent_code and positions.
        
        # Get SDF values
        # The source code uses: sdf.sdf(pos) which implies a wrapper.
        # We call the model directly.
        sdf_values = sdf_model(latent_code_expanded, positions) # Shape: (num_pixels, 1)
        
        # Check convergence
        if torch.all(sdf_values < epsilon):
            break
            
        # Update distances and positions
        distances = distances + sdf_values
        positions = camera.position.unsqueeze(0).expand(num_pixels, -1) + distances * directions
        
        # Check max_dist
        if torch.all(distances.squeeze() > max_dist):
            break
            
    # Estimate normals
    e = 1e-3
    dx = torch.tensor([e, 0, 0], device=device, dtype=dtype)
    dy = torch.tensor([0, e, 0], device=device, dtype=dtype)
    dz = torch.tensor([0, 0, e], device=device, dtype=dtype)
    
    # Calculate normal components
    # Note: Source code has a bug comment 'FIX IT !!!' regarding normal estimation using only first object.
    # We implement the logic as described.
    
    # We need to query SDF at perturbed positions
    # pos + dx, pos - dx, etc.
    
    # To avoid expanding latent_code repeatedly, we assume it's already expanded or handle it.
    # Let's expand once more for clarity.
    latent_code_expanded = latent_code.expand(num_pixels, -1)
    
    sdf_pos_dx = sdf_model(latent_code_expanded, positions + dx)
    sdf_pos_minus_dx = sdf_model(latent_code_expanded, positions - dx)
    sdf_pos_dy = sdf_model(latent_code_expanded, positions + dy)
    sdf_pos_minus_dy = sdf_model(latent_code_expanded, positions - dy)
    sdf_pos_dz = sdf_model(latent_code_expanded, positions + dz)
    sdf_pos_minus_dz = sdf_model(latent_code_expanded, positions - dz)
    
    nx = sdf_pos_dx - sdf_pos_minus_dx
    ny = sdf_pos_dy - sdf_pos_minus_dy
    nz = sdf_pos_dz - sdf_pos_minus_dz
    
    normals = torch.stack([nx, ny, nz], dim=-1)
    normals = torch.nn.functional.normalize(normals, dim=-1)
    
    # Shading
    # Assume single directional light 'lights[0]'
    # Light direction is assumed to be normalized.
    light_dir = lights[0].direction # Assuming light object has direction
    # Clamp dot product
    brightness = torch.clamp(torch.sum(normals * light_dir, dim=-1), 0.0, 1.0)
    
    # Clip rays that exceeded max_dist
    # We need to know which rays exceeded max_dist.
    # We can check the final distances.
    final_distances = distances.squeeze()
    brightness[final_distances > max_dist] = 0.0
    
    # Reshape and convert
    image = brightness.reshape(camera.height, camera.width)
    image_np = image.cpu().numpy()
    image_np = (image_np * 255).astype(np.uint8)
    
    # Convert to RGB (assuming grayscale or single channel)
    # Source converts to uint8, likely grayscale.
    # If RGB is needed, we'd need color from the SDF or material.
    # Source only calculates brightness.
    image_rgb = np.stack([image_np] * 3, axis=-1)
    
    return image_rgb
```

**Edge Cases and Pitfalls:**
*   **Normal Estimation Bug:** The reference implementation contains a hardcoded index for the first object (`objects[0]`) when estimating normals, which fails if multiple objects are present [S1, unofficial].
*   **Screen Initialization:** The PyGame screen initialization is hardcoded to 1280x720 in the reference implementation [S1, unofficial].
*   **Precision:** The choice of `epsilon` affects convergence. Too large a value leads to inaccurate surfaces; too small may cause unnecessary iterations or precision issues [S1, unofficial].
*   **Latent Code:** The method requires a valid latent code for the specific shape being rendered. Using an incorrect latent code will render the wrong geometry [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N \cdot M \cdot D)$, where $N$ is the number of pixels, $M$ is the number of marching steps, and $D$ is the cost of a neural network forward pass. The neural network evaluation is the dominant factor.
*   **Memory Complexity:** $O(N)$ for storing ray positions, distances, and normals on the GPU. The neural network weights are stored in VRAM.
*   **Performance:** Performance depends heavily on the neural network architecture and hardware. The reference implementation uses a simple MLP, which is fast but may lack detail.
*   **Strengths:** Can represent complex, organic shapes without explicit mesh data. Differentiable, allowing for gradient-based optimization of the geometry.
*   **Limitations:** Requires training data and training time. Rendering speed is slower than traditional rasterization or even traditional ray marching due to neural network inference. Quality depends on the training data and network capacity.
*   **When to Use:** When the geometry is defined by data (e.g., point clouds, scans) and needs to be rendered as a smooth surface. When differentiability is required for optimization.
*   **When Not to Use:** When real-time performance is critical and the geometry is simple enough for traditional methods. When training data is unavailable.

## Background
The technique is based on the concept of DeepSDF, which uses a deep neural network to represent a continuous signed distance field [S1, unofficial]. The specific implementation described in the sources is an unofficial project exploring the approximation of SDFs using deep machine learning for ray marching rendering [S1, unofficial].

## Sources
[S1] Kulakov-Nikita/RayMarchingAI, https://github.com/Kulakov-Nikita/RayMarchingAI, fetched 2026-10-26 (unofficial)
