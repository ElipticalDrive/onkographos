# Scene NeRF

Also called: Background NeRF, Static Scene NeRF

## Overview
Scene NeRF is a method for isolating and rendering the static background of a scene by training a Neural Radiance Field (NeRF) exclusively on pixels identified as background via human segmentation masks [S1]. It differs from standard NeRF by explicitly excluding dynamic foreground elements (e.g., humans) from the training data and employing an emptiness regularizer to prevent artifacts in empty space [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A video sequence of the scene; human segmentation masks for each frame; camera poses and sparse scene model from COLMAP; multi-view stereo (MVS) depth maps; dense monocular depth maps.
*   **Outputs:** A trained Scene NeRF model $\mathcal{N}_s$ that renders background colors and densities.

**Data Structures:**
*   **Segmentation Mask $M(\mathbf{r})$:** A binary value for each ray $\mathbf{r}$. $M(\mathbf{r}) = 1$ if the ray intersects the human/foreground, $M(\mathbf{r}) = 0$ if it corresponds to the background [S1].
*   **Fused Depth Map $D_{fuse}(\mathbf{r})$:** A depth estimate for each ray, obtained by fusing MVS and monocular depth maps [S1].

**Preprocessing:**
1.  Use COLMAP to obtain camera poses, sparse scene model, and MVS depth maps $D_{mvs}$ [S1].
2.  Retrieve human segmentation maps using Mask-RCNN [S1].
3.  Fill holes in $D_{mvs}$ using dense monocular depth maps $D_{mono}$ (e.g., Miangoleh et al. [S1]).
4.  Fuse $D_{mvs}$ and $D_{mono}$ into $D_{fuse}$ by finding a linear mapping between the two depth maps using pixels with both estimates, then transforming $D_{mono}$ values to match the $D_{mvs}$ scale [S1].

**Training Steps:**
For each ray $\mathbf{r}$ sampled from the camera:

1.  **Determine Mask:** Check if the ray corresponds to the human or background using the segmentation mask $M(\mathbf{r})$ [S1].
2.  **Sample Terminating Depth:** Determine the terminating depth $\hat{z}_\mathbf{r}$ for the emptiness regularizer by querying the fused depth map: $\hat{z}_\mathbf{r} = D_{fuse}(\mathbf{r})$ [S1].
3.  **Compute RGB Loss:** Calculate the reconstruction loss only for background pixels. If $M(\mathbf{r}) = 1$, the loss is zero. If $M(\mathbf{r}) = 0$, the loss is the squared difference between the rendered color $\mathbf{C}_s(\mathbf{r})$ and the ground-truth color $\hat{\mathbf{C}}(\mathbf{r})$ [S1]:
    $$
    \mathcal{L}_{s,rgb}(\mathbf{r}) = (1 - M(\mathbf{r})) \left\| \mathbf{C}_s(\mathbf{r}) - \hat{\mathbf{C}}(\mathbf{r}) \right\|^2
    $$
    *Note: The source formula uses $\left\| \dots \right\|$ without specifying the norm power, but standard reconstruction loss implies $L_2$ squared. The text says "RGB reconstruction loss... defined as...". We assume $L_2$ squared as is standard, though the source notation $\left\| \dots \right\|$ is ambiguous. Given the context of "minimizing", $L_2$ squared is the standard interpretation for such losses in NeRF literature [S1].*
4.  **Compute Emptiness Regularizer:** Calculate the integral of the scene density $\sigma_s$ from the near plane $t_n$ to a slack margin $\alpha \hat{z}_\mathbf{r}$. This forces density to be zero in the empty space between the camera and the estimated scene surface [S1]:
    $$
    \mathcal{L}_{s,empty}(\mathbf{r}) = \int_{t_n}^{\alpha \hat{z}_\mathbf{r}} \sigma_s(\mathbf{r}(t)) dt
    $$
    where $\alpha = 0.8$ is a slack margin to avoid strong regularization when depth estimates are inaccurate [S1].
5.  **Compute Total Loss:** Sum the RGB loss and the emptiness regularizer weighted by $\lambda_{empty}$ [S1]:
    $$
    \mathcal{L}_{s} = \mathcal{L}_{s,rgb}(\mathbf{r}) + \lambda_{empty} \mathcal{L}_{s,empty}(\mathbf{r})
    $$
    where $\lambda_{empty} = 0.1$ [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def train_scene_nerf(scene_nerf, rays, gt_colors, masks, fused_depths, lambda_empty=0.1, alpha=0.8, t_n=0.0):
    """
    scene_nerf: The NeRF model function C_s(r) returning (color, density)
    rays: List of ray origins and directions
    gt_colors: Ground truth RGB colors for each ray
    masks: Segmentation masks M(r), 1 for human, 0 for background
    fused_depths: Fused depth estimates D_fuse(r)
    lambda_empty: Weight for emptiness regularizer
    alpha: Slack margin for depth
    t_n: Near plane distance
    """
    total_loss = 0.0
    
    for r, c_gt, m, z_fuse in zip(rays, gt_colors, masks, fused_depths):
        # 1. Render from Scene NeRF
        color_s, density_s = scene_nerf(r)
        
        # 2. RGB Loss (only for background)
        # If m=1 (human), loss is 0. If m=0 (background), loss is L2 squared error.
        rgb_loss = (1 - m) * torch.sum((color_s - c_gt) ** 2)
        
        # 3. Emptiness Regularizer
        # Calculate integral of density from t_n to alpha * z_fuse
        # This requires sampling along the ray and summing density * weight
        # Simplified approximation for the integral:
        # We sample points along the ray up to alpha * z_fuse and sum density
        # Note: The source defines it as an integral of sigma_s. 
        # In practice, this is approximated by the sum of densities at sample points 
        # weighted by the step size or transmittance. 
        # However, the source formula is explicit: integral of sigma_s.
        # A common implementation in NeRF regularizers is summing density * delta_t.
        
        # Determine max depth for regularizer
        max_depth = alpha * z_fuse
        
        # Sample points along ray from t_n to max_depth
        # This step assumes a standard volume rendering sampler is available or implemented
        # For the purpose of this article, we assume the integral is computed via sampling
        # as per standard NeRF practices [G].
        
        # Approximation of integral: sum of density * step_size
        # In a full implementation, one would use the same samples as the color render
        # but restrict the upper bound to max_depth.
        
        # Placeholder for the actual integration logic which depends on the specific
        # volume rendering implementation details not fully specified in the source
        # beyond the formula. We note that this regularizer penalizes non-zero density
        # in the empty space.
        
        # Let's assume a standard discrete approximation:
        # density_integral = sum(sigma_s(r(t_i)) * (t_{i+1} - t_i)) for t_i in [t_n, max_depth]
        
        # Since the source does not provide the discretization, we leave this as a comment.
        # The key is that this term forces sigma_s to be 0 in empty space.
        
        empty_loss = compute_density_integral(scene_nerf, r, t_n, max_depth)
        
        # 4. Total Loss
        loss = rgb_loss + lambda_empty * empty_loss
        total_loss += loss
        
    return total_loss

def compute_density_integral(nerf_model, ray, t_near, t_far):
    """
    Computes the integral of density along the ray from t_near to t_far.
    """
    # Sample points along the ray
    # Standard NeRF sampling strategy would be used here [G]
    # For simplicity, assume uniform sampling or stratified sampling
    num_samples = 128
    t_vals = torch.linspace(t_near, t_far, num_samples)
    
    # Get positions
    positions = ray.origin + ray.direction * t_vals
    
    # Get densities
    densities = nerf_model.get_density(positions) # Assuming a method to get density only
    
    # Approximate integral using trapezoidal rule or simple sum
    # delta_t is the distance between samples
    delta_t = (t_far - t_near) / num_samples
    
    integral = torch.sum(densities) * delta_t
    return integral
```

**Parameters:**
*   $\lambda_{empty}$: Hyperparameter controlling the weight of the emptiness regularizer. Set to 0.1 in all experiments [S1].
*   $\alpha$: Slack margin for the emptiness regularizer. Set to 0.8 [S1].

**Edge Cases and Pitfalls:**
*   **Hazy Objects:** Simply minimizing the RGB reconstruction loss leads to 'hazy' objects floating in the scene. This is resolved by adding the emptiness regularizer [S1].
*   **Inaccurate Depth:** The slack margin $\alpha = 0.8$ is used to avoid strong regularization when depth estimates are inaccurate [S1].
*   **Single Human Assumption:** The method assumes a single human being in the scene, though it can be trivially extended [S1].

## Operating characteristics
*   **Time and Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Not given in the sources.
*   **Strengths and Limitations:**
    *   **Strengths:** Effectively isolates static background from dynamic foreground (humans) [S1].
    *   **Limitations:** Requires accurate human segmentation masks and depth estimates. Performance degrades if depth estimates are inaccurate without the slack margin [S1].
*   **When to Use:** When the goal is to reconstruct the static background of a scene with dynamic foreground elements, given video input and segmentation masks [S1].
*   **When Not to Use:** When the scene contains multiple dynamic objects that are not segmented, or when segmentation masks are unavailable [S1].

## Background
*   **Origin:** The method is described in "NeuMan: Neural Human Radiance Field from a Single Video" by W. Jiang et al. [S1].
*   **History and Development:** The approach builds on traditional motion detection work by using a NeRF for the background model [S1]. It follows the approach of Video-NeRF [S1] in using an emptiness regularizer.
*   **Where it is Used:** Used for reconstructing static backgrounds in videos containing human subjects [S1].

## Sources
[S1] NeuMan: Neural Human Radiance Field from a Single Video, https://doi.org/10.1007/978-3-031-19824-3_24, fetched 2026-10-29
