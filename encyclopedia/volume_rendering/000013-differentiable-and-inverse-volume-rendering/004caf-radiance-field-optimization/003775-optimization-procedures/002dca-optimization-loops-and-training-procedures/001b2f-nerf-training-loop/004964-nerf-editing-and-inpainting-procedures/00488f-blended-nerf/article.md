# Blended-NeRF

Also called: Zero-Shot Object Generation and Blending in Existing Neural Radiance Fields

## Overview
Blended-NeRF is a framework for editing specific regions of interest (ROI) in an existing Neural Radiance Field (NeRF) scene based on text prompts and a user-defined 3D ROI box [S1]. It solves the problem of local 3D editing by training a generator NeRF initialized with the original scene's weights and blending the synthesized content with the original scene using volumetric blending [S1]. Unlike methods that synthesize new objects from scratch (e.g., Dream Fields) or require semantic decomposition/2D masks for localization (e.g., Volumetric Disentanglement), Blended-NeRF operates on any region using a 3D box without learning a new feature space or requiring 2D masks [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** An existing NeRF scene $F_{\theta_O}$ representing scene $x_o$, a 3D Region of Interest (ROI) box $B$, and a user-provided text prompt $y$ [S1].
*   **Output:** An edited scene $x_e$ where the region inside $B$ is modified according to $y$, and the region outside $B$ is preserved ($x_e \odot (1 - B) = x_o \odot (1 - B)$) [S1].

**Data Structures:**
*   **NeRF Model ($F_{\theta}$):** A Multi-Layer Perceptron (MLP) mapping 3D coordinates and viewing directions to density $\sigma$ and color $\mathbf{c}$.
*   **Positional Encoding:** High-frequency sinusoidal encoding $\gamma(\mathbf{x}) = [\cos(2^l \mathbf{x}), \sin(2^l \mathbf{x})]_{l=0}^{L-1}$ applied to inputs [S1].
*   **ROI Box ($B$):** Defined by a center and dimensions, localized via a GUI using depth maps from $F_{\theta_O}$ [S1].
*   **Language-Image Model:** Pretrained model (e.g., CLIP) with image encoder $E_{img}$ and text encoder $E_{txt}$ [S1].

**Parameters:**
*   $\alpha$: Hyperparameter for distance smoothing blending strength. $\alpha=0$ weights all to edited scene; higher values weight original scene points closer to ROI center. Typical values: 0, 0.5, 2, 4, 10 [S1].
*   $\lambda_T, \lambda_D$: Weights for transmittance and depth losses [S1].
*   $\tau$: Max transmittance for $L_T$ [S1].
*   $\rho$: Max variance for depth loss $L_D$ [S1].
*   $\epsilon$: Small constant for numerical stability [S1].
*   $\gamma$: Camera Field of View (FOV) [S1].
*   $e_{max}$: Maximum dimension of the ROI box [S1].

**Numbered Steps:**

1.  **Initialization:** Initialize a generator NeRF model $F_{\theta_G}$ with the weights of the original scene model $F_{\theta_O}$ [S1].
2.  **ROI Localization:** Localize a 3D box $B$ inside the scene using a GUI that renders $F_{\theta_O}$ and uses its depth map for 3D understanding [S1].
3.  **Pose Sampling:** For each training iteration, sample a random camera pose around the scene with azimuth/elevation angles $(\theta, \phi)$. Calculate initial distance $d = \frac{e_{max}}{2 \tan(\gamma/2)}$ and randomly sample radius $r$ around $d$ [S1].
4.  **Ray Casting and Sampling:** Cast rays $r$ through the sampled camera pose. Sample points $\mathbf{x}_i$ along the ray.
5.  **Training Mode Selection:**
    *   *Object Insertion/Replacement:* Consider only the radiance field of $F_{\theta_G}$ inside $B$. For points outside $B$, set density to zero.
    *   *Object Blending:* Query both $F_{\theta_O}$ and $F_{\theta_G}$ for points inside $B$.
6.  **Rendering (Training):**
    *   If using *Insertion/Replacement*, render generated content inside ROI $I_{ROI}$ using only points $\mathbf{x}_i \in B$. Points outside $B$ have zero density. The rendered color $C(r)$ is:
        $$
        C(r) = \begin{cases} 
        \frac{\sum_{\mathbf{x}_i \in B} T_i (1 - e^{-\sigma_i \delta_i}) c_i}{\sum_{\mathbf{x}_i \in B} T_i (1 - e^{-\sigma_i \delta_i})} & \text{if } \exists \mathbf{x}_i \in r \text{ s.t. } \mathbf{x}_i \in B \\
        0 & \text{otherwise}
        \end{cases}
        $$
        where $T_i = \exp(-\sum_{j < i} \sigma_j \delta_j)$ [S1].
    *   If using *Blending*, compute alpha values for each point $\mathbf{x}_i$ from both models:
        $$
        \alpha_O(\mathbf{x}_i) = 1 - \exp(\phi(\sigma_O(\mathbf{x}_i)) \cdot \delta_i)
        $$
        $$
        \alpha_G(\mathbf{x}_i) = 1 - \exp(\phi(\sigma_G(\mathbf{x}_i)) \cdot \delta_i)
        $$
        where $\phi$ is a non-negative activation function [S1].
        Compute blended color:
        $$
        c(\mathbf{x}_i) = S\left(\frac{c_O(\mathbf{x}_i) \cdot \alpha_O(\mathbf{x}_i) + c_G(\mathbf{x}_i) \cdot \alpha_G(\mathbf{x}_i)}{\epsilon + \alpha_O(\mathbf{x}_i) + \alpha_G(\mathbf{x}_i)}\right)
        $$
        where $S$ is the sigmoid function [S1].
        Compute blended density $\sigma(\mathbf{x}_i)$ using one of two options:
        *   Option 1 (Sum in activation): $\sigma(\mathbf{x}_i) = \phi(\sigma_O(\mathbf{x}_i) + \sigma_G(\mathbf{x}_i))$ [S1].
        *   Option 2 (Sum out activation): $\sigma(\mathbf{x}_i) = \phi(\sigma_O(\mathbf{x}_i)) + \phi(\sigma_G(\mathbf{x}_i))$ [S1].
        Use these blended values to compute transmittance and render $I_{ROI}$ [S1].
7.  **Loss Calculation:**
    *   **Similarity Loss:** $L_{sim} = -E_{img}(I_{ROI})^T E_{txt}(y)$ [S1].
    *   **Transmittance Loss:** $L_T = -\min(\tau, \text{mean}(T(P)))$, where $T(P)$ is the transmittance of the rendered view from pose $P$ [S1].
    *   **Depth Loss:** $L_D = -\min(\rho, \sigma^2(D(P)))$, where $\sigma^2(D(P))$ is the variance of the disparity map of the rendered view from pose $P$ [S1].
    *   **Total Loss:** $L_{total} = L_{sim} + \lambda_T L_T + \lambda_D L_D$ [S1].
    *   *Note:* $L_T$ and $L_D$ are gradually introduced during training using an annealing strategy [S1].
8.  **Optimization:** Optimize $F_{\theta_G}$ by minimizing $L_{total}$ [S1].
9.  **Blending (Inference):** After training, render the edited scene by blending sample points from $F_{\theta_O}$ and $F_{\theta_G}$ along each view ray.
    *   Points inside $B$ are blended using the distance smoothing operator. Let $d(\mathbf{x})$ be the Euclidean distance from $\mathbf{x}$ to the ROI center, and $diag$ be the box diagonal. Compute blending weight:
        $$
        f(\mathbf{x}) = 1 - \exp\left(-\alpha \frac{d(\mathbf{x})}{diag}\right)
        $$
        Compute blended density and color:
        $$
        \sigma_{blend}(\mathbf{x}) = f(\mathbf{x}) \cdot \sigma_O(\mathbf{x}) + (1 - f(\mathbf{x})) \cdot \sigma_G(\mathbf{x})
        $$
        $$
        c_{blend}(\mathbf{x}) = f(\mathbf{x}) \cdot c_O(\mathbf{x}) + (1 - f(\mathbf{x})) \cdot c_G(\mathbf{x})
        $$
        [S1].
    *   Points outside $B$ use $F_{\theta_O}$ directly.
    *   The final rendered view $x_e$ is obtained by blending the raw densities and RGB values inside and outside the ROI along each ray using the standard volume rendering equation [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def positional_encoding(x, L=10):
    """High frequency sinusoidal positional encoding."""
    # x is shape (..., 3)
    scales = np.array([2**l for l in range(L)])
    pe = []
    for scale in scales:
        pe.append(np.cos(scale * x))
        pe.append(np.sin(scale * x))
    return np.concatenate(pe, axis=-1)

def volume_rendering(samples, densities, colors, transmittance_threshold=1e-4):
    """
    Standard volume rendering.
    samples: (N, 3) 3D points along ray
    densities: (N,) sigma values
    colors: (N, 3) RGB values
    """
    # Compute delta distances
    # Assuming samples are sorted along ray
    deltas = np.diff(np.concatenate([samples[:-1], samples[-1:]]), axis=0)
    # Handle first point distance if needed, usually delta_i = t_{i+1} - t_i
    # For simplicity, assume deltas are provided or computed from t coordinates
    # Here we assume 'samples' are t coordinates for delta calculation, 
    # but standard NeRF uses 3D points. 
    # Re-reading source: delta_i = t_{i+1} - t_i. 
    # We need t coordinates. Let's assume input 'samples' are actually t coordinates 
    # or we compute distance between 3D points. 
    # Source eq 1: delta_i = t_{i+1} - t_i. 
    # Let's assume we have t_coords.
    pass 

def blended_nerf_training(F_theta_O, B, text_prompt, num_iterations, lambda_T, lambda_D, alpha_smooth):
    """
    F_theta_O: Original NeRF model
    B: ROI Box
    text_prompt: String
    """
    # 1. Initialize Generator
    F_theta_G = copy_weights(F_theta_O)
    
    # 2. Get CLIP encoders
    E_img, E_txt = load_clip()
    
    for i in range(num_iterations):
        # 3. Pose Sampling
        theta, phi = sample_random_angles()
        e_max = get_max_box_dim(B)
        gamma = camera_fov
        d_init = e_max / (2 * np.tan(gamma / 2))
        r = np.random.uniform(d_init * 0.8, d_init * 1.2)
        pose = get_camera_pose(theta, phi, r)
        
        # 4. Cast Rays
        rays = cast_rays_from_pose(pose, image_resolution)
        
        # 5. Sample Points
        # Sample points along rays. 
        # For training, we focus on ROI.
        # If Insertion/Replacement mode:
        #   Only render inside B. Outside B density = 0.
        # If Blending mode:
        #   Query both F_theta_O and F_theta_G inside B.
        
        rendered_color = render_training_view(F_theta_O, F_theta_G, rays, B, mode='blending')
        
        # 6. Compute Losses
        # Similarity Loss
        I_ROI = extract_roi(rendered_color, B)
        img_feat = E_img(I_ROI)
        txt_feat = E_txt(text_prompt)
        L_sim = -np.dot(img_feat, txt_feat)
        
        # Transmittance Loss
        T_P = compute_transmittance(rendered_color)
        tau = 0.9 # Example value
        L_T = -np.minimum(tau, np.mean(T_P))
        
        # Depth Loss
        D_P = compute_disparity(rendered_color)
        rho = 1.0 # Example value
        sigma_sq = np.var(D_P)
        L_D = -np.minimum(rho, sigma_sq)
        
        # Annealing for L_T and L_D
        weight_T = anneal(i, start=0, end=1000)
        weight_D = anneal(i, start=0, end=1000)
        
        L_total = L_sim + lambda_T * weight_T * L_T + lambda_D * weight_D * L_D
        
        # 7. Optimize
        gradients = compute_gradients(L_total, F_theta_G)
        update_weights(F_theta_G, gradients)
        
    return F_theta_G

def blended_nerf_rendering(F_theta_O, F_theta_G, B, alpha_smooth):
    """
    Render final edited scene.
    """
    # For each view ray
    for ray in rays:
        samples = sample_along_ray(ray)
        
        # Split samples into inside B and outside B
        samples_outside = [s for s in samples if s not in B]
        samples_inside = [s for s in samples if s in B]
        
        # Render outside using Original
        sigma_out = F_theta_O.density(samples_outside)
        c_out = F_theta_O.color(samples_outside)
        color_out, trans_out = volume_render(samples_outside, sigma_out, c_out)
        
        # Render inside using Blended
        sigma_O_in = F_theta_O.density(samples_inside)
        c_O_in = F_theta_O.color(samples_inside)
        sigma_G_in = F_theta_G.density(samples_inside)
        c_G_in = F_theta_G.color(samples_inside)
        
        # Distance smoothing
        center = get_roi_center(B)
        diag = get_box_diagonal(B)
        d_x = distance(samples_inside, center)
        f_x = 1 - np.exp(-alpha_smooth * d_x / diag)
        
        sigma_blend = f_x * sigma_O_in + (1 - f_x) * sigma_G_in
        c_blend = f_x * c_O_in + (1 - f_x) * c_G_in
        
        color_in, trans_in = volume_render(samples_inside, sigma_blend, c_blend)
        
        # Combine
        # Standard volume rendering combines transmittance
        final_color = color_out + trans_out * color_in
        final_trans = trans_out * trans_in
        
    return final_color
```

## Operating characteristics
*   **Time Complexity:** Training time depends on the number of iterations and the cost of querying two NeRF models per ray sample during the blending mode. The use of CLIP similarity loss requires rendering views and encoding them, which is computationally expensive compared to simple photometric losses.
*   **Memory Complexity:** Requires storing two NeRF models ($F_{\theta_O}$ and $F_{\theta_G}$) during training. $F_{\theta_O}$ is typically frozen or used for reference, while $F_{\theta_G}$ is optimized.
*   **Strengths:**
    *   Enables zero-shot editing of existing NeRFs using text prompts.
    *   Preserves the unedited scene regions perfectly ($x_e \odot (1 - B) = x_o \odot (1 - B)$).
    *   Does not require semantic masks or 2D segmentation for localization; uses a simple 3D box.
    *   Supports multiple editing modes: Object Insertion/Replacement and Object Blending.
*   **Limitations:**
    *   Quality depends heavily on the text prompt and the accuracy of the ROI box localization.
    *   May produce "amorphous" or "transparent" scenes if priors (transmittance/depth loss) are not properly annealed.
    *   Computationally intensive due to CLIP feedback loop and dual-model querying during blending.
*   **When to use:** When you need to edit a specific object or region in an existing NeRF scene based on a text description without retraining the entire scene or having semantic masks.
*   **When not to use:** When high-fidelity texture preservation of the edited region is required without any blending artifacts, or when computational resources are extremely limited.

## Background
Blended-NeRF was introduced by Ori Gordon, Omri Avrahami, and Dani Lischinski from The Hebrew University in 2023 [S1]. It builds upon the principles of Dream Fields, which used CLIP to guide NeRF generation, but adapts it for editing existing scenes rather than generating new ones from scratch. It addresses limitations in previous editing methods like Volumetric Disentanglement, which required semantic decomposition or 2D masks [S1].

## Sources
[S1] Blended-NeRF: Zero-Shot Object Generation and Blending in Existing Neural Radiance Fields, https://arxiv.org/abs/2306.12760v2, fetched 2026-10-26
