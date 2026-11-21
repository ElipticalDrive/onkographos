# RS-Recon

Also called: Ray-Distance Volume Rendering for Neural Scene Reconstruction

## Overview
RS-Recon is a neural scene reconstruction method that replaces the standard Signed Distance Function (SDF) with a Signed Ray Distance Function (SRDF) to model volume density, specifically addressing noise and inaccuracies in multi-object indoor scenes [S1]. Unlike SDF-based methods which consider surface points across the entire scene and may exhibit fluctuations leading to local minimum values, SRDF measures the shortest distance to the surface along a specific camera ray, eliminating the influence of surrounding surfaces not located on that ray [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Multi-view RGB images of an indoor scene [S1].
*   **Output:** Reconstructed 3D geometry and synthesized novel views [S1].

**Data Structures:**
The method utilizes three distinct Multi-Layer Perceptrons (MLPs) and associated feature tensors [S1]:
1.  **Geometry MLP ($f_g$):**
    *   **Input:** Encoded position $\mathbf{x}$ (and optionally grid features) [S1].
    *   **Output:** SDF value $s_{SDF}(\mathbf{x})$ and geometry features $\mathbf{F}_g$ [S1].
2.  **SRDF MLP ($f_s$):**
    *   **Input:** Encoded position $\mathbf{x}$ and camera ray direction $\mathbf{d}$ [S1].
    *   **Output:** Signed Ray Distance value $s_{SRDF}(\mathbf{x}, \mathbf{d})$ and visibility probability $v(\mathbf{x}, \mathbf{d})$ [S1].
3.  **Color MLP ($f_c$):**
    *   **Input:** Geometry features $\mathbf{F}_g$ and viewing direction $\mathbf{d}$ [S1].
    *   **Output:** Color $\mathbf{c}(\mathbf{x}, \mathbf{d})$ [S1].

**Numbered Steps:**
1.  **Ray Sampling:** For each pixel, cast a ray through the scene and sample points $\mathbf{x}_i$ along the ray [G].
2.  **SRDF Prediction:** Pass each point $\mathbf{x}_i$ and the ray direction $\mathbf{d}$ into the SRDF MLP ($f_s$) to predict the signed ray distance $s_{SRDF}$ and visibility probability $v$ [S1].
3.  **SDF Prediction:** Pass each point $\mathbf{x}_i$ into the Geometry MLP ($f_g$) to predict the signed distance $s_{SDF}$ and geometry features $\mathbf{F}_g$ [S1].
4.  **Density Conversion:** Convert the SRDF value $s_{SRDF}$ into a ray-conditioned volume density $\sigma$ using a differentiable mapping that concentrates density near the zero-level set [G].
5.  **Color Prediction:** Use the geometry features $\mathbf{F}_g$ and viewing direction $\mathbf{d}$ in the Color MLP ($f_c$) to predict the color $\mathbf{c}$ for each point [S1].
6.  **Volume Rendering:** Render the final image by integrating the predicted colors and densities along the ray using standard differentiable volumetric rendering [G].
7.  **Loss Computation:** Compute the total loss using photometric consistency, SRDF-SDF consistency, and self-supervised visibility tasks [S1].

**Formulas:**
*   **SRDF-SDF Consistency Loss ($\mathcal{L}_{con}$):** Constrains the signs of the SRDF and SDF outputs to ensure they share the same sign indicating spatial occupancy [S1]. The exact mathematical formulation is not provided in the sources [G]. It enforces $\text{sign}(s_{SRDF}) = \text{sign}(s_{SDF})$ [S1].
*   **Visibility Task:** A self-supervised visibility task predicts the probability of whether sampled points are physically visible along the ray [S1]. It combines priors from predicted SRDF and SDF as pseudo labels [S1].

**Parameters:**
*   **SRDF MLP ($f_s$):** Dedicated MLP for SRDF values [S1].
*   **Geometry MLP ($f_g$):** MLP for SDF and geometry features [S1].
*   **Color MLP ($f_c$):** MLP for color prediction [S1].
*   Typical values for hyperparameters (e.g., learning rate, MLP depth) are not given in the sources [S1].

**Edge Cases and Pitfalls:**
*   **Multi-object Scenes:** Standard SDF may introduce noise due to fluctuations from surrounding surfaces [S1]. SRDF mitigates this by being ray-specific [S1].
*   **Visibility Ground Truth:** Unlike methods like VIP-NeRF that depend on multi-view geometry computation for visibility labels, RS-Recon's self-supervised visibility task requires minimal computational increase and does not depend on multi-view geometry for ground truth generation [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def convert_srdf_to_density(srdf_values):
    # [G] Standard differentiable mapping concentrates density near s_SRDF=0.
    # Implementation varies; here is a common sigmoid-based approximation.
    return 1.0 / (1.0 + np.exp(-srdf_values))

def mse_loss(pred, target):
    return np.mean((pred - target) ** 2)

def sign_consistency_loss(sdf, srdf):
    # [G] Not given in sources. Enforces sign agreement.
    return np.mean(np.abs(np.sign(sdf) - np.sign(srdf)))

def visibility_loss(vis_probs):
    # [G] Not given in sources. Typically binary cross-entropy against pseudo labels.
    # Pseudo labels are derived from SRDF/SDF priors [S1].
    return np.mean(-vis_probs * np.log(vis_probs + 1e-8) - (1 - vis_probs) * np.log(1 - vis_probs + 1e-8))

def rs_recon_forward(positions, ray_directions, geometry_mlp, srdf_mlp, color_mlp):
    """
    Forward pass for RS-Recon.
    
    Args:
        positions: Tensor of shape (N, 3) containing 3D points along rays.
        ray_directions: Tensor of shape (N, 3) containing ray directions for each point.
        geometry_mlp: MLP predicting SDF and geometry features.
        srdf_mlp: MLP predicting SRDF and visibility.
        color_mlp: MLP predicting color.
        
    Returns:
        colors: Tensor of predicted colors.
        densities: Tensor of predicted densities.
        visibility_probs: Tensor of visibility probabilities.
        sdf_values: Tensor of SDF values.
        srdf_values: Tensor of SRDF values.
    """
    # Step 2: SRDF Prediction
    srdf_values, visibility_probs = srdf_mlp(positions, ray_directions)
    
    # Step 3: SDF Prediction
    sdf_values, geometry_features = geometry_mlp(positions)
    
    # Step 4: Density Conversion
    densities = convert_srdf_to_density(srdf_values)
    
    # Step 5: Color Prediction
    colors = color_mlp(geometry_features, ray_directions)
    
    return colors, densities, visibility_probs, sdf_values, srdf_values

def rs_recon_loss(colors, visibility_probs, sdf_values, srdf_values, gt_images):
    """
    Compute the total loss for RS-Recon.
    
    Args:
        colors: Predicted colors.
        visibility_probs: Predicted visibility probabilities.
        sdf_values: Predicted SDF values.
        srdf_values: Predicted SRDF values.
        gt_images: Ground truth images.
        
    Returns:
        total_loss: Scalar loss value.
    """
    # Photometric consistency loss (standard NeRF loss)
    photometric_loss = mse_loss(colors, gt_images)
    
    # SRDF-SDF Consistency Loss
    consistency_loss = sign_consistency_loss(sdf_values, srdf_values)
    
    # Self-supervised visibility loss
    visibility_loss_val = visibility_loss(visibility_probs)
    
    total_loss = photometric_loss + consistency_loss + visibility_loss_val
    return total_loss
```

## Operating characteristics
*   **Time and Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Validated on indoor datasets, achieving improved performance in both reconstruction and view synthesis compared to SDF-based methods [S1].
*   **Scaling:** Not given in the sources.
*   **Strengths:** Eliminates influence of surrounding surfaces not on the camera ray; aligns density distribution with local maxima near intersected surfaces; self-supervised visibility task requires minimal computational increase [S1].
*   **Limitations:** Specific conversion from SRDF to density is not detailed; relies on consistency between SRDF and SDF which may require careful tuning [G].
*   **When to use:** Indoor scene reconstruction with multi-object scenes where standard SDF may introduce noise [S1].
*   **When not:** Scenes where ray-specific distance is not the dominant geometric prior [G].

## Background
RS-Recon was proposed in the paper "Ray-Distance Volume Rendering for Neural Scene Reconstruction" [S1]. It addresses the issue of noise and inaccuracies in multi-object indoor scene reconstruction by modeling the volume density function using the Signed Ray Distance Function (SRDF) instead of the Signed Distance Function (SDF) [S1]. The method was validated on indoor datasets, demonstrating improved performance in reconstruction and view synthesis [S1].

## Sources
[S1] Ray-Distance Volume Rendering for Neural Scene Reconstruction, https://doi.org/10.1007/978-3-031-72630-9_22, fetched 2026-10-26
