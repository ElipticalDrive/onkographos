# Control-NeRF

Also called: Editable Feature Volumes for Scene Rendering and Manipulation

## Overview
Control-NeRF is a novel view synthesis method that decouples scene-specific geometry and appearance from the rendering model by learning dense feature volumes for each scene and a single shared neural rendering network that generalizes across scenes [S1]. Unlike standard NeRF, which memorizes radiance in a scene-specific neural network making editing difficult, Control-NeRF uses interpretable feature volumes allowing for 3D-aware manipulation like shape manipulation and combining scenes [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of input RGB images $I = \{I^s_i\}$ from $M$ training scenes $s \in \mathcal{S}$, where $N$ images are available per scene. Camera poses and intrinsics are required for ray generation.
*   **Outputs:** For a given scene $s$, a latent volumetric representation $V_s \in \mathbb{R}^{W \times H \times D \times F}$ and rendered pixel colors $\hat{C}(r, V_s, \Theta)$ for novel viewpoints.

**Data Structures:**
*   **Feature Volume ($V_s$):** A 3D grid with spatial resolution $W \times H \times D$ and feature vector length $F$. Each cell contains a feature vector $v_s \in \mathbb{R}^F$.
    *   **Layout:** Dense tensor.
    *   **Sizes:** Default experimental values are $W=H=D=128$ and $F=64$ [S1].
    *   **Memory:** $128^3 \times 64 \times 4$ bytes $\approx 512$ MB per scene (assuming float32) [S1].
*   **Rendering Network ($F_\Theta$):** A multi-layer perceptron (MLP) with parameters $\Theta$. It takes a feature vector and a viewing direction to predict density and color.

**Numbered Steps:**

1.  **Scene Representation Learning (Initial Training):**
    *   Initialize the rendering network parameters $\Theta$ and feature volumes $V_s$ for all training scenes $s \in \mathcal{S}$.
    *   Use a hierarchical training strategy (see *Multi-Resolution Volume Training*).
    *   For each iteration:
        a. Select a scene $s$ and a batch of rays $R_s$ from its training images.
        b. For each ray $r(t) = o + td$, sample points $p$ along the ray.
        c. Sample the feature vector $v_s$ from the volume $V_s$ at point $p$ using trilinear resampling: $v_s = S(V_s, p)$ [S1].
        d. Compute positional encoding $\gamma(d)$ for the viewing direction $d$.
        e. Pass $(v_s, \gamma(d))$ through the rendering network $F_\Theta$ to obtain density $\sigma$ and color $c$ [S1].
        f. Compute the coarse and fine density integrals $\hat{C}_c(r, V_s, \Theta)$ and $\hat{C}_f(r, V_s, \Theta)$ using volume rendering [S1].
        g. Compute the reconstruction loss $L_r$ against ground truth pixel colors $C(r)$ [S1].
        h. Compute the Total Variation loss $L_{tv}(V_s)$ [S1].
        i. Update $\Theta$ and $V_s$ to minimize the total loss [S1].

2.  **Multi-Resolution Volume Training:**
    *   Start with a feature volume resolution of $16^3$ [S1].
    *   Train until convergence, optimizing both $V_s$ and $\Theta$ [S1].
    *   Upsample the volume by a factor of 2 (e.g., using trilinear interpolation) [S1].
    *   Repeat for 4 stages until reaching the target resolution of $128^3$ [S1].

3.  **Generalization to Novel Scenes:**
    *   Given a new set of scenes $\mathcal{G}$ not used in initial training:
    *   Fix the rendering network parameters $\Theta$ [S1].
    *   Initialize feature volumes $V_g$ for each new scene $g \in \mathcal{G}$.
    *   Optimize only $V_g$ using the hierarchical training strategy and losses, keeping $\Theta$ fixed [S1].

4.  **Scene Editing and Manipulation:**
    *   **Spatial Deformation:** Apply trilinear resampling $S$ to contiguous subregions of the feature volume. If $V_o$ is the original volume and $P$ is a matrix of 3D coordinates indicating where to sample from, the modified volume is $V_m = S(V_o, P)$ [S1].
    *   **Scene Mixing:** Mix features from multiple scenes to simulate inserting objects from one scene into another [S1].

**Formulas:**

*   **Rendering Function:**
    $$F_\Theta : (S(V_s, p)), \gamma(d)) \rightarrow (c, \sigma)$$
    Where $c \in \mathbb{R}^3$ is the RGB radiance, $\sigma \in \mathbb{R}$ is the density, $p \in \mathbb{R}^3$ is the point, $d \in \mathbb{R}^3$ is the viewing direction, $\gamma$ is positional encoding, and $S$ is trilinear resampling [S1].

*   **Total Variation (TV) Metric for a Subregion $R$:**
    $$T(R) = \sqrt{\sum_{i,j,k} \left( |R_{i+1,j,k} - R_{i,j,k}|^2 + |R_{i,j+1,k} - R_{i,j,k}|^2 + |R_{i,j,k+1} - R_{i,j,k}|^2 \right)}$$
    Where $R$ is a subregion of the feature volume [S1].

*   **Total Variation Loss:**
    $$L_{tv}(V_s) = \mathbb{E}_{R \sim V_s} [|T(R)|]$$
    Where $R$ is a randomly sampled contiguous subregion of $V_s$ that is $1/4$ of the current latent feature volume dimensions [S1].

*   **Reconstruction Loss:**
    $$L_r(R_s, I_s, V_s, \Theta) = \mathbb{E}_{r \sim R_s} \left[ \|\hat{C}_c(r, V_s, \Theta) - C(r)\|^2 + \|\hat{C}_f(r, V_s, \Theta) - C(r)\|^2 \right]$$
    Where $\hat{C}_c$ and $\hat{C}_f$ are density integrals of coarse and fine networks, and $C(r)$ is the ground truth color [S1].

*   **Joint Optimization Loss (Initial Training):**
    $$\arg \min_{V, \Theta} L(R, I, V, \Theta) = \mathbb{E}_{s \sim \mathcal{S}} \left[ L_r(R_s, I_s, V_s, \Theta) + \lambda L_{tv}(V_s) \right]$$
    Where $\lambda = 10^{-4}$ [S1].

*   **Novel Scene Optimization Loss:**
    $$\arg \min_{V} L(R', I', V', \Theta) = \mathbb{E}_{g \sim \mathcal{G}} \left[ L_r(R'_g, I'_g, V'_g, \Theta) + \lambda L_{tv}(V'_g) \right]$$
    Where $\Theta$ is fixed [S1].

**Parameters:**
*   **Spatial Resolution:** $W=H=D=128$ [S1].
*   **Feature Vector Length:** $F=64$ [S1].
*   **TV Regularization Weight:** $\lambda = 10^{-4}$ [S1].
*   **Hierarchical Stages:** 4 stages, starting at $16^3$ and doubling resolution each stage [S1].
*   **Scene Transition Frequency:** 50 consecutive iterations between scene transitions during multi-scene training [S1].

**Edge Cases and Pitfalls:**
*   **Memory/Computation Intensity:** Training at full resolution ($128^3$ with 64-dim features) is intensive. This is mitigated by the hierarchical coarse-to-fine training strategy [S1].
*   **Generalization:** The method requires a sufficient number of training scenes (e.g., 6) to train a generalizable radiance function [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def trilinear_resample(volume, points):
    """
    Trilinear resampling of a 3D volume at given 3D points.
    volume: np.ndarray of shape (W, H, D, F)
    points: np.ndarray of shape (N, 3) with coordinates in [0, 1] or [0, W-1] etc.
            Assuming normalized coordinates [0, 1] for simplicity, scaled to volume dims.
    Returns: np.ndarray of shape (N, F)
    """
    # Implementation details of trilinear interpolation omitted for brevity,
    # but standard libraries or manual implementation using floor/ceil and weights apply.
    # This function maps continuous 3D coordinates to feature vectors.
    pass

def positional_encoding(x, min_deg, max_deg):
    """
    Standard positional encoding for NeRF.
    """
    # Scales x by powers of 2 and applies sin/cos
    pass

def render_ray(ray_origin, ray_direction, volume, network_params, num_samples_coarse, num_samples_fine):
    """
    Renders a single ray using coarse and fine sampling.
    """
    # 1. Sample points along ray
    # 2. Resample features from volume at these points
    # 3. Encode viewing direction
    # 4. Predict sigma and color via network
    # 5. Volume rendering integration (alpha blending)
    pass

def compute_tv_loss(volume, subregion_size_ratio=0.25):
    """
    Computes Total Variation loss on a random subregion of the volume.
    """
    # 1. Randomly select a contiguous subregion R of size (W/4, H/4, D/4)
    # 2. Compute T(R) as defined in formula
    # 3. Return mean T(R)
    pass

def train_control_nerf(scenes, images, poses, initial_volume_res=16, target_volume_res=128, lambda_tv=1e-4):
    """
    Main training loop for Control-NeRF.
    """
    # Initialize rendering network F_Theta
    network_params = initialize_network()
    
    # Initialize feature volumes for all scenes
    volumes = {s: np.zeros((initial_volume_res, initial_volume_res, initial_volume_res, 64)) for s in scenes}
    
    current_res = initial_volume_res
    
    # Hierarchical Training
    while current_res <= target_volume_res:
        print(f"Training at resolution {current_res}")
        
        # Train until convergence
        while not converged():
            # Sample a scene s
            s = sample_scene(scenes)
            
            # Sample rays from scene s
            rays = sample_rays(images[s], poses[s])
            
            total_loss = 0
            
            for ray in rays:
                # Render ray
                color_coarse, color_fine = render_ray(
                    ray.origin, ray.direction, 
                    volumes[s], network_params, 
                    num_samples_coarse=64, num_samples_fine=128
                )
                
                # Reconstruction Loss
                gt_color = ray.gt_color
                loss_rec = np.mean((color_coarse - gt_color)**2) + np.mean((color_fine - gt_color)**2)
                total_loss += loss_rec
            
            # Total Variation Loss
            tv_loss = compute_tv_loss(volumes[s])
            total_loss += lambda_tv * tv_loss
            
            # Backpropagate to update network_params and volumes[s]
            update_parameters(network_params, volumes[s], total_loss)
        
        # Upsample volumes for next stage
        if current_res < target_volume_res:
            new_res = current_res * 2
            for s in scenes:
                volumes[s] = upsample_volume(volumes[s], new_res)
            current_res = new_res

def generalize_to_new_scene(new_scene_images, new_scene_poses, trained_network_params, target_volume_res=128):
    """
    Generalize to a novel scene by optimizing only the feature volume.
    """
    # Initialize new volume
    new_volume = np.zeros((target_volume_res, target_volume_res, target_volume_res, 64))
    
    # Optimize new_volume while keeping trained_network_params fixed
    # Use same hierarchical strategy if needed, or direct optimization
    # Loss is same as L_r + lambda_tv * L_tv
    
    return new_volume

def edit_scene(original_volume, deformation_map):
    """
    Edit scene by resampling from original volume using deformation map.
    """
    # deformation_map: matrix of 3D coordinates P
    # V_m = S(V_o, P)
    edited_volume = trilinear_resample(original_volume, deformation_map)
    return edited_volume
```

## Operating characteristics
*   **Time Complexity:** Training is intensive due to the large feature volumes ($128^3 \times 64$). The hierarchical coarse-to-fine strategy reduces training time by starting at $16^3$ and doubling resolution [S1].
*   **Memory Complexity:** High. A single $128^3$ volume with 64-dim features requires significant GPU memory (~512 MB per scene) [S1].
*   **Measured Performance:** On the LLFF dataset test images, Control-NeRF achieves PSNR 25.635, SSIM 0.853, and LPIPS 0.181 [S1]. This compares favorably against NSVF (PSNR 20.414) and NPBG (PSNR 19.430) [S1].
*   **Strengths:**
    *   Generalizes to novel scenes by optimizing only the feature volume, keeping the rendering network fixed [S1].
    *   Enables 3D-aware editing, compositing, and manipulation of scenes post-hoc without retraining the rendering network [S1].
    *   Supports moving objects across multiple scenes, unlike object-compositional NeRF which is scene-specific [S1].
*   **Limitations:**
    *   Requires a sufficient number of training scenes (e.g., 6) to learn a generalizable radiance function [S1].
    *   Memory intensive at full resolution [S1].

## Background
Control-NeRF was introduced in the paper "Control-NeRF: Editable Feature Volumes for Scene Rendering and Manipulation" [S1]. It addresses the limitation of standard NeRF, which memorizes radiance in a scene-specific neural network, making editing and compositing difficult [S1]. By decoupling the scene representation (feature volumes) from the rendering network, it allows for interpretable and controllable scene representations [S1].

## Sources
[S1] Control-NeRF: Editable Feature Volumes for Scene Rendering and Manipulation, https://arxiv.org/abs/2204.10850, fetched 2026-10-26
