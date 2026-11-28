# Texture-Space Sampling

Also called: Texture-space batch sampling, UV-space sampling for training

## Overview
Texture-space sampling is a method for forming training batches in differentiable inverse volume rendering and radiance-field optimization by sampling unknowns (texture variables) uniformly directly in texture space, rather than sampling a random subset of views [S1]. This approach reduces gradient noise by ensuring that batches of observations are directly relevant to the selected unknowns, providing even coverage within an iteration compared to standard image-based sampling [S1]. It sets itself apart from [Batch Ray Sampling](2767-batch-ray-sampling.md) by prioritizing the uniformity of the latent representation (texture) over the uniformity of the input data (images), and from [Texture-Space Differentiable Rendering](5555-texture-space-differentiable-rendering.md) by focusing on the batch formation strategy rather than the gradient computation mechanics alone.

## How it works
The method constructs a training batch by selecting texture variables first, then determining which camera observations are relevant to those variables.

**Inputs:**
*   Texture space representation (e.g., UV map of a mesh or implicit surface).
*   Inverse UV mapping function $U^{-1}: (u, v) \rightarrow \mathbf{p}$, mapping texture coordinates to 3D surface positions.
*   Set of reference camera views (intrinsics, extrinsics, and RGB frames).
*   Batch size $N$, adjustable based on GPU memory.

**Outputs:**
*   A batch of rays connecting sampled 3D positions to reference camera centers.
*   Corresponding target pixel values from reference frames.
*   Gradients for the texture parameters.

**Data Structures:**
*   **Sampled Points:** A list of $N$ texture coordinate pairs $(u_i, v_i)$.
*   **3D Positions:** A list of $N$ 3D points $\mathbf{p}_i = U^{-1}(u_i, v_i)$.
*   **Valid Rays:** A subset of rays $(\mathbf{o}_j, \mathbf{d}_j)$ where $\mathbf{o}_j$ is the camera center and $\mathbf{d}_j$ is the direction to $\mathbf{p}_i$, filtered for visibility.

**Steps:**
1.  **Sample Texture Coordinates:** At the start of each iteration, select $N$ points uniformly at random over the texture space domain $[0, 1] \times [0, 1]$. Let these be $\{(u_i, v_i)\}_{i=1}^N$ [S1].
2.  **Map to 3D:** Use the precomputed inverse UV mapping to lookup the corresponding 3D positions on the scene surfaces: $\mathbf{p}_i = U^{-1}(u_i, v_i)$ [S1].
3.  **Connect to Views:** For each sampled 3D position $\mathbf{p}_i$, determine connections to a set of reference view positions. This involves checking if $\mathbf{p}_i$ is visible from the camera frustum and not occluded by other geometry [S1].
4.  **Filter Occlusions:** Discard connections where the ray from the camera center to $\mathbf{p}_i$ is occluded by geometry or where $\mathbf{p}_i$ falls outside the camera’s frustum [S1].
5.  **Fetch Targets:** For the remaining valid 3D positions, fetch the corresponding pixel values from the reference RGB frames [S1].
6.  **Render:** Estimate the current radiance values for the batch using differentiable path tracing [S1].
7.  **Compute Loss and Gradients:** Compute the per-pixel loss (averaged over all rays in the batch). Backpropagate through the rendering algorithm to obtain gradients [S1].
8.  **Update:** Update scene parameters (texture values) with an optimizer step [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def texture_space_sampling_iteration(texture_map, inverse_uv_map, cameras, rgb_frames, batch_size, optimizer):
    """
    Performs one iteration of texture-space sampling.
    
    Args:
        texture_map: Current state of texture parameters.
        inverse_uv_map: Function mapping (u, v) to 3D point p.
        cameras: List of camera objects with intrinsics, extrinsics, and RGB frames.
        rgb_frames: List of reference RGB images corresponding to cameras.
        batch_size: Number of texture samples to draw.
        optimizer: Optimization object with step method.
        
    Returns:
        Updated texture_map.
    """
    # Step 1: Sample uniform texture coordinates
    u_samples = random.uniform(0, 1, batch_size)
    v_samples = random.uniform(0, 1, batch_size)
    texture_coords = list(zip(u_samples, v_samples))
    
    # Step 2: Map to 3D positions
    # Note: inverse_uv_map may return None for invalid UVs (e.g., holes in mesh)
    points_3d = []
    valid_indices = []
    for i, (u, v) in enumerate(texture_coords):
        p = inverse_uv_map(u, v)
        if p is not None:
            points_3d.append(p)
            valid_indices.append(i)
            
    # If no valid points, skip iteration or reduce batch size
    if not points_3d:
        return texture_map
        
    points_3d = np.array(points_3d)
    
    # Step 3 & 4: Connect to views and filter occlusions
    # For each point, find visible cameras
    rays = []
    targets = []
    
    for p in points_3d:
        for camera, frame in zip(cameras, rgb_frames):
            # Check if point is in frustum
            if not camera.is_in_frustum(p):
                continue
                
            # Check for occlusion
            # Ray from camera center to p
            ray_origin = camera.center
            ray_direction = p - ray_origin
            ray_direction = ray_direction / np.linalg.norm(ray_direction)
            
            if not is_occluded(ray_origin, ray_direction, p, scene_geometry):
                # Fetch pixel value
                # Project p to image plane to get pixel coordinates
                pixel_coords = camera.project(p)
                if pixel_coords is valid_pixel(pixel_coords, frame.shape):
                    target_value = frame[pixel_coords]
                    rays.append((ray_origin, ray_direction))
                    targets.append(target_value)
                    
    if not rays:
        return texture_map
        
    rays = np.array(rays)
    targets = np.array(targets)
    
    # Step 6: Differentiable rendering
    # Estimate radiance for the batch
    # Note: This assumes a differentiable renderer that can handle batched rays
    rendered_radiance = differentiable_path_trace(rays, texture_map)
    
    # Step 7: Loss and Gradients
    loss = mean_squared_error(rendered_radiance, targets)
    gradients = compute_gradients(loss, texture_map)
    
    # Step 8: Update
    optimizer.step(gradients)
    
    return texture_map

def is_occluded(origin, direction, target, geometry):
    """
    Checks if the ray from origin to target is occluded by geometry.
    Returns True if occluded, False if visible.
    """
    # Simple ray intersection test
    # In a real implementation, this would use a BVH or similar structure
    closest_hit = geometry.ray_intersect(origin, direction)
    if closest_hit is None:
        return False
    distance_to_hit = np.linalg.norm(closest_hit - origin)
    distance_to_target = np.linalg.norm(target - origin)
    
    # If the hit is significantly closer than the target, it's occluded
    # Use a small epsilon to avoid self-intersection issues
    return distance_to_hit < (distance_to_target - 1e-6)
```

**Parameters:**
*   **Batch Size ($N$):** The number of texture samples per iteration. Adjustable based on available GPU memory [S1].
*   **Inverse UV Mapping:** Precomputed mapping from texture space to 3D surface. Must handle degenerate cases (e.g., UVs outside mesh bounds) [S1].

**Edge Cases and Pitfalls:**
*   **Invalid UVs:** Some texture locations may not map to any scene surface (e.g., holes in the mesh or UV islands outside the geometry). These result in zero gradients for those locations [S1]. The implementation must handle `None` or invalid returns from the inverse UV map.
*   **Occlusion:** Connections discarded due to occlusion reduce the effective batch size. If too many are discarded, the gradient signal may become weak.
*   **Frustum Culling:** Points outside the camera frustum are discarded, ensuring only relevant observations are used [S1].

## Subtopics

### Texture-space Differentiable Rendering
A rendering and optimization approach where sampling and gradient computation occur in texture space rather than image space, ensuring even coverage and avoiding Jacobian factors that cause exploding gradients or uneven convergence [S1]. It selects texels uniformly at random and connects them to camera positions to form rays for path tracing [S1]. Gradients are computed by omitting Jacobian factors (the derivative of the mapping from texture to image space) to prevent exploding gradients when distance $d$ approaches zero and to obtain gradients of comparable magnitude regardless of observation distance or angle [S1]. Unlike standard image-based differentiable rendering, which leads to nonzero gradients only within the frusta of cameras selected for the iteration, texture-space sampling provides even coverage within an iteration [S1]. Some texture locations may not be mapped to any scene surface, resulting in zero gradients for those locations [S1].

## Operating characteristics
*   **Time Complexity:** Dominated by the differentiable path tracing step for the batch size $N$. The UV lookup and visibility checks are $O(N)$ but depend on the complexity of the scene geometry for occlusion testing.
*   **Memory Complexity:** $O(N)$ for storing the batch of rays and targets, plus the memory for the texture map and scene geometry.
*   **Performance:** More efficient than sampling a random subset of views because it reduces noise in gradients by proceeding with batches of observations directly relevant to the selected unknowns [S1].
*   **Strengths:** Even coverage of texture space within an iteration; avoids exploding gradients from Jacobian factors; reduces gradient noise.
*   **Limitations:** Requires a precomputed inverse UV mapping; some texture locations may have zero gradients if not mapped to surface; performance depends on the efficiency of the differentiable path tracer.
*   **When to Use:** When optimizing texture parameters for a scene with known geometry (e.g., mesh-based rendering); when even coverage of the texture space is desired.
*   **When Not to Use:** When the scene geometry is unknown or implicit; when memory is extremely limited and $N$ must be very small.

## Background
Originated in the context of material and lighting reconstruction for complex indoor scenes. The method was proposed to address the inefficiencies of standard image-based differentiable rendering, where gradients are sparse and unevenly distributed [S1]. It builds on the idea of sampling unknowns directly in their native space (texture space) to improve optimization stability and convergence [S1].

## Sources
[S1] Material and Lighting Reconstruction for Complex Indoor Scenes with Texture-space Differentiable Rendering, https://doi.org/10.2312/sr.20211292, fetched 2026-10-29
