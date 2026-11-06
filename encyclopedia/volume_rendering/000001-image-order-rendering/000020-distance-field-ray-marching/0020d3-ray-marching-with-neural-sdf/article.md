# Ray Marching with Neural SDF

Also called: DeepSDF Rendering, Neural SDF Ray Marching

## Overview
Ray Marching with Neural SDF is a rendering technique that approximates the Space Distance Field (SDF) using a deep neural network and renders images by marching rays along the estimated distance field [S1, unofficial]. It replaces explicit geometric primitives or voxel grids with a learned implicit representation, allowing for the rendering of complex, continuous shapes defined by the network's output [S1, unofficial]. Unlike traditional sphere tracing which uses analytical SDFs, this method queries a neural network for distance values, enabling shape synthesis and editing through latent space manipulation [S1, unofficial].

## How it works
The method extends standard sphere tracing by substituting the analytical SDF query $f(\mathbf{p})$ with a neural network inference step [S1, unofficial]. The network takes a 3D coordinate and a latent code as input and outputs the signed distance to the surface [S1, unofficial].

**Inputs and Outputs:**
- **Inputs:** Camera parameters (position, direction vectors), a trained Neural SDF model (DeepSDF), a latent code representing the shape, and rendering parameters (max steps, precision, max distance).
- **Outputs:** A 2D image (height $\times$ width) representing the rendered scene.

**Data Structures:**
- **Camera:** Defined by position $\mathbf{o}$, width $W$, height $H$, field of view $fov$, and aspect ratio $ar = W/H$ [S1, unofficial]. Ray directions are precomputed as a tensor of shape $[W \cdot H, 3]$ [S1, unofficial].
- **Neural SDF Model:** A multi-layer perceptron (MLP) taking concatenated latent code $\mathbf{z}$ and coordinate $\mathbf{p}$ as input [S1, unofficial].
    - Architecture example: Input dimension $D_z + 3$, hidden layers (e.g., 256 units with ReLU), output dimension 1 [S1, unofficial].
- **Distances Tensor:** Shape $[W \cdot H, 1]$ or $[W \cdot H]$, initialized to zeros, tracking the accumulated distance $t$ for each ray [S1, unofficial].

**Parameters:**
- `max_steps`: Maximum number of ray marching iterations. Typical value: 30–40 [S1, unofficial].
- `epsilon`: Convergence threshold. Typical value: $10^{-3}$ [S1, unofficial].
- `max_dist`: Maximum travel distance. Typical value: 10.0 [S1, unofficial].

**Algorithm Steps:**
1.  **Initialization:** Create a tensor `distances` of size $N = W \cdot H$ initialized to 0.0 [S1, unofficial].
2.  **Ray Setup:** For each pixel $i$, calculate the ray origin $\mathbf{o}_i$ (camera position) and direction $\mathbf{d}_i$ [S1, unofficial].
3.  **Marching Loop:** Iterate $k$ from 0 to `max_steps` - 1:
    a.  Calculate current positions: $\mathbf{p}_i = \mathbf{o}_i + \text{distances}[i] \cdot \mathbf{d}_i$ for all $i$ [S1, unofficial].
    b.  Query the Neural SDF: For each unique position $\mathbf{p}_i$, compute $d_i = \text{NeuralSDF}(\mathbf{z}, \mathbf{p}_i)$ [S1, unofficial].
    c.  **Convergence Check:** If all $d_i < \epsilon$, break the loop [S1, unofficial].
    d.  **Step Update:** Update distances: $\text{distances}[i] \leftarrow \text{distances}[i] + d_i$ [S1, unofficial].
    e.  **Escape Check:** If all rays have either converged ($d_i < \epsilon$) or exceeded `max_dist` ($\text{distances}[i] > \text{max\_dist}$), break the loop [S1, unofficial].
4.  **Normal Estimation:** For rays that hit the surface (converged), estimate normals using central finite differences with step $\epsilon = 10^{-3}$ [S1, unofficial]:
    $$ \nabla f(\mathbf{p}) \approx \begin{bmatrix} f(\mathbf{p} + \epsilon \hat{x}) - f(\mathbf{p} - \epsilon \hat{x}) \\ f(\mathbf{p} + \epsilon \hat{y}) - f(\mathbf{p} - \epsilon \hat{y}) \\ f(\mathbf{p} + \epsilon \hat{z}) - f(\mathbf{p} - \epsilon \hat{z}) \end{bmatrix} $$
    Normalize the gradient to get $\mathbf{n}_i$ [S1, unofficial].
5.  **Shading:** Calculate brightness using a directional light $\mathbf{l}$: $b_i = \max(0, \mathbf{n}_i \cdot \mathbf{l})$ [S1, unofficial].
6.  **Clipping:** Set brightness $b_i = 0$ for pixels where $\text{distances}[i] > \text{max\_dist}$ [S1, unofficial].
7.  **Output:** Reshape brightness to $H \times W$, scale by 255, and convert to uint8 [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn

def render_neural_sdf(camera, sdf_model, latent_code, max_steps=30, epsilon=1e-3, max_dist=10.0):
    """
    Render an image using ray marching with a neural SDF.
    
    Args:
        camera: Object with attributes .position (3,), .directions (N, 3), .width, .height
        sdf_model: Neural network function taking (latent_code, coords) -> distances
        latent_code: Tensor of shape (D_z,)
        max_steps: int
        epsilon: float
        max_dist: float
        
    Returns:
        numpy array of shape (height, width, 3) uint8
    """
    device = camera.position.device
    dtype = camera.position.dtype
    N = camera.directions.shape[0]
    
    # Initialize distances tensor
    distances = torch.zeros(N, 1, device=device, dtype=dtype)
    
    # Precompute ray origins (same for all rays in this simple camera model)
    origins = camera.position.unsqueeze(0).expand(N, -1)
    
    # Ray Marching Loop
    for step in range(max_steps):
        # Calculate current positions
        positions = origins + distances * camera.directions
        
        # Query Neural SDF
        # Expand latent code to match batch size
        latent_batch = latent_code.unsqueeze(0).expand(N, -1)
        sdf_values = sdf_model(latent_batch, positions).squeeze(-1) # Shape (N,)
        
        # Check convergence: if distance is very small, we hit the surface
        hit_mask = sdf_values < epsilon
        
        # Check escape: if accumulated distance exceeds max_dist
        escaped_mask = distances.squeeze(-1) > max_dist
        
        # Combine masks for rays that are done
        done_mask = hit_mask | escaped_mask
        
        # If all rays are done, break
        if torch.all(done_mask):
            break
            
        # Update distances for rays that are not done
        # Note: In a strict implementation, only update rays that haven't hit/escaped
        # However, the source adds the step regardless, then checks bounds.
        # To be safe and consistent with source logic:
        distances[~done_mask] += sdf_values[~done_mask].unsqueeze(-1)
        
    # Final positions
    final_positions = origins + distances * camera.directions
    
    # Estimate Normals for hit points
    # Create a mask for rays that actually hit the surface (hit and not escaped)
    # Note: Source logic sets brightness to 0 if max_dist exceeded, implying 
    # normals might not be needed or valid for escaped rays.
    hit_and_valid = (distances.squeeze(-1) <= max_dist) & (sdf_values < epsilon) # Re-query or store last sdf?
    # The source re-queries or uses the last step's sdf. Let's assume we use the last computed sdf_values
    # But strictly, we need positions.
    
    # Estimate normals using finite differences
    eps = 1e-3
    dx = torch.tensor([eps, 0, 0], device=device, dtype=dtype).unsqueeze(0)
    dy = torch.tensor([0, eps, 0], device=device, dtype=dtype).unsqueeze(0)
    dz = torch.tensor([0, 0, eps], device=device, dtype=dtype).unsqueeze(0)
    
    normals = torch.zeros(N, 3, device=device, dtype=dtype)
    
    # Compute gradient components
    # Forward and backward points
    p_plus_x = final_positions + dx
    p_minus_x = final_positions - dx
    p_plus_y = final_positions + dy
    p_minus_y = final_positions - dy
    p_plus_z = final_positions + dz
    p_minus_z = final_positions - dz
    
    # Query SDF for all perturbed points
    # This is computationally expensive (6 queries per pixel)
    # Optimization: Only query for hit points? Source queries all.
    
    sdf_plus_x = sdf_model(latent_batch, p_plus_x).squeeze(-1)
    sdf_minus_x = sdf_model(latent_batch, p_minus_x).squeeze(-1)
    sdf_plus_y = sdf_model(latent_batch, p_plus_y).squeeze(-1)
    sdf_minus_y = sdf_model(latent_batch, p_minus_y).squeeze(-1)
    sdf_plus_z = sdf_model(latent_batch, p_plus_z).squeeze(-1)
    sdf_minus_z = sdf_model(latent_batch, p_minus_z).squeeze(-1)
    
    normals[:, 0] = sdf_plus_x - sdf_minus_x
    normals[:, 1] = sdf_plus_y - sdf_minus_y
    normals[:, 2] = sdf_plus_z - sdf_minus_z
    
    # Normalize normals
    norms = torch.norm(normals, dim=1, keepdim=True)
    norms = torch.clamp(norms, min=1e-5) # Avoid division by zero
    normals = normals / norms
    
    # Shading
    # Assume a single directional light from (0, 1, 0) normalized
    light_dir = torch.tensor([0, 1, 0], device=device, dtype=dtype).unsqueeze(0)
    light_dir = light_dir / torch.norm(light_dir)
    
    # Dot product
    dot_product = torch.sum(normals * light_dir, dim=1)
    brightness = torch.clamp(dot_product, 0, 1)
    
    # Set brightness to 0 for escaped rays
    brightness[distances.squeeze(-1) > max_dist] = 0
    
    # Reshape and convert to image
    # brightness is (N,) -> (H, W)
    brightness = brightness.reshape(camera.height, camera.width)
    
    # Convert to RGB (grayscale in this simple model)
    image = brightness.unsqueeze(-1).expand(-1, -1, 3)
    image = (image * 255).byte().cpu().numpy()
    
    return image
```

**Subtopics:**
- [Differentiable Sphere Tracing](2316-differentiable-sphere-tracing.md): Extends this method by allowing gradients to flow through the ray marching process, enabling end-to-end training of the SDF from images.
- [Acceleration and Optimization](6222-acceleration-and-optimization.md): Discusses techniques like hierarchical neural SDFs or BVH acceleration to speed up the neural queries.
- [Surface Point Sampling for SDFs](2708-surface-point-sampling-for-sdfs.md): Details how training data for the neural SDF is generated, which is critical for the quality of the rendered result.

## Operating characteristics
- **Time Complexity:** $O(N \cdot S \cdot Q)$, where $N$ is the number of pixels, $S$ is the average number of steps per ray, and $Q$ is the cost of a neural network forward pass. This is significantly slower than analytical SDFs due to the overhead of neural inference [S1, unofficial].
- **Memory Complexity:** $O(N)$ for storing distances and positions, plus the memory footprint of the neural network weights [S1, unofficial].
- **Performance:** Rendering speed is limited by the neural network inference time. On modern GPUs, a small MLP can be evaluated in microseconds, but for high-resolution images, it remains a bottleneck compared to voxel-based methods [S1, unofficial].
- **Strengths:** Ability to represent complex, continuous geometry without explicit mesh or voxel structures; compact storage of shape information in network weights; ease of shape editing via latent space [S1, unofficial].
- **Limitations:** High computational cost per pixel; potential for artifacts if the network is not well-trained; loss of high-frequency details if the network capacity is insufficient [S1, unofficial].
- **When to use:** When the shape is complex and defined implicitly, or when shape editing/synthesis is required. Not suitable for real-time rendering of simple scenes where analytical SDFs or meshes are available.

## Background
The method originates from the DeepSDF paper by Park et al. (2019), which introduced the use of deep neural networks to represent continuous signed distance functions for 3D shapes [S1, unofficial]. The specific implementation described in the notes is from a research project by Kulakov-Nikita, demonstrating the application of DeepSDF for image rendering via ray marching [S1, unofficial].

## Sources
[S1] Kulakov-Nikita/RayMarchingAI, https://github.com/Kulakov-Nikita/RayMarchingAI, fetched 2026-10-26 (unofficial)
