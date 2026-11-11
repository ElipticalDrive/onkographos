# Blended-NeRF

Also called: Zero-Shot Object Generation and Blending in Existing Neural Radiance Fields

## Overview
Blended-NeRF is a framework for editing a specific region of interest (ROI) in an existing Neural Radiance Field (NeRF) scene based on text prompts and a user-defined 3D ROI box [S1]. It solves the problem of local 3D editing by initializing a generator NeRF with the weights of the original scene and blending the synthesized content with the original scene using volumetric blending [S1]. Unlike methods that synthesize objects from scratch or require semantic decomposition, Blended-NeRF operates on any region using a 3D box without learning a new feature space or requiring 2D masks [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** An existing NeRF scene represented by a model $F_{\theta_O}$, a 3D ROI box $B$ localized inside the scene, and a user-provided text prompt $y$ [S1].
*   **Output:** An edited scene $x_e$ where the content inside $B$ is consistent with $y$, and the content outside $B$ is preserved ($x_e \odot (1 - B) = x_o \odot (1 - B)$) [S1].

**Data Structures:**
*   **NeRF Model ($F_\theta$):** A Multi-Layer Perceptron (MLP) mapping 3D coordinates and viewing directions to density ($\sigma$) and color ($c$) [S1].
*   **Positional Encoding:** Inputs are embedded using high-frequency sinusoidal positional encoding $\gamma(x) = [\cos(2^l x), \sin(2^l x)]_{l=0}^{L-1}$ [S1].
*   **ROI Box ($B$):** Defined by a center and dimensions, localized via a GUI using the original NeRF's depth map [S1].

**Parameters:**
*   $\alpha$: Hyperparameter for distance smoothing blending strength. $\alpha=0$ gives all weight to the edited scene; higher values give more attention to the original scene points closer to the ROI center. Typical values demonstrated: 0, 0.5, 2, 4, 10 [S1].
*   $\lambda_T, \lambda_D$: Weights for transmittance and depth losses [S1].
*   $\tau$: Max transmittance for transmittance loss [S1].
*   $\rho$: Max variance for depth loss [S1].
*   $\epsilon$: Small constant for numerical stability [S1].
*   $\gamma$: Camera FOV [S1].
*   $e_{max}$: Max dimension of the ROI box [S1].

**Algorithm Steps:**

1.  **Initialization:** Initialize a generator NeRF model $F_{\theta_G}$ with the weights of the original scene model $F_{\theta_O}$ [S1].
2.  **ROI Localization:** Localize a 3D box $B$ inside the scene using a GUI that renders the original NeRF and uses the output depth map for 3D understanding [S1].
3.  **Pose Sampling:** For each training iteration, sample a random camera pose around the scene with azimuth/elevation angles $(\theta, \phi)$. The initial distance $d$ is calculated as $d = \frac{e_{max}}{2 \tan(\gamma/2)}$, where $e_{max}$ is the max box dimension and $\gamma$ is the camera FOV. Randomly sample radius $r$ around $d$ [S1].
4.  **Ray Casting and Sampling:** Cast rays $r$ from the sampled camera pose. Sample points $x_i$ along the ray.
5.  **Training Mode Selection:**
    *   **Object Insertion/Replacement:** Consider only the radiance field of $F_{\theta_G}$ inside the ROI. For points outside $B$, set density to zero. Render generated content $I_{ROI}$ using only points $x_i \in B$ [S1].
    *   **Object Blending:** Query both $F_{\theta_O}$ and $F_{\theta_G}$ inside the box. Compute alpha values for each point $x_i$:
        $$ \alpha_O(x_i) = 1 - \exp(\phi(\sigma_O(x_i)) \cdot \delta_i) $$
        $$ \alpha_G(x_i) = 1 - \exp(\phi(\sigma_G(x_i)) \cdot \delta_i) $$
        where $\phi$ is a non-negative activation function [S1].
        Compute blended color:
        $$ c(x_i) = S\left(\frac{c_O(x_i) \cdot \alpha_O(x_i) + c_G(x_i) \cdot \alpha_G(x_i)}{\epsilon + \alpha_O(x_i) + \alpha_G(x_i)}\right) $$
        where $S$ is the sigmoid function [S1].
        Compute blended density using one of two options [S1]:
        *   Option 1 (Sum in activation): $\sigma(x_i) = \phi(\sigma_O(x_i) + \sigma_G(x_i))$. Allows $F_{\theta_G}$ to remove original densities [S1].
        *   Option 2 (Sum out activation): $\sigma(x_i) = \phi(\sigma_O(x_i)) + \phi(\sigma_G(x_i))$. Allows only addition of new densities [S1].
6.  **Loss Calculation:**
    *   **Similarity Loss:** Minimize cosine distance between rendered ROI image $I_{ROI}$ and text prompt $y$ using CLIP encoders $E_{img}$ and $E_{txt}$ [S1]:
        $$ L_{sim} = -E_{img}(I_{ROI})^T E_{txt}(y) $$
    *   **Transmittance Loss:** Encourage sparse transmittance [S1]:
        $$ L_T = -\min(\tau, \text{mean}(T(P))) $$
        where $T(P)$ is the transmittance of the rendered view from pose $P$ [S1].
    *   **Depth Loss:** Encourage volumetric geometry (especially for forward-facing scenes) [S1]:
        $$ L_D = -\min(\rho, \sigma^2(D(P))) $$
        where $\sigma^2(D(P))$ is the variance of the disparity map of the rendered view from pose $P$ [S1].
    *   **Total Loss:**
        $$ L_{total} = L_{sim} + \lambda_T L_T + \lambda_D L_D $$
        $L_T$ and $L_D$ are gradually introduced during training using an annealing strategy [S1].
7.  **Optimization:** Optimize $F_{\theta_G}$ by minimizing $L_{total}$ [S1].
8.  **Blending (Inference):** After training, render the edited scene by blending sample points from $F_{\theta_O}$ and $F_{\theta_G}$ along each view ray. Points inside the box are rendered by $F_{\theta_G}$ and points outside by $F_{\theta_O}$ [S1].
    *   Apply distance smoothing operator for points $x$ inside the ROI [S1]:
        $$ f(x) = 1 - \exp\left(-\alpha \frac{d(x)}{diag}\right) $$
        $$ \sigma_{blend}(x) = f(x) \cdot \sigma_O(x) + (1 - f(x)) \cdot \sigma_G(x) $$
        $$ c_{blend}(x) = f(x) \cdot c_O(x) + (1 - f(x)) \cdot c_G(x) $$
        where $d(x)$ is the Euclidean distance from the ROI center, $diag$ is the box diagonal, and $\alpha$ is the smoothing parameter [S1].
    *   Render the final view $x_e$ using standard volume rendering on the blended densities and colors [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def positional_encoding(x, L=10):
    """High frequency sinusoidal positional encoding."""
    # x is shape (..., 3)
    # Returns shape (..., 3 * 2 * L)
    freq_bands = 2.0 ** np.arange(L)
    encoded = []
    for l in range(L):
        freq = freq_bands[l]
        encoded.append(np.cos(freq * x))
        encoded.append(np.sin(freq * x))
    return np.concatenate(encoded, axis=-1)

def volume_rendering(rays_origin, rays_direction, samples_t, sigma, c, T_prev=None):
    """
    Standard volume rendering.
    rays_origin: (N, 3)
    rays_direction: (N, 3)
    samples_t: (N, K) - distances along ray
    sigma: (N, K) - density
    c: (N, K, 3) - color
    T_prev: (N,) - transmittance from previous segments (outside ROI)
    """
    deltas = samples_t[:, 1:] - samples_t[:, :-1]
    deltas = np.concatenate([deltas, np.array([1e10])], axis=1) # Handle last segment
    
    # Alpha
    alpha = 1.0 - np.exp(-sigma * deltas)
    
    # Transmittance
    # T_i = exp(-sum_{j=1}^{i-1} sigma_j * delta_j)
    # We need to accumulate from the start of the ray
    # Assuming samples_t are sorted and continuous
    accumulated_sigma_delta = np.cumsum(sigma * deltas, axis=1)
    T = np.exp(-accumulated_sigma_delta)
    
    # Weight
    weights = T * alpha[:, :, np.newaxis]
    
    # Color
    C = np.sum(weights * c, axis=1)
    
    # Transmittance at end
    T_final = T[:, -1]
    
    return C, T_final

def blended_nerf_training(F_theta_O, F_theta_G, B, text_prompt, num_iterations, clip_model):
    """
    F_theta_O: Original NeRF model
    F_theta_G: Generator NeRF model (initialized with F_theta_O weights)
    B: ROI Box (center, dimensions)
    text_prompt: String
    clip_model: CLIP model with encode_image and encode_text
    """
    optimizer = Adam(F_theta_G.parameters())
    
    for i in range(num_iterations):
        # 1. Pose Sampling
        theta = np.random.uniform(0, 2 * np.pi)
        phi = np.random.uniform(-np.pi/2, np.pi/2)
        # Calculate initial distance
        e_max = max(B.dimensions)
        gamma = camera_fov # Assumed known
        d_init = e_max / (2 * np.tan(gamma / 2))
        r = np.random.normal(d_init, d_init * 0.1) # Random radius around d
        # Convert spherical to Cartesian for camera position relative to B.center
        cam_pos = B.center + r * np.array([np.sin(phi)*np.cos(theta), np.sin(phi)*np.sin(theta), np.cos(phi)])
        cam_target = B.center
        rays_origin, rays_direction = generate_rays(cam_pos, cam_target, image_resolution)
        
        # 2. Ray Casting and Sampling
        # Sample points along rays
        t_min, t_max = get_ray_bounds(rays_origin, rays_direction, B)
        samples_t = np.linspace(t_min, t_max, num_samples)
        samples_xyz = rays_origin[:, :, np.newaxis] + rays_direction[:, :, np.newaxis] * samples_t[np.newaxis, np.newaxis, :]
        
        # 3. Query Models
        # Encode inputs
        samples_xyz_enc = positional_encoding(samples_xyz)
        dirs_enc = positional_encoding(rays_direction[:, :, np.newaxis])
        
        sigma_O, c_O = F_theta_O(samples_xyz_enc, dirs_enc)
        sigma_G, c_G = F_theta_G(samples_xyz_enc, dirs_enc)
        
        # 4. Compute Losses
        # Similarity Loss (using ROI mask)
        roi_mask = is_inside_box(samples_xyz, B) # (N, K)
        
        # Render ROI content
        # For Insertion/Replacement: use only G inside B, 0 outside
        # For Blending: use blended G and O inside B
        
        # Example for Blending Mode
        alpha_O = 1.0 - np.exp(sigma_O) # Assuming delta=1 for simplicity or use actual deltas
        alpha_G = 1.0 - np.exp(sigma_G)
        
        # Blend colors
        blended_c = sigmoid((c_O * alpha_O[:, :, np.newaxis] + c_G * alpha_G[:, :, np.newaxis]) / (epsilon + alpha_O[:, :, np.newaxis] + alpha_G[:, :, np.newaxis]))
        
        # Blend densities (Option 2: Sum out activation)
        blended_sigma = relu(sigma_O) + relu(sigma_G)
        
        # Render blended ROI
        # Note: In practice, render only points inside B for L_sim
        roi_samples_t = samples_t[roi_mask]
        roi_blended_c = blended_c[roi_mask]
        roi_blended_sigma = blended_sigma[roi_mask]
        
        # Render to get I_ROI
        # Simplified: assume we render a view and mask it
        I_ROI, _ = volume_rendering(rays_origin, rays_direction, samples_t, blended_sigma, blended_c)
        I_ROI_masked = I_ROI * roi_mask.mean(axis=1) # Simplified masking for loss
        
        img_emb = clip_model.encode_image(I_ROI_masked)
        txt_emb = clip_model.encode_text(text_prompt)
        L_sim = -np.dot(img_emb, txt_emb)
        
        # Transmittance Loss
        T_mean = np.mean(volume_rendering(rays_origin, rays_direction, samples_t, blended_sigma, blended_c)[1])
        L_T = -np.minimum(tau, T_mean)
        
        # Depth Loss
        disparity_map = compute_disparity(rays_origin, rays_direction, samples_t, blended_sigma)
        L_D = -np.minimum(rho, np.var(disparity_map))
        
        # Total Loss
        L_total = L_sim + lambda_T * L_T + lambda_D * L_D
        
        # Backprop
        optimizer.zero_grad()
        L_total.backward()
        optimizer.step()

def blended_nerf_rendering(F_theta_O, F_theta_G, B, alpha_smooth, rays_origin, rays_direction):
    """
    Render the final edited scene.
    """
    # Sample points
    samples_t = ...
    samples_xyz = ...
    
    sigma_O, c_O = F_theta_O(samples_xyz)
    sigma_G, c_G = F_theta_G(samples_xyz)
    
    # Apply Distance Smoothing inside B
    d_x = distance_to_center(samples_xyz, B.center)
    f_x = 1.0 - np.exp(-alpha_smooth * d_x / B.diagonal)
    
    sigma_blend = np.where(is_inside_box(samples_xyz, B), 
                           f_x * sigma_O + (1 - f_x) * sigma_G, 
                           sigma_O)
    c_blend = np.where(is_inside_box(samples_xyz, B), 
                       f_x * c_O + (1 - f_x) * c_G, 
                       c_O)
    
    # Standard Volume Rendering
    C, _ = volume_rendering(rays_origin, rays_direction, samples_t, sigma_blend, c_blend)
    return C
```

**Edge Cases and Pitfalls:**
*   **Scene Diffusion:** Without background augmentation, the scene may populate the background with noise. Use random backgrounds (Gaussian noise, checkerboard, random Fourier textures) during optimization [S1].
*   **Flat Geometry:** In forward-facing scenes, the edit may result in a flat billboard effect. Use depth loss to encourage volumetric geometry [S1].
*   **Transparency:** Without transmittance loss, the scene may become completely transparent. Use transmittance loss to encourage sparsity [S1].
*   **View Inconsistency:** Use directional dependent prompts (concatenating view-dependent text to the original caption) to constrain object description across views [S1].

## Operating characteristics
*   **Time Complexity:** Depends on the number of training iterations, number of rays, and number of samples per ray. Similar to standard NeRF training but with additional queries to two models during blending mode.
*   **Memory Complexity:** Requires storing parameters for both $F_{\theta_O}$ and $F_{\theta_G}$.
*   **Strengths:**
    *   Enables local editing in any region using a 3D box without requiring 2D masks or semantic decomposition [S1].
    *   Preserves the original scene outside the ROI [S1].
    *   Generates natural-looking and view-consistent results using volumetric blending [S1].
    *   Supports object insertion, replacement, and blending [S1].
*   **Limitations:**
    *   Requires a pre-trained NeRF scene [S1].
    *   Quality depends on the text prompt and the accuracy of the ROI box localization [S1].
    *   May require careful tuning of hyperparameters ($\alpha$, $\lambda_T$, $\lambda_D$) [S1].
*   **When to use:** When you need to edit a specific object or region in an existing NeRF scene based on a text prompt, and you have a rough 3D bounding box for that region.
*   **When not to use:** When you need to synthesize a new scene from scratch, or when you have precise 2D masks and semantic labels for the region (other methods might be more efficient).

## Background
Blended-NeRF was introduced by Ori Gordon, Omri Avrahami, and Dani Lischinski from The Hebrew University in 2023 [S1]. It builds on the principles of Dream Fields [27] and CLIP-NeRF [67], adapting their text-guided generation and priors to the context of editing existing NeRF scenes. It addresses the limitations of previous editing methods that required semantic decomposition or 2D masks [S1].

## Sources
[S1] Blended-NeRF: Zero-Shot Object Generation and Blending in Existing Neural Radiance Fields, https://arxiv.org/abs/2306.12760v2, fetched 2026-10-26
