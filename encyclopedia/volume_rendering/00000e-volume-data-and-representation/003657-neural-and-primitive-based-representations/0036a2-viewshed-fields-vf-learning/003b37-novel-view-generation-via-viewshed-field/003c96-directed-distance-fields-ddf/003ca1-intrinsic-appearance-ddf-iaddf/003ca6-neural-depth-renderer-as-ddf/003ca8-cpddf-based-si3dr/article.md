# CPDDF-based SI3DR

Also called: Conditional PDDF Reconstruction, SI3DR with DDFs, Single-Image 3D Reconstruction (SI3DR)

## Overview
CPDDF-based SI3DR is a method for single-image 3D reconstruction that predicts the latent shape and camera parameters from a single input image, then uses a Conditional Probabilistic Directed Distance Field (CPDDF) to render geometry from arbitrary viewpoints [S1]. It differs from direct mesh or point cloud prediction methods by implicitly representing the shape as a differentiable field conditioned on a latent code, allowing for view-consistent rendering and explicit sampling [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A single color image $I$ and optionally an alpha channel $I_\alpha$ [S1].
*   **Output:** A predicted latent shape code $z_s$, a predicted camera parameter set $\Pi_b$, and the ability to render depth $d_b(p, v|z_s)$ and visibility $\xi_b(p, v|z_s)$ for any camera position $p$ and ray direction $v$ [S1].

**Data Structures:**
*   **Encoder Output:** A tuple $(z_{bs}, \Pi_b)$ where $z_{bs}$ is the latent shape vector and $\Pi_b$ represents camera parameters (intrinsics and extrinsics) [S1].
*   **CPDDF:** A neural network (modulated SIREN) that takes the latent code $z_s$ and camera parameters $(p, v)$ as input, outputting depth and visibility [S1].
*   **Sampling Points:** For evaluation, points are sampled along camera rays. The number of points varies by dataset configuration: 5000 points for general evaluation and 2466 points for comparison with Pixel2Mesh (P2M) [S1].

**Algorithm Steps:**
1.  **Encode:** Pass the input image $I$ through an encoder $E$ (implemented as two ResNet-18 networks) to obtain the latent shape $z_{bs}$ and predicted camera $\Pi_b$ [S1]:
    $$ E(I) = (z_{bs}, \Pi_b) $$
2.  **Decode (Render):** Use the CPDDF conditioned on $z_{bs}$ to compute depth $d_b(p, v|z_{bs})$ and visibility $\xi_b(p, v|z_{bs})$ for the predicted camera $\Pi_b$ [S1].
3.  **Compute Losses:** Calculate three loss terms:
    *   **Shape Loss ($L_S$):** DDF fitting loss in the canonical pose (as defined in Eq. 2 of the source) [S1].
    *   **Camera Loss ($L_\Pi$):** Mean squared error between the ground truth camera $\Pi_g$ and the predicted camera $\Pi_b$ [S1]:
        $$ L_\Pi = ||\Pi_g - \Pi_b||^2_2 $$
    *   **Mask Matching Loss ($L_M$):** Binary Cross Entropy (BCE) between the input alpha channel $I_\alpha$ and the rendered visibility mask $R_\xi(z_s|\Pi_b)$ [S1]:
        $$ L_M = \text{BCE}(I_\alpha, R_\xi(z_s|\Pi_b)) $$
4.  **Optimize:** Minimize the full objective function:
    $$ L_{SI3DR} = \gamma_{R,S} L_S + \gamma_{R,\Pi} L_\Pi + \gamma_{R,M} L_M $$
    where $\gamma_{R,S}, \gamma_{R,\Pi}, \gamma_{R,M}$ are weighting hyperparameters [S1].
5.  **Camera Correction (Optional):** To improve results, correct the predicted camera $\Pi_b$ to $\Pi_b^\nabla$ via gradient descent on the mask matching loss $L_M$, starting from $\Pi_b$ [S1]:
    $$ \Pi_b^\nabla = \arg\min_\Pi L_M $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def train_si3dr(image_I, alpha_I, ground_truth_camera_Pi_g, ground_truth_shape_S):
    # 1. Encode image to latent shape and predicted camera
    # Encoder E consists of two ResNet-18 networks
    latent_shape_z_bs, predicted_camera_Pi_b = Encoder_E(image_I)
    
    # 2. Decode using CPDDF (Modulated SIREN)
    # CPDDF takes latent shape and camera params (p, v)
    # Returns depth d_b and visibility xi_b
    # Note: p and v are derived from Pi_b and camera intrinsics
    depth_d_b, visibility_xi_b = CPDDF_Decoder(latent_shape_z_bs, predicted_camera_Pi_b)
    
    # 3. Compute Shape Loss (DDF fitting in canonical pose)
    # Uses ground truth shape S and canonical pose
    loss_shape = compute_ddf_fitting_loss(depth_d_b, visibility_xi_b, ground_truth_shape_S)
    
    # 4. Compute Camera Loss
    loss_camera = mean_squared_error(ground_truth_camera_Pi_g, predicted_camera_Pi_b)
    
    # 5. Compute Mask Matching Loss
    # Render visibility mask from predicted camera
    rendered_mask = render_visibility_mask(visibility_xi_b)
    loss_mask = binary_cross_entropy(alpha_I, rendered_mask)
    
    # 6. Combine Losses
    # Gamma values are hyperparameters
    total_loss = (gamma_R_S * loss_shape + 
                  gamma_R_Pi * loss_camera + 
                  gamma_R_M * loss_mask)
    
    return total_loss

def extract_point_cloud(latent_shape_z, camera_Pi):
    """
    Extract explicit point cloud from CPDDF.
    For each pixel, trace ray v from camera center p.
    If visibility xi(p, v) == 1, point q = p + d(p, v)*v is on surface.
    """
    points = []
    for each pixel in image:
        p = camera_center_of_projection(camera_Pi)
        v = ray_direction(camera_Pi, pixel_coordinates)
        d = CPDDF_Decoder(latent_shape_z, p, v).depth
        xi = CPDDF_Decoder(latent_shape_z, p, v).visibility
        
        if xi == 1:
            q = p + d * v
            points.append(q)
    return points
```

**Parameters:**
*   **Encoder:** Two ResNet-18 networks [S1].
*   **Decoder:** Modulated SIREN [S1].
*   **Sampling:** 5000 points for general evaluation; 2466 points for P2M comparison [S1].
*   **Hidden Dimensionality:** 512 for layers [S1].

**Edge Cases and Pitfalls:**
*   **Pose Estimation Errors:** Incorrect camera prediction leads to blurry shape parts, particularly for rare shapes where the model is uncertain [S1].
*   **Surface Intersections:** Near surface intersections, the directed distance value $v^*$ is ill-defined, causing difficulties in reconstruction [S1].
*   **Discontinuities:** Discontinuities in $v^*$ occur at surfaces and on the medial surface of the shape $S$, where multiple valid values of $v^*$ exist [S1].

## Operating characteristics
*   **Performance:** DDF-based rendering is significantly faster than sphere tracing (DIST). DDF takes <0.01 seconds per render compared to 5.3 seconds for DIST, a ~100× speedup [S1].
*   **Accuracy:** DDF-derived point clouds perform similarly to directly learning point clouds but underperform Pixel2Mesh (P2M) overall. However, they outperform P2M in shape quality when camera prediction error is excluded [S1].
*   **Scalability:** DDF runtime is unaffected by camera distance, unlike sphere tracers which slow down when the camera moves closer to the shape [S1].
*   **Strengths:** High rendering efficiency, differentiable rendering for gradient-based optimization, and ability to generate 3D-aware representations from single images [S1].
*   **Limitations:** Susceptibility to blurry outputs for rare shapes due to pose uncertainty; ill-defined distances at medial surfaces and intersections [S1].

## Background
CPDDF-based SI3DR is described in the paper "Probabilistic Directed Distance Fields for Ray-Based Shape Representations" [S1]. It builds on the concept of treating a neural depth renderer as a conditional DDF [Parent]. The method was developed to leverage the efficiency and differentiability of DDFs for single-image 3D reconstruction tasks.

## Sources
[S1] Probabilistic Directed Distance Fields for Ray-Based Shape Representations, https://doi.org/10.1109/tpami.2025.3594225, fetched 2026-10-29
