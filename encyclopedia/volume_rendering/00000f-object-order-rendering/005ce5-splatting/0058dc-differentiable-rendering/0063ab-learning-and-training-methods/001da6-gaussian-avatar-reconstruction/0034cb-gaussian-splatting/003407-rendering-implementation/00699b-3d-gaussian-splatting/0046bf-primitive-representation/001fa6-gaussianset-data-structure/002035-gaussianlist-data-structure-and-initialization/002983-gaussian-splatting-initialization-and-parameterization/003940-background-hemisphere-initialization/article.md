# Background Hemisphere Initialization

Also called: Background Hemisphere Initialization and Optimization, Hemisphere Background Model

## Overview
Background Hemisphere Initialization is a technique for handling unbounded scenes in 3D Gaussian Splatting by modeling the distant background (sky) as a separate, fixed-radius hemisphere of Gaussians [S1]. It solves the problem of "cloud-like" artifacts and occlusion errors that occur when foreground Gaussians attempt to model distant sky regions, which degrades novel view synthesis quality [S1, S2, unofficial]. It sets itself apart from standard background models by explicitly disentangling the sky from the foreground geometry, often using a hemisphere to avoid occluding the ground plane [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of $K$ reference images with known camera poses, defining the foreground volume's midpoint.
*   **Output:** A set of background Gaussian primitives $\mathcal{G}_{bg}$ with defined positions, colors, scales, fixed rotations, and fixed opacities.

**Data Structures:**
The background is represented as a set of $N_{bg}$ 3D Gaussians. Each Gaussian $i$ has:
*   Position $\boldsymbol{\mu}_i$: Located on the surface of a hemisphere.
*   Color $\mathbf{c}_{bg}$: Regressed from projected 2D colors.
*   Scale $\mathbf{s}_{bg}$: Initial scale from KNN plus a learned residual.
*   Rotation $\mathbf{q}_{bg}$: Fixed unit quaternion $[1, 0, 0, 0]$.
*   Opacity $\alpha_{bg}$: Fixed value $1.0$.

**Steps:**
1.  **Hemisphere Placement:** Center a hemisphere at the midpoint of the foreground volume. Set a fixed radius $r_{bg} = 100$ meters [S1]. The hemisphere moves with the vehicle/camera such that the relative distance to the target camera remains fixed [S1].
2.  **Point Sampling:** Uniformly sample points over the hemisphere surface [S1].
3.  **Color Projection:** Project the sampled background points onto $K$ reference images to retrieve their 2D colors $\{\mathbf{c}_k\}_{k=1}^K$ [S1].
4.  **Scale Initialization:** Initialize the base Gaussian scales $\mathbf{s}_{bg}^{init}$ using the K-Nearest Neighbors (KNN) algorithm on the projected points [S1].
5.  **Parameter Regression:** Use a lightweight two-layer MLP $\mathcal{M}_{bg}$ to regress the color $\mathbf{c}_{bg}$ and the scale residual $\Delta \mathbf{s}_{bg}$ from the projected colors $\{\mathbf{c}_k\}_{k=1}^K$ [S1].
6.  **Final Parameter Assembly:** Compute the final scale by adding the residual to the initial scale. Set rotation to $[1, 0, 0, 0]$ and opacity to $1$ [S1].

**Formulas:**
The final scale $\mathbf{s}_{bg}$ is computed as:
$$ \mathbf{s}_{bg} = \Delta \mathbf{s}_{bg} + \mathbf{s}_{bg}^{init} $$
where the residual is:
$$ \Delta \mathbf{s}_{bg} = \mathcal{M}_{bg}(\{\mathbf{c}_k\}_{k=1}^K) $$
The color is directly regressed:
$$ \mathbf{c}_{bg} = \mathcal{M}_{bg}(\{\mathbf{c}_k\}_{k=1}^K) $$
Note: $\mathcal{M}_{bg}$ outputs both color and scale residuals [S1].

**Parameters:**
*   $r_{bg}$: Fixed radius of the background hemisphere. Value: $100$ meters [S1].
*   $K$: Number of reference images used for color projection [S1].

**Edge Cases and Pitfalls:**
*   **Occlusion:** Projecting points to reference images may retrieve inaccurate color information if the line of sight is occluded by foreground objects [S1]. The method relies on the assumption that the background is distant and largely unoccluded, or that the network can handle some noise.
*   **Fixed Parameters:** Rotation and opacity are not optimized. If the background requires varying opacity (e.g., semi-transparent clouds), this model may fail [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def background_hemisphere_initialization(foreground_volume_midpoint, reference_images, K, r_bg=100.0):
    """
    Initialize background Gaussians using a fixed-radius hemisphere model.
    
    Args:
        foreground_volume_midpoint: 3D vector, center of the foreground volume.
        reference_images: List of K images with known camera poses.
        K: Number of reference images.
        r_bg: Fixed radius of the hemisphere (default 100m).
        
    Returns:
        background_gaussians: List of Gaussian objects with pos, color, scale, rot, alpha.
    """
    # 1. Define Hemisphere Geometry
    # Center at foreground midpoint, radius r_bg
    hemi_center = foreground_volume_midpoint
    
    # 2. Sample Points on Hemisphere
    # Uniformly sample points on the hemisphere surface relative to the center
    # Note: The hemisphere orientation should face the camera/view direction
    # For simplicity, assume standard upward-facing hemisphere or oriented towards view
    sampled_points = sample_hemisphere_surface(hemi_center, r_bg)
    
    # 3. Project to Reference Images to get Colors
    projected_colors = []
    for i, point in enumerate(sampled_points):
        colors_k = []
        for k in range(K):
            cam_pose = reference_images[k].pose
            # Project point to image k
            # Handle occlusion? The source notes mention visibility maps but 
            # the core initialization step just retrieves colors.
            # [G] Assuming a simple projection that returns color if visible, else None
            color_k = project_and_query_color(point, cam_pose, reference_images[k].image)
            if color_k is not None:
                colors_k.append(color_k)
        projected_colors.append(colors_k)
        
    # 4. Initialize Scales via KNN
    # Initialize base scales based on local density of points
    s_bg_init = compute_knn_scales(sampled_points)
    
    # 5. Regress Parameters using MLP M_bg
    # M_bg is a lightweight two-layer MLP
    # Input: Concatenated colors from K images for each point
    # Output: Color c_bg and Scale Residual Delta_s_bg
    
    # Prepare input features: For each point, flatten the K colors
    inputs = []
    for colors_k in projected_colors:
        # Pad or handle missing views if necessary
        # [G] Assuming all K views are available or padded with zeros
        feature_vector = flatten_colors(colors_k)
        inputs.append(feature_vector)
        
    inputs_tensor = torch.stack(inputs)
    
    # Forward pass through M_bg
    # M_bg outputs [Color, Scale_Residual]
    outputs = M_bg(inputs_tensor)
    
    # Split outputs
    c_bg = outputs[:, :3]  # Assuming RGB
    delta_s_bg = outputs[:, 3:6] # Assuming 3D scale vector
    
    # 6. Compute Final Scales
    s_bg = s_bg_init + delta_s_bg
    
    # 7. Set Fixed Rotation and Opacity
    # Rotation: Unit quaternion [1, 0, 0, 0]
    # Opacity: 1.0
    q_bg = torch.tensor([1.0, 0.0, 0.0, 0.0])
    alpha_bg = 1.0
    
    # 8. Construct Gaussians
    background_gaussians = []
    for i in range(len(sampled_points)):
        g = Gaussian(
            pos=sampled_points[i],
            color=c_bg[i],
            scale=s_bg[i],
            rotation=q_bg,
            opacity=alpha_bg
        )
        background_gaussians.append(g)
        
    return background_gaussians
```

## Subtopics

### Generalizable Background Model
A variant where the hemisphere model is designed to be generalizable across scenes. It uniformly samples points over a hemisphere with fixed radius $r_{bg}$ that moves along with the camera motion. It employs a lightweight two-layer MLP $\mathcal{M}_{bg}$ to learn color $\mathbf{c}_{bg}$ and scale $\mathbf{s}_{bg}$ from projected colors $\{\mathbf{c}_k\}_{k=1}^K$ [S1]. The formula is $\mathbf{c}_{bg}, \mathbf{s}_{bg} = \mathcal{M}_{bg}(\{\mathbf{c}_k\}_{k=1}^K)$ [S1]. Rotation and opacity remain fixed parameters [S1].

## Operating characteristics
*   **Time Complexity:** Depends on the number of sampled points $N_{bg}$ and the number of reference images $K$. Projection and MLP inference are linear in $N_{bg}$ and $K$.
*   **Memory Complexity:** $O(N_{bg})$ to store the background Gaussians.
*   **Strengths:** Efficiently handles distant backgrounds without requiring foreground Gaussians to extend to infinity, reducing artifacts. Keeps background rendering efficient via standard 3DGS rasterization [S1].
*   **Limitations:** Assumes the background is distant and static relative to the camera motion. Fixed opacity and rotation limit flexibility for complex sky conditions (e.g., varying cloud density) [S1].
*   **When to use:** Urban view synthesis or autonomous driving scenarios where the background (sky, distant buildings) is hundreds of meters away and relatively stable [S1].

## Background
Originated in **EVolSplat: Efficient Volume-based Gaussian Splatting for Urban View Synthesis** (2025) [S1]. It addresses the limitations of standard 3D Gaussian Splatting in unbounded urban environments, where foreground Gaussians struggle to represent distant sky regions, leading to "cloud-like" artifacts [S1, S2, unofficial]. The method was developed to enable efficient reconstruction of full urban scenes by separating foreground and background representations [S1].

## Sources
[S1] EVolSplat: Efficient Volume-based Gaussian Splatting for Urban View Synthesis, https://doi.org/10.1109/CVPR52734.2025.01054, fetched 2026-11-03
