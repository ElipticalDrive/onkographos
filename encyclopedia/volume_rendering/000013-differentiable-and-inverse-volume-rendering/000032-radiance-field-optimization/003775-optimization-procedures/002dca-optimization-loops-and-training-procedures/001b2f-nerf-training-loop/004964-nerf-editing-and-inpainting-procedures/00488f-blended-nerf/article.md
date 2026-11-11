# Blended-NeRF

Also called: Zero-Shot Object Generation and Blending in Existing Neural Radiance Fields

## Overview
Blended-NeRF is a framework for editing a specific region of interest (ROI) in an existing Neural Radiance Field (NeRF) scene based on text prompts and a user-defined 3D ROI box [S1]. It solves the problem of local 3D editing by initializing a generator NeRF with the original scene's weights and blending the synthesized content with the original scene using volumetric blending [S1]. Unlike methods that synthesize new objects from scratch (e.g., Dream Fields) or require semantic decomposition/2D masks for localization, Blended-NeRF operates on any region using a 3D box without learning a new feature space [S1].

## How it works
**Inputs:**
*   Existing NeRF model $F_{\theta_O}$ representing scene $x_o$.
*   3D Region of Interest (ROI) box $B$ localized inside the scene.
*   Text prompt $y$ describing the desired edit.
*   Camera parameters for rendering.

**Outputs:**
*   Edited scene representation $x_e$ where $x_e \odot B$ is consistent with $y$ and $x_e \odot (1 - B) = x_o \odot (1 - B)$.

**Data Structures:**
*   $F_{\theta_O}$: MLP mapping 3D position and view direction to density $\sigma$ and color $c$.
*   $F_{\theta_G}$: Generator MLP, initialized with weights from $F_{\theta_O}$.
*   $B$: Axis-aligned 3D box defined by min/max coordinates.
*   $\gamma(x)$: Sinusoidal positional encoding function.

**Positional Encoding:**
Inputs are embedded into a high-dimensional space using high-frequency sinusoidal positional encoding [S1]:
$$ \gamma(x) = [\cos(2^l x), \sin(2^l x)]_{l=0}^{L-1} $$

**Training Steps:**
1.  **Initialization:** Initialize generator model $F_{\theta_G}$ with the weights of the original scene model $F_{\theta_O}$ [S1].
2.  **ROI Localization:** Localize the 3D box $B$ inside the scene using a GUI that renders the original NeRF and uses the output depth map for 3D understanding [S1].
3.  **Ray Sampling:** For each training iteration, sample a random camera pose around the scene. The initial distance $d$ from the ROI is calculated based on the maximum box dimension $e_{max}$ and camera FOV $\gamma$:
    $$ d = \frac{e_{max}}{2 \tan(\gamma/2)} $$
    Randomly sample radius $r$ around $d$ [S1]. Cast rays $r$ that pass through the box $B$.
4.  **Point Sampling:** Sample points $x_i$ along the rays. For points outside $B$, set density to zero [S1].
5.  **Rendering (ROI):** Render the generated content inside the ROI box $I_{ROI}$ using only points $x_i \in B$. The rendered color $C(r)$ for a ray $r$ is:
    $$ C(r) = \frac{\sum_{x_i \in B} T_i (1 - e^{-\sigma_i \delta_i}) c_i}{\sum_{x_i \in B} T_i (1 - e^{-\sigma_i \delta_i})} \quad \text{if } \exists x_i \in r \text{ s.t. } x_i \in B, \text{ else } 0 $$
    where $T_i = \exp(-\sum_{j < i} \sigma_j \delta_j)$ is the transmittance, $\delta_i = t_{i+1} - t_i$ is the distance between adjacent samples, $\sigma_i$ is density, and $c_i$ is color [S1].
6.  **Loss Computation:**
    *   **Similarity Loss:** Minimize cosine distance between image and text encoders of a language-image model (e.g., CLIP) [S1]:
        $$ L_{sim} = -E_{img}(I_{ROI})^T E_{txt}(y) $$
    *   **Transmittance Loss:** Encourage sparse/coherent results [S1]:
        $$ L_T = -\min(\tau, \text{mean}(T(P))) $$
        where $T(P)$ is the average transmittance of the rendered view from pose $P$, and $\tau$ is the max transmittance.
    *   **Depth Loss:** Encourage volumetric 3D shapes (prevents flat billboards) [S1]:
        $$ L_D = -\min(\rho, \sigma^2(D(P))) $$
        where $\sigma^2(D(P))$ is the variance of the disparity map of the rendered view from pose $P$, and $\rho$ is the max variance allowed.
    *   **Total Objective:**
        $$ L_{total} = L_{sim} + \lambda_T L_T + \lambda_D L_D $$
        where $\lambda_T, \lambda_D$ are weights. $L_T$ and $L_D$ are gradually introduced during training using an annealing strategy [S1].
7.  **Optimization:** Optimize $F_{\theta_G}$ by minimizing $L_{total}$ [S1].

**Blending Steps (Inference):**
After training, render the edited scene by blending sample points from $F_{\theta_O}$ and $F_{\theta_G}$ along each view ray [S1].

1.  **Query Models:** Query both $F_{\theta_O}$ and $F_{\theta_G}$ for the same set of rays. Points inside $B$ are rendered by $F_{\theta_G}$ and points outside by $F_{\theta_O}$ [S1].
2.  **Mode Selection:**
    *   **Object Insertion/Replacement:** Consider only the radiance field of $F_{\theta_G}$ inside the ROI during training. After training, blend using distance smoothing [S1].
    *   **Object Blending:** Query both models inside the box. Compute alpha values for each point $x_i$ on the ray separately from each model [S1]:
        $$ \alpha_O(x_i) = 1 - \exp(\phi(\sigma_O(x_i)) \cdot \delta_i) $$
        $$ \alpha_G(x_i) = 1 - \exp(\phi(\sigma_G(x_i)) \cdot \delta_i) $$
        where $\phi$ is a non-negative activation function [S1].
3.  **Density Blending (Object Blending Mode):** Two options for combined density $\sigma(x_i)$ [S1]:
    *   Option 1 (Sum in activation): $\sigma(x_i) = \phi(\sigma_O(x_i) + \sigma_G(x_i))$. Allows $F_{\theta_G}$ to remove original densities.
    *   Option 2 (Sum out activation): $\sigma(x_i) = \phi(\sigma_O(x_i)) + \phi(\sigma_G(x_i))$. Allows only addition of new densities.
4.  **Color Blending (Object Blending Mode):**
    $$ c(x_i) = S\left(\frac{c_O(x_i) \cdot \alpha_O(x_i) + c_G(x_i) \cdot \alpha_G(x_i)}{\epsilon + \alpha_O(x_i) + \alpha_G(x_i)}\right) $$
    where $S$ is the sigmoid function and $\epsilon$ is a small constant for numerical stability [S1].
5.  **Distance Smoothing (General Blending):** To get smooth blending, use a distance smoothing operator per point inside the box considering its distance $d(x)$ from the center of the ROI (center of mass, computed during training) and box diagonal $diag$ [S1]:
    $$ f(x) = 1 - \exp\left(-\alpha \frac{d(x)}{diag}\right) $$
    $$ \sigma_{blend}(x) = f(x) \cdot \sigma_O(x) + (1 - f(x)) \cdot \sigma_G(x) $$
    $$ c_{blend}(x) = f(x) \cdot c_O(x) + (1 - f(x)) \cdot c_G(x) $$
    where $\alpha$ is a hyperparameter controlling blending strength [S1].
6.  **Final Rendering:** The resulting raw densities and RGB values inside and outside the ROI are blended along each ray using the standard volume rendering equation (Eq. 1 in [S1]) to get the final rendered view $x_e$ [S1].

**Augmentations and Priors:**
*   **Background Augmentation:** Use random backgrounds (Gaussian noise, checkerboard, random Fourier textures) during optimization to prevent scene diffusion [S1].
*   **Directional Dependent Prompts:** Concatenate view-dependent text prompt to the original caption to constrain object description across views [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def positional_encoding(x, L=10):
    """High frequency sinusoidal positional encoding."""
    # x is shape (N, 3)
    freq_bands = 2.0 ** np.arange(L)
    encoded = []
    for l in range(L):
        freq = freq_bands[l]
        encoded.append(np.cos(freq * x))
        encoded.append(np.sin(freq * x))
    return np.concatenate(encoded, axis=-1)

def volume_rendering(rays, model, box_B, is_training_roi=False):
    """
    Render a batch of rays using a NeRF model.
    If is_training_roi is True, only samples inside box_B contribute.
    """
    # rays: (N, 2) [origin, direction]
    # model: callable (x, direction) -> (sigma, color)
    # box_B: AABB [min, max]
    
    # Sample points along rays (standard NeRF sampling)
    # t_vals: (N, N_samples)
    # points: (N, N_samples, 3)
    t_vals, points = sample_rays(rays, N_samples=256)
    
    # Compute deltas
    deltas = t_vals[:, 1:] - t_vals[:, :-1]
    # Add infinite delta for last point
    deltas = np.concatenate([deltas, np.array([1e10])], axis=1)
    
    # Evaluate model
    # Note: In practice, use batching for memory efficiency
    sigma_all, color_all = model(points)
    
    # Apply density mask if training ROI
    if is_training_roi:
        inside_box = np.all((points >= box_B.min) & (points <= box_B.max), axis=-1)
        sigma_all = sigma_all * inside_box.astype(np.float32)
        
    # Compute transmittance
    # alpha = 1 - exp(-sigma * delta)
    alpha = 1.0 - np.exp(-sigma_all * deltas)
    
    # Transmittance T_i = exp(-sum_{j<i} sigma_j * delta_j)
    # T = prod(1 - alpha)
    T = np.cumprod(1.0 - alpha, axis=1)
    # Shift T to align with samples: T_i corresponds to sample i
    # T[:, 0] is 1.0 (before first sample)
    T_shifted = np.concatenate([np.ones((T.shape[0], 1)), T[:, :-1]], axis=1)
    
    # Weighted sum
    weights = T_shifted * alpha
    color = np.sum(weights[:, :, np.newaxis] * color_all, axis=1)
    
    # If training ROI and no points inside box, return 0 or background
    if is_training_roi:
        has_points = np.any(np.all((points >= box_B.min) & (points <= box_B.max), axis=-1), axis=1)
        color = color * has_points[:, np.newaxis]
        
    return color, T_shifted, alpha, points, deltas

def blend_models(rays, model_O, model_G, box_B, alpha_smooth=2.0, mode='insertion'):
    """
    Blend results from original and generator models.
    """
    # Sample points
    t_vals, points = sample_rays(rays, N_samples=256)
    deltas = t_vals[:, 1:] - t_vals[:, :-1]
    deltas = np.concatenate([deltas, np.array([1e10])], axis=1)
    
    # Query models
    sigma_O, color_O = model_O(points)
    sigma_G, color_G = model_G(points)
    
    # Determine blend weights based on distance to ROI center
    roi_center = box_B.center
    diag = box_B.diagonal
    dists = np.linalg.norm(points - roi_center, axis=-1)
    f = 1.0 - np.exp(-alpha_smooth * dists / diag)
    
    if mode == 'insertion':
        # Use only generator inside box, original outside?
        # The paper says: "inside the ROI we consider only the radiance field of F_G during training. 
        # After training, we blend the two scenes as described above [distance smoothing]."
        # So we apply distance smoothing to both sigma and color
        sigma_blend = f * sigma_O + (1 - f) * sigma_G
        color_blend = f * color_O + (1 - f) * color_G
        alpha = 1.0 - np.exp(-sigma_blend * deltas)
        T = np.cumprod(1.0 - alpha, axis=1)
        T_shifted = np.concatenate([np.ones((T.shape[0], 1)), T[:, :-1]], axis=1)
        final_color = np.sum(T_shifted[:, :, np.newaxis] * alpha[:, :, np.newaxis] * color_blend, axis=1)
        
    elif mode == 'blending':
        # Compute alphas separately
        # phi is non-negative activation (e.g., softplus or relu)
        phi = lambda x: np.maximum(0, x) # Placeholder for specific activation
        alpha_O = 1.0 - np.exp(-phi(sigma_O) * deltas)
        alpha_G = 1.0 - np.exp(-phi(sigma_G) * deltas)
        
        # Blend colors
        epsilon = 1e-8
        numerator = color_O * alpha_O + color_G * alpha_G
        denominator = epsilon + alpha_O + alpha_G
        blended_color = np.sigmoid(numerator / denominator)
        
        # Blend densities (Option 2: sum out activation)
        sigma_blend = phi(sigma_O) + phi(sigma_G)
        alpha_blend = 1.0 - np.exp(-sigma_blend * deltas)
        
        T = np.cumprod(1.0 - alpha_blend, axis=1)
        T_shifted = np.concatenate([np.ones((T.shape[0], 1)), T[:, :-1]], axis=1)
        final_color = np.sum(T_shifted[:, :, np.newaxis] * alpha_blend[:, :, np.newaxis] * blended_color, axis=1)
        
    return final_color

def train_blended_nerf(model_O, box_B, text_prompt, num_iterations):
    """
    Train the generator model F_G initialized with F_O.
    """
    # Initialize F_G with weights from F_O
    model_G = copy_weights(model_O)
    
    optimizer = Adam(model_G.parameters())
    
    for i in range(num_iterations):
        # 1. Sample random camera pose
        # d = e_max / (2 * tan(gamma/2))
        # r = random around d
        pose = sample_random_pose(box_B, fov=60)
        
        # 2. Cast rays
        rays = generate_rays(pose, image_resolution=(800, 800))
        
        # 3. Render ROI content
        # Only samples inside box_B contribute to loss
        color_roi, T, alpha, points, deltas = volume_rendering(
            rays, model_G, box_B, is_training_roi=True
        )
        
        # 4. Compute Similarity Loss (CLIP)
        # E_img and E_txt are CLIP encoders
        img_emb = clip_image_encoder(color_roi)
        txt_emb = clip_text_encoder(text_prompt)
        L_sim = -np.dot(img_emb, txt_emb)
        
        # 5. Compute Priors (Annealed)
        # Transmittance Loss
        mean_T = np.mean(T)
        tau = 0.9
        L_T = -np.minimum(tau, mean_T)
        
        # Depth Loss (requires disparity map computation)
        # D(P) is disparity map
        disparity = compute_disparity(points, deltas, pose)
        var_D = np.var(disparity)
        rho = 1.0
        L_D = -np.minimum(rho, var_D)
        
        # Annealing weights
        lambda_T = anneal(i, start=0, end=1000)
        lambda_D = anneal(i, start=0, end=1000)
        
        L_total = L_sim + lambda_T * L_T + lambda_D * L_D
        
        # 6. Backprop
        optimizer.zero_grad()
        L_total.backward()
        optimizer.step()
        
        # Background Augmentation (optional, applied to rays not in ROI or as background)
        # Use random backgrounds (Gaussian, checkerboard, Fourier)
        
    return model_G
```

## Operating characteristics
*   **Time Complexity:** Training time depends on the number of iterations and the cost of CLIP inference per iteration. Ray marching complexity is similar to standard NeRF ($O(N \cdot M)$ where $N$ is rays, $M$ is samples).
*   **Memory Complexity:** Requires storing two NeRF models ($F_{\theta_O}$ and $F_{\theta_G}$) and their gradients.
*   **Strengths:** Allows local editing of any region in an existing NeRF without retraining the whole scene or requiring 2D masks/semantic decomposition [S1]. Generates view-consistent results [S1].
*   **Limitations:** Quality depends on the quality of the initial NeRF and the text prompt. May produce artifacts if the ROI box is poorly localized or if the prompt is ambiguous [S1]. The blending process can be complex to tune (e.g., choice of density blending option, $\alpha$ parameter) [S1].
*   **When to use:** When you need to insert, replace, or blend objects into an existing NeRF scene using text prompts, and you have a 3D box defining the region of interest.
*   **When not to use:** When global scene editing is required, or when high-fidelity semantic segmentation/masks are available and a different method (like Volumetric Disentanglement) is preferred for specific object removal tasks [S1].

## Background
Blended-NeRF was introduced by Ori Gordon, Omri Avrahami, and Dani Lischinski from The Hebrew University in 2023 [S1]. It builds upon previous work in text-guided NeRF generation (e.g., Dream Fields, DreamFusion) and NeRF editing (e.g., Volumetric Disentanglement) [S1]. The method addresses the limitations of prior editing techniques that required semantic decomposition or 2D masks, offering a zero-shot approach using a 3D ROI box and volumetric blending [S1].

## Sources
[S1] Blended-NeRF: Zero-Shot Object Generation and Blending in Existing Neural Radiance Fields, https://arxiv.org/abs/2306.12760v2, fetched 2026-10-26
