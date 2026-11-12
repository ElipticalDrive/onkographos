# Physics-based Pseudo Label Generation

Also called: Post-processing of Stage 1, Pseudo reflectance generation, Pseudo shading generation

## Overview
This method generates high-fidelity pseudo reflectance and shading labels from a trained multi-light NeRF model to guide intrinsic decomposition training in Stage 2 [S1]. Unlike previous statistical priors, this physics-based pipeline provides a more reliable basis for disentanglement by leveraging explicit geometric and illumination models [S1]. It serves as the bridge between the implicit representation learned in Stage 1 and the intrinsic-aware NeRF in Stage 2 [S1].

## How it works
**Inputs:**
- Trained SDF network and RGB MLP from Stage 1 (MLI-NeRF).
- Camera poses and light positions for multiple lighting conditions.
- Rendered RGB images $I$ for each light condition.

**Outputs:**
- Pseudo shading image $S$.
- Pseudo reflectance image $R$.
- Weight maps $W_R$ and $W_S$ for reliability estimation.

**Data Structures:**
- **Points:** 3D coordinates $(x, y, z)$ sampled along camera rays.
- **Normals:** Surface normal vectors $\vec{N}$ derived from SDF gradients.
- **Light Visibility:** Binary or continuous mask $V$ indicating if a point is illuminated by a specific light source.
- **Images:** 2D arrays of RGB values for $I$, $S$, and $R$.

**Steps:**

1.  **Obtain Normal and Light Visibility:**
    Use the trained SDF network to perform sphere tracing and volume rendering. For each point along the camera ray, compute the surface normal $\vec{N}$ via the gradient of the SDF. Determine light visibility $V$ by checking if the point is occluded from the light source [S1].

2.  **Generate Pseudo Shading:**
    Compute the dot product of the normal $\vec{N}$ and the light ray direction $\vec{L}$. Apply gamma correction to adapt the pseudo-shading to the nonlinear representation of digital images, mimicking sensor response [S1].
    
    The shading $S$ is calculated as:
    $$ S = (\max(\vec{N} \cdot \vec{L}, 0) \otimes V)^\gamma $$
    where:
    - $\vec{N}$ is the normal ray.
    - $\vec{L}$ is the light ray direction.
    - $V$ is the light visibility mask.
    - $\gamma$ is the gamma value (typically 2.2 for sRGB).
    - $\otimes$ denotes pixel-wise product.
    - $(\cdot)^\gamma$ represents gamma correction [S1].

3.  **Generate Pseudo Reflectance:**
    Calculate initial reflectance for each light condition using pixel-wise division, ignoring the residual component (specular and indirect light) [S1]:
    $$ R = I \oslash S $$
    where $\oslash$ denotes pixel-wise division.
    
    To handle multiple light conditions, use K-means clustering at the pixel level. Incorporate weights of each pseudo shading to select the most probable pseudo reflectance, creating a merged reflectance image [S1].

4.  **Hole Filling:**
    Identify regions in the merged reflectance that appear as holes due to the absence of direct illumination in all lighting conditions. Compute a weighted distance between hole pixels and non-hole pixels based on:
    - Spatial distance in the image.
    - Angular difference of their normals.
    - RGB color difference.
    
    Assign the color of the nearest non-hole pixel (based on weighted distance) to the hole pixel. This is a rough supplementary approach with minimal impact on overall results, as hole regions are typically small under multi-light settings [S1].

5.  **Compute Weight Maps:**
    Compute weight maps $W_R$ and $W_S$ for pseudo reflectance and shading respectively. Weights are based on edges of pseudo shading and visibility. Areas with higher pseudo shading values or further from visibility edges have greater credibility. Areas closer to visibility edges or with lower pseudo shading values are deemed less reliable [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_pseudo_labels(sdf_network, rgb_mlp, camera_poses, light_positions, gamma=2.2):
    """
    Generates pseudo shading and reflectance labels from a trained MLI-NeRF.
    """
    pseudo_shadings = []
    pseudo_reflectances = []
    
    for pose, light_pos in zip(camera_poses, light_positions):
        # Step A: Obtain Normal and Light Visibility
        # Perform sphere tracing using SDF network
        points, normals, visibilities = sphere_trace_and_render(
            sdf_network, camera_poses=pose, light_positions=light_pos
        )
        
        # Compute light ray direction
        light_ray = normalize(light_pos - points)
        
        # Step B: Generate Pseudo Shading
        # Dot product of normal and light ray, masked by visibility
        dot_product = max(dot(normals, light_ray), 0) * visibilities
        # Apply gamma correction
        shading = dot_product ** gamma
        pseudo_shadings.append(shading)
        
        # Render RGB image for this light condition
        rgb_image = render_rgb(rgb_mlp, points)
        
        # Step C: Generate Pseudo Reflectance (Initial)
        # Avoid division by zero
        safe_shading = max(shading, 1e-6)
        initial_reflectance = rgb_image / safe_shading
        pseudo_reflectances.append(initial_reflectance)
        
    # Merge reflectances using K-means clustering with weights
    merged_reflectance = kmeans_merge(pseudo_reflectances, pseudo_shadings)
    
    # Step D: Hole Filling
    # Identify holes (e.g., where shading was near zero for all lights)
    hole_mask = is_hole(merged_reflectance, pseudo_shadings)
    filled_reflectance = fill_holes(merged_reflectance, hole_mask, normals)
    
    # Compute Weight Maps
    weight_shading = compute_shading_weight(pseudo_shadings)
    weight_reflectance = compute_reflectance_weight(filled_reflectance, weight_shading)
    
    return filled_reflectance, pseudo_shadings[0], weight_reflectance, weight_shading

def fill_holes(reflectance, hole_mask, normals):
    """
    Fills holes in reflectance using weighted distance to nearest non-hole pixel.
    """
    filled = reflectance.copy()
    hole_pixels = get_pixel_indices(hole_mask)
    non_hole_pixels = get_pixel_indices(~hole_mask)
    
    for hole in hole_pixels:
        min_dist = infinity
        best_pixel = None
        
        for non_hole in non_hole_pixels:
            # Spatial distance
            d_spatial = distance(hole, non_hole)
            # Angular difference of normals
            d_angle = angle_diff(normals[hole], normals[non_hole])
            # RGB color difference
            d_color = color_diff(reflectance[hole], reflectance[non_hole])
            
            # Weighted distance (weights can be tuned)
            dist = d_spatial + d_angle + d_color
            
            if dist < min_dist:
                min_dist = dist
                best_pixel = non_hole
        
        if best_pixel is not None:
            filled[hole] = reflectance[best_pixel]
            
    return filled
```

## Operating characteristics
- **Time Complexity:** Depends on the number of light conditions and pixels. K-means clustering adds overhead but is performed per-pixel across light conditions. Hole filling is $O(N \log N)$ or $O(N^2)$ depending on implementation, but $N$ (holes) is small.
- **Memory Complexity:** Requires storing normals, visibilities, and intermediate images for all light conditions.
- **Strengths:** Provides a physically grounded basis for intrinsic decomposition, reducing reliance on statistical priors. Handles multi-light scenarios effectively.
- **Limitations:** Relies on the accuracy of Stage 1 SDF and RGB networks. Hole filling is a heuristic and may introduce artifacts in complex scenes, though impact is minimal. Accuracy of visibility estimation affects shading quality.
- **When to Use:** When training an intrinsic-aware NeRF (Stage 2) and high-fidelity pseudo labels are needed for supervision.
- **When Not to Use:** If computational resources are extremely limited, as it requires additional post-processing steps. If the scene has significant specular or indirect lighting that cannot be ignored in the residual component, the simplified $R = I \oslash S$ may be inaccurate.

## Background
Originated in the MLI-NeRF paper (2024) [S1]. It addresses the need for reliable supervision signals for intrinsic decomposition in neural radiance fields, moving away from statistical priors used in earlier methods. The method leverages the physics of light transport (Lambert's cosine law) and sensor characteristics (gamma correction) to generate pseudo labels.

## Sources
[S1] MLI-NeRF: Multi-Light Intrinsic-Aware Neural Radiance Fields, https://arxiv.org/abs/2411.17235v1, fetched 2026-10-26
