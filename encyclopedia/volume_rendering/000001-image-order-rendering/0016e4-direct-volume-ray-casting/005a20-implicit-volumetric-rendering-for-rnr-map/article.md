# Implicit Volumetric Rendering for RNR-Map

Also called: GSN-style rendering, RNR-Map rendering

## Overview
Implicit Volumetric Rendering for RNR-Map is a technique used in the RNR-Map decoder to compute pixel colors by integrating occupancy and appearance along rays, modified from GSN to render individual pixels efficiently without upsampling [S1]. It differs from its parent [Direct volume ray casting](19-direct-volume-ray-casting.md) by using a neural decoder to map latent features to optical properties and by targeting sparse pixel rendering for navigation tasks rather than full-image synthesis [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:**
    *   Query pose $\mathbf{p}$ (camera position and orientation) [S1].
    *   Camera intrinsics [S1].
    *   RNR-Map $m$, a tensor of size $128 \times 128 \times 32$ [S1].
    *   Ray $\mathbf{r}$ defined by pixel coordinates $(h, w)$ [S1].
*   **Outputs:**
    *   Rendered color $c(\mathbf{r}, m)$ for the ray [S1].

**Data Structures:**
*   **RNR-Map ($m$):** A 3D tensor with dimensions $128 \times 128 \times 32$ [S1].
    *   Spatial dimensions ($128 \times 128$) represent a $32\text{m} \times 32\text{m}$ area [S1].
    *   Each grid cell represents a physical region of $25\text{cm} \times 25\text{cm}$ [S1].
    *   The depth dimension (32) is the feature dimension $D$ [S1].
*   **Latent Code ($q$):** A vector of size $D=32$ retrieved from $m$ via bilinear interpolation [S1].

**Steps:**
1.  **Ray Definition:** Given query pose $\mathbf{p}$ and camera intrinsics, determine the 3D position corresponding to pixel $(h, w)$ [S1]. Define a ray $\mathbf{r}(u)$ passing through this pixel, where $u$ is the distance from the camera center along the ray [S1].
2.  **Sampling:** Sample 3D points along the ray segment $[u_n, u_f]$ (near and far clipping planes) [S1].
3.  **Feature Retrieval:** For each sampled 3D point, calculate its position in the RNR-Map coordinate system [S1]. Retrieve the corresponding latent feature $q$ from the map $m$ using bilinear interpolation, as map positions have continuous values [S1].
4.  **Local Coordinate Transformation:** Compute the local pose $\mathbf{p}'$ (relative pose in a grid) for the sampled point [S1]. Determine the view direction $\mathbf{d}$ [S1].
5.  **Decoding:** Pass $\mathbf{p}'$, $\mathbf{d}$, and $q$ into the decoder network [S1].
    *   The latent code $q$ modulates a linear layer [S1].
    *   The decoder outputs occupancy $\sigma$ and appearance $\mathbf{a} \in \mathbb{R}^3$ for the sampled point [S1].
6.  **Integration:** Calculate the rendered color $c(\mathbf{r}, m)$ by integrating the product of transmittance, occupancy, and appearance along the ray [S1].

**Formulas:**
The rendered color is computed as [S1]:
$$
c(\mathbf{r},m) = \int_{u_n}^{u_f} Tr(u) \sigma(\mathbf{r}(u),q)\mathbf{a}(\mathbf{r}(u), \mathbf{d}, q) du
$$
where transmittance $Tr(u)$ is defined as [S1]:
$$
Tr(u) = \mathrm{exp} \left( -\int_{u_n}^{u} \sigma(\mathbf{r}(u'), q) du' \right)
$$
Here, $\mathbf{r}$ is the ray, $m$ is the map, $u$ is the distance along the ray, $u_n$ and $u_f$ are near and far clipping planes, $\sigma$ is occupancy, and $\mathbf{a}$ is appearance [S1].

**Parameters:**
*   Image resolution: $128 \times 128$ pixels [S1].
*   RNR-Map tensor size: $128 \times 128 \times 32$ [S1].
*   Grid cell physical size: $25\text{cm} \times 25\text{cm}$ [S1].

**Pitfalls:**
*   Unobserved regions are rendered as empty space [S1].
*   Small objects and texture details are often ignored [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_rnr_map(ray_origin, ray_direction, camera_intrinsics, query_pose, rnr_map):
    """
    Renders a single ray using the RNR-Map implicit volumetric rendering method.
    
    Args:
        ray_origin: 3D vector, camera center.
        ray_direction: 3D vector, viewing direction for pixel (h, w).
        camera_intrinsics: Camera matrix parameters.
        query_pose: Pose of the query camera.
        rnr_map: Tensor of shape (128, 128, 32).
        
    Returns:
        color: 3D vector, rendered color.
    """
    # 1. Define ray segment [u_n, u_f]
    # Intersection with volume bounding box to find u_n and u_f
    u_n, u_f = intersect_ray_bbox(ray_origin, ray_direction)
    
    # 2. Sample points along the ray
    # Number of samples N is not explicitly given, but integration is required
    # We assume a discrete approximation with N samples
    N = 64 # Placeholder for implementation detail not in source
    u_samples = linspace(u_n, u_f, N)
    
    color_accum = 0.0
    transmittance_accum = 1.0
    
    for i, u in enumerate(u_samples):
        # 3. Calculate 3D point on ray
        point_3d = ray_origin + u * ray_direction
        
        # 4. Calculate map position and retrieve latent code q
        # Map position calculation depends on coordinate system alignment
        # Assuming a standard transformation from world to map grid
        map_pos = world_to_map_grid(point_3d)
        
        # Bilinear interpolation to get q from rnr_map
        q = bilinear_interpolate(rnr_map, map_pos)
        
        # 5. Compute local pose p' and view direction d
        # Local pose is relative pose in a grid
        p_prime = compute_local_pose(point_3d, query_pose)
        d = ray_direction # View direction
        
        # 6. Decode occupancy and appearance
        # Decoder network structure adopted from GSN
        # q modulates a linear layer
        sigma, appearance = decoder_network(p_prime, d, q)
        
        # 7. Integrate
        # Transmittance calculation
        # Note: In discrete form, we update transmittance based on previous sigma
        # Tr(u) = exp(-integral from u_n to u of sigma)
        # We approximate the integral incrementally
        
        # Calculate transmittance at current sample
        # Tr(u_i) = Tr(u_{i-1}) * exp(-sigma_i * delta_u)
        # However, the formula defines Tr(u) as the integral up to u
        # Standard front-to-back compositing uses accumulated transmittance
        
        # delta_u is step size
        delta_u = (u_f - u_n) / N
        
        # Update transmittance for the current segment
        # T_i = exp(-integral_{u_n}^{u_i} sigma)
        # We can maintain an accumulated integral of sigma
        # Or use the standard discrete approximation:
        # alpha_i = 1 - exp(-sigma_i * delta_u)
        # T_i = product_{j=0}^{i-1} (1 - alpha_j)
        
        # Using the explicit integral form from source:
        # c(r) = integral Tr(u) * sigma(u) * a(u) du
        
        # Discrete approximation:
        # color += T_current * sigma * appearance * delta_u
        
        # Calculate current transmittance T(u)
        # T(u) = exp(-sum_{j=0}^{i-1} sigma_j * delta_u)
        # We need to track the accumulated sigma integral
        if i == 0:
            accumulated_sigma_integral = 0.0
        else:
            accumulated_sigma_integral += sigma * delta_u
            
        T_u = exp(-accumulated_sigma_integral)
        
        # Add contribution
        color_accum += T_u * sigma * appearance * delta_u
        
        # Update accumulated transmittance for next step if needed for alpha
        # But here we integrate directly
        
    return color_accum
```
*Note: The pseudocode above implements the integral equation directly. The source states GSN renders a small-size feature map and upsamples, while RNR-Map renders individual pixels. The decoder structure is from GSN [S1].*

## Subtopics

### GSN (Global Scene Network)
The parent method from which RNR-Map is modified. GSN renders a small-size feature map and upsamples it using a convolutional network to generate a realistic full image [S1]. Choose GSN when full-image synthesis is required for scene evaluation [S1]. [GSN](1576-neural-volume-rendering.md) (Link placeholder, assuming GSN is a sibling or parent concept in the broader tree, but here it is the direct reference).

### RNR-Map
The specific implementation for visual navigation. It renders individual pixels efficiently without upsampling, suitable for sparse pixel calculation in navigation tasks. Choose RNR-Map for image-based localization and navigation where computational efficiency for specific pixels is prioritized over full-image quality. [RNR-Map](1576-neural-volume-rendering.md) (Link placeholder).

## Operating characteristics
*   **Time Complexity:** Depends on the number of samples along the ray and the cost of the decoder network. The source does not provide explicit complexity analysis.
*   **Memory Complexity:** The RNR-Map tensor size is $128 \times 128 \times 32$ [S1].
*   **Measured Performance:** The method is used in image-goal navigation tasks [S1]. It can reconstruct images from unseen environments and arbitrary scenes [S1].
*   **Strengths:** Efficient calculation for a small subset of pixels; can embed and reconstruct images from arbitrary scenes; sufficient for accurate camera pose inference [S1].
*   **Limitations:** Unobserved regions are rendered as empty space; small objects and texture details are often ignored; image quality is not state-of-the-art [S1].
*   **When to use:** Image-based localization and navigation tasks where efficient rendering of specific pixels is needed [S1].
*   **When not to use:** When high-fidelity full-image synthesis is required [S1].

## Background
The method was introduced in the paper "Renderable Neural Radiance Map for Visual Navigation" [S1]. It is based on the Global Scene Network (GSN) [S1]. The RNR-Map is used for image-based localization, where it is convolved with a target RNR-Map to infer camera pose [S1]. The technique allows for the reconstruction of images from arbitrary scenes and novel views [S1].

## Sources
[S1] Renderable Neural Radiance Map for Visual Navigation, https://doi.org/10.1109/cvpr52729.2023.00878, fetched 2026-10-26
