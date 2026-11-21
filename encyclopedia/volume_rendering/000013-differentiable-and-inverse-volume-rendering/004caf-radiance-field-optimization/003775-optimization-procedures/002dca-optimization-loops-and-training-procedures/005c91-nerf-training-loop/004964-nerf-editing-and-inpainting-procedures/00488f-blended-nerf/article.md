# Blended-NeRF

Also called: Zero-Shot Object Generation and Blending in Existing Neural Radiance Fields, Blending-NeRF

## Overview
Blended-NeRF is a framework for editing specific regions of interest (ROI) in an existing Neural Radiance Field (NeRF) scene using text prompts and a user-defined 3D bounding box [S1]. It differs from generative methods that synthesize objects from scratch (e.g., Dream Fields) by initializing a generator NeRF with the weights of the original scene and blending the synthesized content with the original scene using a novel volumetric blending technique [S1]. Unlike methods requiring semantic masks or feature decomposition, it operates on any region defined by a 3D box without learning a new feature space [S1].

## How it works
**Inputs:**
*   Pre-trained NeRF model $F_{\theta_O}: (\mathbf{x}, \mathbf{d}) \to (\sigma_O, \mathbf{c}_O)$ representing the original scene [S1].
*   A 3D Region of Interest (ROI) box $B$ localized within the scene [S1].
*   A text prompt $y$ describing the desired edit [S1].
*   A language-image model (e.g., CLIP) with image encoder $E_{img}$ and text encoder $E_{txt}$ [S1].

**Outputs:**
*   Edited NeRF model $F_{\theta_E}$ (implicitly defined by the blending of $F_{\theta_O}$ and $F_{\theta_G}$) [S1].
*   Rendered views of the edited scene [S1].

**Data Structures:**
*   $F_{\theta_O}$: MLP mapping $(\mathbf{x}, \mathbf{d})$ to density $\sigma$ and color $\mathbf{c}$ [S1].
*   $F_{\theta_G}$: MLP initialized with weights from $F_{\theta_O}$, mapping $(\mathbf{x}, \mathbf{d})$ to density $\sigma_G$ and color $\mathbf{c}_G$ [S1].
*   $B$: Axis-aligned 3D box defined by min/max coordinates [S1].
*   $\gamma(\mathbf{x})$: Sinusoidal positional encoding function [S1].

**Parameters:**
*   $\alpha$: Hyperparameter for distance smoothing blending strength. Controls the weight given to the original scene vs. the edited scene based on distance from the ROI center. Typical values: 0, 0.5, 2, 4, 10 [S1].
*   $\lambda_T, \lambda_D$: Weights for transmittance and depth losses [S1].
*   $\tau$: Max transmittance threshold for $L_T$ [S1].
*   $\rho$: Max variance threshold for $L_D$ [S1].
*   $\epsilon$: Small constant for numerical stability in blending [S1].
*   $\phi$: Non-negative activation function (e.g., ReLU) for densities [S1].
*   $S$: Sigmoid function [S1].
*   $e_{max}$: Maximum dimension of the ROI box [S1].
*   $\gamma_{fov}$: Camera field of view angle [S1].

**Numbered Steps:**

1.  **Initialization:** Initialize the generator NeRF $F_{\theta_G}$ with the weights of the original scene model $F_{\theta_O}$ [S1].
2.  **ROI Localization:** Localize the 3D box $B$ inside the scene using a GUI that renders the original NeRF and uses the output depth map for 3D understanding [S1].
3.  **Training Loop:** For each iteration:
    a. **Pose Sampling:** Sample a random camera pose around the scene with azimuth/elevation angles $(\theta, \phi)$. The initial distance $d$ from the ROI is calculated as $d = \frac{e_{max}}{2 \tan(\gamma_{fov}/2)}$, and the radius is randomly sampled around this value [S1].
    b. **Ray Casting:** Cast rays $r$ that pass through the box $B$. Sample points $\mathbf{x}_i$ along these rays [S1].
    c. **Forward Pass (Generator):** Query $F_{\theta_G}$ for points $\mathbf{x}_i \in B$. For points outside $B$, set density to zero [S1].
    d. **Forward Pass (Original):** Query $F_{\theta_O}$ for all points along the ray [S1].
    e. **Blending (Optional/Mode Dependent):** If performing "Object Blending" (adding to non-empty area), compute alpha values $\alpha_O(\mathbf{x}_i) = 1 - \exp(\phi(\sigma_O(\mathbf{x}_i)) \cdot \delta_i)$ and $\alpha_G(\mathbf{x}_i) = 1 - \exp(\phi(\sigma_G(\mathbf{x}_i)) \cdot \delta_i)$. Compute blended color $\mathbf{c}(\mathbf{x}_i) = S\left(\frac{\mathbf{c}_O(\mathbf{x}_i) \cdot \alpha_O(\mathbf{x}_i) + \mathbf{c}_G(\mathbf{x}_i) \cdot \alpha_G(\mathbf{x}_i)}{\epsilon + \alpha_O(\mathbf{x}_i) + \alpha_G(\mathbf{x}_i)}\right)$ [S1]. For density, choose one option:
        *   Option 1 (Sum in activation): $\sigma(\mathbf{x}_i) = \phi(\sigma_O(\mathbf{x}_i) + \sigma_G(\mathbf{x}_i))$ [S1].
        *   Option 2 (Sum out activation): $\sigma(\mathbf{x}_i) = \phi(\sigma_O(\mathbf{x}_i)) + \phi(\sigma_G(\mathbf{x}_i))$ [S1].
        *   If performing "Object Insertion/Replacement", consider only $F_{\theta_G}$ inside the ROI during training [S1].
    f. **Rendering:** Render the generated content inside the ROI $I_{ROI}$ using standard volume rendering on the blended/selected points:
        $$
        C(r) = \frac{\sum_{\mathbf{x}_i \in B} T_i (1 - e^{-\sigma_i \delta_i}) \mathbf{c}_i}{\sum_{\mathbf{x}_i \in B} T_i (1 - e^{-\sigma_i \delta_i})}
        $$
        where $T_i = \exp(-\sum_{j < i} \sigma_j \delta_j)$, if $\exists \mathbf{x}_i \in r$ s.t. $\mathbf{x}_i \in B$, else $0$ [S1].
    g. **Loss Calculation:**
        *   **Similarity Loss:** $L_{sim} = -E_{img}(I_{ROI})^T E_{txt}(y)$ [S1].
        *   **Transmittance Loss:** $L_T = -\min(\tau, \text{mean}(T(P)))$, where $T(P)$ is the transmittance of the rendered view from pose $P$ [S1].
        *   **Depth Loss:** $L_D = -\min(\rho, \sigma^2(D(P)))$, where $\sigma^2(D(P))$ is the variance of the disparity map of the rendered view from pose $P$ [S1].
        *   **Total Loss:** $L_{total} = L_{sim} + \lambda_T L_T + \lambda_D L_D$ [S1].
    h. **Optimization:** Update $\theta_G$ to minimize $L_{total}$ [S1].
    i. **Augmentations:** Use random backgrounds (Gaussian noise, checkerboard, random Fourier textures) to prevent scene diffusion [S1]. Use directional dependent prompts by concatenating view-dependent text to the original caption [S1]. Gradually introduce $L_T$ and $L_D$ during training using an annealing strategy to prevent completely transparent or amorphous scenes [S1].
4.  **Blending (Post-Training):** After training, render the final edited scene by blending the sample points generated by $F_{\theta_O}$ and $F_{\theta_G}$ along each view ray [S1].
    a. For points outside $B$, use $F_{\theta_O}$ [S1].
    b. For points inside $B$, apply distance smoothing blending:
        $$
        f(\mathbf{x}) = 1 - \exp\left(-\alpha \frac{d(\mathbf{x})}{diag}\right)
        $$
        $$
        \sigma_{blend}(\mathbf{x}) = f(\mathbf{x}) \cdot \sigma_O(\mathbf{x}) + (1 - f(\mathbf{x})) \cdot \sigma_G(\mathbf{x})
        $$
        $$
        \mathbf{c}_{blend}(\mathbf{x}) = f(\mathbf{x}) \cdot \mathbf{c}_O(\mathbf{x}) + (1 - f(\mathbf{x})) \cdot \mathbf{c}_G(\mathbf{x})
        $$
        where $d(\mathbf{x})$ is the Euclidean distance from the ROI center, $diag$ is the box diagonal, and $\alpha$ is the smoothing parameter [S1].
    c. Render the final view $x_e$ using standard volume rendering with the blended densities and colors [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def sinusoidal_encoding(x, L=10):
    """High frequency sinusoidal positional encoding."""
    freq_bands = 2.0 ** np.arange(L)
    encoded = []
    for l in range(L):
        freq = freq_bands[l]
        encoded.append(np.cos(freq * x))
        encoded.append(np.sin(freq * x))
    return np.concatenate(encoded, axis=-1)

def ray_box_intersection(origin, direction, box_min, box_max):
    """Returns t_min, t_max for ray-AABB intersection."""
    inv_dir = 1.0 / direction
    t1 = (box_min - origin) * inv_dir
    t2 = (box_max - origin) * inv_dir
    t_min = np.maximum(t1, t2).min(axis=1)
    t_max = np.minimum(t1, t2).max(axis=1)
    if np.any(t_max < 0) or np.any(t_min > t_max):
        return np.full(origin.shape[0], -1.0), np.full(origin.shape[0], -1.0)
    return t_min, t_max

def sample_camera_pose(r, theta, phi, center):
    """Samples a camera origin and direction on a sphere around center."""
    x = r * np.sin(phi) * np.cos(theta) + center[0]
    y = r * np.sin(phi) * np.sin(theta) + center[1]
    z = r * np.cos(phi) + center[2]
    origin = np.array([x, y, z])
    direction = center - origin
    direction = direction / np.linalg.norm(direction)
    return origin, direction

def volume_render(t_samples, sigma, color):
    """Standard volume rendering. Returns color, transmittance, alpha, delta_t."""
    delta_t = np.diff(t_samples, axis=1, prepend=np.zeros((t_samples.shape[0], 1)))
    delta_t = np.maximum(delta_t, 1e-8)
    alpha = 1.0 - np.exp(-sigma * delta_t)
    one_minus_alpha = 1.0 - alpha
    T_cum = np.ones_like(one_minus_alpha)
    T_cum[:, 1:] = np.cumprod(one_minus_alpha[:, :-1], axis=1)
    contribution = T_cum * alpha * color
    rendered_color = np.sum(contribution, axis=1)
    final_T = T_cum[:, -1] * (1 - alpha[:, -1])
    return rendered_color, T_cum, alpha, delta_t

def blended_nerf_edit(F_theta_O, F_theta_G, B_min, B_max, text_prompt, clip_model, 
                      num_iterations, alpha_smooth, lambda_T, lambda_D, tau, rho, epsilon):
    e_max = max(B_max[0] - B_min[0], B_max[1] - B_min[1], B_max[2] - B_min[2])
    gamma_fov = np.radians(60)
    d_init = e_max / (2 * np.tan(gamma_fov / 2))
    center = (B_min + B_max) / 2.0
    
    for _ in range(num_iterations):
        r = np.random.uniform(d_init * 0.8, d_init * 1.2)
        theta = np.random.uniform(0, 2 * np.pi)
        phi = np.random.uniform(-np.pi/2, np.pi/2)
        rays_origin, rays_direction = sample_camera_pose(r, theta, phi, center)
        
        t_min, t_max = ray_box_intersection(rays_origin, rays_direction, B_min, B_max)
        valid_mask = t_max > 0
        rays_origin = rays_origin[valid_mask]
        rays_direction = rays_direction[valid_mask]
        t_min = t_min[valid_mask]
        t_max = t_max[valid_mask]
        
        num_samples = 128
        t_samples = np.linspace(t_min, t_max, num_samples)
        samples_3d = rays_origin[:, np.newaxis, :] + rays_direction[:, np.newaxis, :] * t_samples[:, :, np.newaxis]
        
        mask_inside_B = (samples_3d >= B_min) & (samples_3d <= B_max)
        mask_inside_B = np.all(mask_inside_B, axis=2)
        
        points_G = samples_3d[mask_inside_B]
        sigma_G_raw = np.zeros_like(points_G[:, 0])
        color_G = np.zeros_like(points_G[:, :3])
        if len(points_G) > 0:
            pos_enc_G = sinusoidal_encoding(points_G)
            dir_enc_G = sinusoidal_encoding(rays_direction[mask_inside_B[:, 0]])
            sigma_G_raw, color_G = F_theta_G(pos_enc_G, dir_enc_G)
            
        points_O = samples_3d.reshape(-1, 3)
        pos_enc_O = sinusoidal_encoding(points_O)
        rays_dir_repeated = np.repeat(rays_direction[:, np.newaxis, :], num_samples, axis=1).reshape(-1, 3)
        dir_enc_O = sinusoidal_encoding(rays_dir_repeated)
        sigma_O_raw, color_O_raw = F_theta_O(pos_enc_O, dir_enc_O)
        
        sigma_O = sigma_O_raw.reshape(num_samples, -1).T
        color_O = color_O_raw.reshape(num_samples, -1, 3).transpose(1, 0, 2)
        
        sigma_G = np.zeros_like(sigma_O)
        color_G_resampled = np.zeros_like(color_O)
        sigma_G[mask_inside_B] = sigma_G_raw
        color_G_resampled[mask_inside_B] = color_G
        
        phi_act = lambda x: np.maximum(x, 0)
        delta_t = np.diff(t_samples, axis=1, prepend=np.zeros((t_samples.shape[0], 1)))
        delta_t = np.maximum(delta_t, 1e-8)
        
        alpha_O = 1.0 - np.exp(-phi_act(sigma_O) * delta_t)
        alpha_G = 1.0 - np.exp(-phi_act(sigma_G) * delta_t)
        
        blended_color = (color_O * alpha_O[:, :, np.newaxis] + color_G_resampled * alpha_G[:, :, np.newaxis]) / (epsilon + alpha_O[:, :, np.newaxis] + alpha_G[:, :, np.newaxis])
        blended_color = 1.0 / (1.0 + np.exp(-blended_color))
        
        sigma_blend = phi_act(sigma_O + sigma_G)
        
        rendered_color, T_cum, _, _ = volume_render(t_samples, sigma_blend, blended_color)
        
        roi_mask = np.any(mask_inside_B, axis=1)
        I_ROI = rendered_color[roi_mask]
        
        img_emb = clip_model.encode_image(I_ROI)
        txt_emb = clip_model.encode_text(text_prompt)
        L_sim = -np.dot(img_emb, txt_emb)
        
        mean_T = np.mean(T_cum[:, -1])
        L_T = -np.minimum(tau, mean_T)
        
        t_roi = t_samples[roi_mask]
        disparity = 1.0 / np.maximum(t_roi, 1e-8)
        L_D = -np.minimum(rho, np.var(disparity))
        
        L_total = L_sim + lambda_T * L_T + lambda_D * L_D
        F_theta_G.update_gradients(L_total)

    diag = np.sqrt(np.sum((B_max - B_min)**2))
    # Post-training blending would re-cast rays and apply f(x) smoothing
    # sigma_blend_final and color_blend_final are computed per point using f(x)
    return rendered_color
```

**Edge Cases and Pitfalls:**
*   **Scene Diffusion:** Using white/black backgrounds during optimization can cause the scene to populate the background. Use random backgrounds (Gaussian noise, checkerboard, random Fourier textures) [S1].
*   **Flat Geometry:** In forward-facing scenes, limited viewing intervals can lead to flat billboard geometry. Use depth loss to encourage volumetric shapes [S1].
*   **Transparency:** Without regularization, the scene can become completely transparent. Use transmittance loss and annealing [S1].
*   **View Consistency:** Without pose sampling, the object may not be consistent across views. Sample random camera poses [S1].
*   **Directional Dependence:** The object may look different in different views. Use directional dependent prompts [S1].

## Operating characteristics
*   **Time Complexity:** Depends on the number of iterations, number of rays, and number of samples per ray, similar to standard NeRF training [S1].
*   **Memory Complexity:** Requires storing the weights of $F_{\theta_O}$ and $F_{\theta_G}$ [S1].
*   **Strengths:** Can edit any region using a 3D box without semantic masks; blends naturally with the original scene; supports object insertion, replacement, and blending [S1].
*   **Limitations:** Requires a pre-trained NeRF; relies on CLIP for guidance, which may not always align perfectly with the user's intent; requires tuning of hyperparameters ($\alpha$, $\lambda_T$, $\lambda_D$) [S1].
*   **When to use:** When you want to edit a specific region of an existing NeRF scene using text prompts and a bounding box [S1].
*   **When not to use:** When you need to generate a new object from scratch without an existing scene (use Dream Fields); when you need semantic segmentation-based editing (use other methods) [S1].

## Background
Blended-NeRF was introduced by Ori Gordon, Omri Avrahami, and Dani Lischinski from The Hebrew University in 2023 [S1]. It builds on the principles of NeRF [Parent] and Dream Fields [S1], adapting them for local editing of existing scenes. It addresses the challenge of editing local regions in 3D scenes represented by NeRFs, which is difficult due to the implicit nature of the representation [S1].

## Sources
[S1] Blended-NeRF: Zero-Shot Object Generation and Blending in Existing Neural Radiance Fields, https://arxiv.org/abs/2306.12760v2, fetched 2026-10-26
[S2] A survey on 3D editing based on NeRF and 3DGS, https://link.springer.com/article/10.1007/s11704-025-41176-9, fetched 2026-10-28
