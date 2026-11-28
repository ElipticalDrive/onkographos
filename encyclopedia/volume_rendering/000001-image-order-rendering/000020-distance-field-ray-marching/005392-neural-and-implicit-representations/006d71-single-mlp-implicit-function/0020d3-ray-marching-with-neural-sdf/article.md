# Ray Marching with Neural SDF
Also called: DeepSDF Rendering, Neural SDF Ray Marching

## Overview
Ray Marching with Neural SDF is a rendering technique that approximates the Space Distance Field (SDF) using a deep neural network and renders images by marching rays along the estimated distance field [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A trained neural network model representing the SDF, camera parameters, and lighting configuration [S1, unofficial].
*   **Outputs:** A 2D image representing the rendered scene [S1, unofficial].

**Data Structures:**
*   **Camera:** `width` (int), `height` (int), `fov` (float, default $\pi/3$), `position` (torch.Tensor), `aspect_ratio` (float, `width/height`) [S1, unofficial].
*   **Distances tensor:** Shape `[width * height, 1]`, initialized to zeros on the GPU [S1, unofficial].

**Parameters:**
*   `max_steps`: Maximum number of ray marching iterations. Default is 30 [S1, unofficial].
*   `epsilon`: Convergence threshold. If all distances are less than `epsilon`, the loop breaks [S1, unofficial]. Default is $10^{-3}$ [S1, unofficial].
*   `max_dist`: Maximum distance a ray can travel before being clipped. Default is 10.0 [S1, unofficial].

**Algorithm Steps:**
1. Initialize `distances` tensor to zeros with shape `[width * height, 1]` on the GPU [S1, unofficial].
2. Calculate ray origins from the camera and initial positions as $\mathbf{p} = \mathbf{o} + \text{distances} \cdot \mathbf{d}$ [S1, unofficial].
3. Loop up to `max_steps` times:
   a. Query the neural SDF for distances at current positions [S1, unofficial].
   b. If all distances are less than `epsilon`, break the loop [S1, unofficial].
   c. Update distances by adding the sampled SDF value and recalculate positions [S1, unofficial].
   d. Check if all rays have either converged (distance < `epsilon`) or exceeded `max_dist`. If so, break [S1, unofficial].
4. Estimate normals at the final positions using central finite differences with step size $e = 10^{-3}$ [S1, unofficial].
   * Compute offsets: $\mathbf{dx} = (e, 0, 0)$, $\mathbf{dy} = (0, e, 0)$, $\mathbf{dz} = (0, 0, e)$ [S1, unofficial].
   * Calculate normal components: $n_x = f(\mathbf{p} + \mathbf{dx}) - f(\mathbf{p} - \mathbf{dx})$, $n_y = f(\mathbf{p} + \mathbf{dy}) - f(\mathbf{p} - \mathbf{dy})$, $n_z = f(\mathbf{p} + \mathbf{dz}) - f(\mathbf{p} - \mathbf{dz})$ [S1, unofficial].
   * Normalize the resulting vector $\mathbf{n} = (n_x, n_y, n_z)$ [S1, unofficial].
5. Calculate brightness using a single directional light by clamping the dot product of normals and light direction to $[0, 1]$: $\text{brightness} = \max(0, \mathbf{n} \cdot \mathbf{l})$ [S1, unofficial].
6. Set brightness to 0 for pixels where the final distance exceeds `max_dist` [S1, unofficial].
7. Reshape brightness to height $\times$ width, convert to numpy, scale by 255, and cast to `uint8` for the final image [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import numpy as np

def render_neural_sdf(camera, sdf_model, latent_code, lights, max_steps=30, epsilon=1e-3, max_dist=10.0):
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
    latent_code_expanded = latent_code.expand(num_pixels, -1)
    
    for step in range(max_steps):
        sdf_values = sdf_model(latent_code_expanded, positions)
        
        if torch.all(sdf_values < epsilon):
            break
            
        distances = distances + sdf_values
        positions = camera.position.unsqueeze(0).expand(num_pixels, -1) + distances * directions
        
        if torch.all(distances.squeeze() > max_dist):
            break
            
    e = 1e-3
    dx = torch.tensor([e, 0, 0], device=device, dtype=dtype)
    dy = torch.tensor([0, e, 0], device=device, dtype=dtype)
    dz = torch.tensor([0, 0, e], device=device, dtype=dtype)
    
    nx = sdf_model(latent_code_expanded, positions + dx) - sdf_model(latent_code_expanded, positions - dx)
    ny = sdf_model(latent_code_expanded, positions + dy) - sdf_model(latent_code_expanded, positions - dy)
    nz = sdf_model(latent_code_expanded, positions + dz) - sdf_model(latent_code_expanded, positions - dz)
    
    normals = torch.nn.functional.normalize(torch.stack([nx, ny, nz], dim=-1), dim=-1)
    
    light_dir = lights[0].direction
    brightness = torch.clamp(torch.sum(normals * light_dir, dim=-1), 0.0, 1.0)
    
    final_distances = distances.squeeze()
    brightness[final_distances > max_dist] = 0.0
    
    image = brightness.reshape(camera.height, camera.width)
    image_np = image.cpu().numpy()
    image_np = (image_np * 255).astype(np.uint8)
    return np.stack([image_np] * 3, axis=-1)
```

**Edge Cases and Pitfalls:**
*   The reference implementation contains a hardcoded index for the first object (`objects[0]`) when estimating normals, which fails if multiple objects are present [S1, unofficial].
*   The PyGame screen initialization is hardcoded to 1280x720 in the reference implementation [S1, unofficial].

## Subtopics

### Differentiable Ray Marching (SDFs)
Reconstructs surfaces represented by Signed Distance Functions (SDFs) by making the ray marching and shading process differentiable [S2]. A neural network represents an SDF, outputting the distance to the nearest surface for any 3D point [S2]. A differentiable ray marcher steps along a ray, querying the SDF network to find the zero-crossing (the surface) [S2]. The surface normal is computed via automatic differentiation of the SDF network [S2]. A shading model (e.g., Phong) is applied, and the final pixel color is compared to the target image [S2]. Frameworks like NeuS use this for high-fidelity 3D shape reconstruction [S2].

## Operating characteristics
Not given in the sources.

## Background
The technique is based on the concept of DeepSDF, which uses a deep neural network to represent a continuous signed distance field [S1, unofficial]. The specific implementation described in the sources is an unofficial project exploring the approximation of SDFs using deep machine learning for ray marching rendering [S1, unofficial].

## Sources
[S1] Kulakov-Nikita/RayMarchingAI, https://github.com/Kulakov-Nikita/RayMarchingAI, fetched 2026-10-26 (unofficial)
[S2] Previous Differentiable Rendering Differentiable rendering is a framework that formulates the image synthesis process as a differentiable function of scene parameters, enabling gradient-based optimiza, https://inferensys.com/glossary/neural-radiance-fields-and-spatial-computing/neural-scene-representations/differentiable-rendering, fetched 2026-10-29
