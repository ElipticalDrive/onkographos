# Blended-NeRF

Also called: Zero-Shot Object Generation and Blending in Existing Neural Radiance Fields

## Overview
Blended-NeRF is a framework for editing a specific region of interest (ROI) in an existing Neural Radiance Field (NeRF) scene based on text prompts and a 3D ROI box [S1]. It solves the problem of local 3D editing by initializing a generator NeRF with the original scene's weights and blending the synthesized content with the original scene using volumetric blending [S1]. Unlike methods that synthesize new objects from scratch or require semantic decomposition/2D masks for localization, Blended-NeRF operates on any region using a 3D box without learning a new feature space [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** An existing NeRF scene $F_{\theta_O}$ representing scene $x_o$, a 3D ROI box $B$ localized inside the scene, and a user-provided text prompt $y$ [S1].
*   **Output:** An edited scene $x_e$ where $x_e \odot B$ is consistent with the text prompt and $x_e \odot (1 - B) = x_o \odot (1 - B)$ [S1].

**Data Structures:**
*   **NeRF Model ($F_\theta$):** A 3D Multi-Layer Perceptron (MLP) mapping 3D position $\mathbf{x}$ and viewing direction $\mathbf{d}$ to volume density $\sigma$ and color $\mathbf{c}$ [S1].
*   **Positional Encoding:** Inputs are embedded using high-frequency sinusoidal positional encoding $\gamma(\mathbf{x}) = [\cos(2^l \mathbf{x}), \sin(2^l \mathbf{x})]_{l=0}^{L-1}$ [S1].
*   **Language-Image Model:** Pretrained model (e.g., CLIP) with image encoder $E_{img}$ and text encoder $E_{txt}$ [S1].

**Parameters:**
*   **$\alpha$:** Hyperparameter for distance smoothing blending strength. $\alpha=0$ gives all weight to the edited scene; higher $\alpha$ gives more attention to original scene points closer to the ROI center. Typical values demonstrated: 0, 0.5, 2, 4, 10 [S1].
*   **$\lambda_T, \lambda_D$:** Weights for transmittance and depth losses [S1].
*   **$\tau$:** Max transmittance for transmittance loss [S1].
*   **$\rho$:** Max variance allowed for depth loss [S1].
*   **$\epsilon$:** Small constant for numerical stability in blending [S1].
*   **$\gamma$:** Camera Field of View (FOV) [S1].
*   **$e_{max}$:** Maximum dimension of the ROI box [S1].

**Numbered Steps:**

1.  **Initialization:** Initialize a 3D MLP generator model $F_{\theta_G}$ with the weights of the original scene model $F_{\theta_O}$ [S1].
2.  **ROI Localization:** Localize a 3D box $B$ inside the scene using a GUI that renders the original NeRF and uses the output depth map for 3D understanding [S1].
3.  **Pose Sampling:** For each training iteration, sample a random camera pose around the scene with azimuth/elevation angles $(\theta, \phi)$. The initial distance $d$ is calculated as $d = \frac{e_{max}}{2 \tan(\gamma/2)}$, where $e_{max}$ is the max box dimension and $\gamma$ is the camera FOV. Randomly sample radius $r$ around $d$ [S1].
4.  **Ray Casting and Sampling:** Cast rays $r$ that pass through the box $B$. Sample points along the ray. For sample points outside $B$, set density to zero [S1].
5.  **Rendering Generated Content:** Render the generated content inside the ROI box $I_{ROI}$ using only points $\mathbf{x}_i \in B$. The rendered color $C(r)$ for a ray $r$ is:
    $$
    C(r) = \begin{cases} 
    \frac{\sum_{\mathbf{x}_i \in B} T_i (1 - e^{-\sigma_i \delta_i}) c_i}{\sum_{\mathbf{x}_i \in B} T_i (1 - e^{-\sigma_i \delta_i})} & \text{if } \exists \mathbf{x}_i \in r \text{ s.t. } \mathbf{x}_i \in B \\
    0 & \text{otherwise}
    \end{cases}
    $$
    where $T_i = \exp(-\sum_{j < i} \sigma_j \delta_j)$ [S1].
6.  **Loss Computation:** Optimize $F_{\theta_G}$ by minimizing the total objective $L_{total} = L_{sim} + \lambda_T L_T + \lambda_D L_D$ [S1].
    *   **Similarity Loss ($L_{sim}$):** Align generated content with the text prompt $y$.
        $$
        L_{sim} = -E_{img}(I_{ROI})^T E_{txt}(y)
        $$
        where $E_{img}$ and $E_{txt}$ are image and text encoders [S1].
    *   **Transmittance Loss ($L_T$):** Encourage sparse/coherent results.
        $$
        L_T = -\min(\tau, \text{mean}(T(P)))
        $$
        where $T(P)$ is transmittance of rendered view from pose $P$ [S1].
    *   **Depth Loss ($L_D$):** Encourage volumetric 3D shapes (prevents flat billboard effect).
        $$
        L_D = -\min(\rho, \sigma^2(D(P)))
        $$
        where $\sigma^2(D(P))$ is variance of disparity map of rendered view from pose $P$ [S1].
    *   **Annealing:** Gradually introduce $L_T$ and $L_D$ during training to prevent completely transparent or amorphous scenes [S1].
7.  **Augmentations:**
    *   **Background Augmentation:** Use random backgrounds (Gaussian noise, checkerboard, random Fourier textures) during optimization to prevent scene diffusion [S1].
    *   **Directional Dependent Prompts:** Concatenate view-dependent text prompt to original caption to constrain object description across views [S1].
8.  **Blending (Post-Training):** After training, render the edited scene by blending sample points generated by $F_{\theta_O}$ and $F_{\theta_G}$ along each view ray [S1].
    *   **Object Insertion/Replacement Mode:** During training inside ROI, consider only radiance field of $F_{\theta_G}$. After training, blend using distance smoothing [S1].
    *   **Object Blending Mode:** Query both $F_{\theta_O}$ and $F_{\theta_G}$ inside the box. Compute alpha values:
        $$
        \alpha_O(\mathbf{x}_i) = 1 - \exp(\phi(\sigma_O(\mathbf{x}_i)) \cdot \delta_i), \quad \alpha_G(\mathbf{x}_i) = 1 - \exp(\phi(\sigma_G(\mathbf{x}_i)) \cdot \delta_i)
        $$
        where $\phi$ is non-negative activation [S1].
    *   **Density Blending Options:**
        *   Option 1 (Sum in activation): $\sigma(\mathbf{x}_i) = \phi(\sigma_O(\mathbf{x}_i) + \sigma_G(\mathbf{x}_i))$, allowing $F_{\theta_G}$ to remove original densities [S1].
        *   Option 2 (Sum out activation): $\sigma(\mathbf{x}_i) = \phi(\sigma_O(\mathbf{x}_i)) + \phi(\sigma_G(\mathbf{x}_i))$, allowing only addition of new densities [S1].
    *   **Color Blending:**
        $$
        c(\mathbf{x}_i) = S\left(\frac{c_O(\mathbf{x}_i) \cdot \alpha_O(\mathbf{x}_i) + c_G(\mathbf{x}_i) \cdot \alpha_G(\mathbf{x}_i)}{\epsilon + \alpha_O(\mathbf{x}_i) + \alpha_G(\mathbf{x}_i)}\right)
        $$
        where $S$ is sigmoid, $\epsilon$ is small constant [S1].
    *   **Distance Smoothing:** For points inside the ROI, blend densities and colors based on distance $d(\mathbf{x})$ from ROI center and box diagonal $diag$:
        $$
        f(\mathbf{x}) = 1 - \exp\left(-\alpha \frac{d(\mathbf{x})}{diag}\right)
        $$
        $$
        \sigma_{blend}(\mathbf{x}) = f(\mathbf{x}) \cdot \sigma_O(\mathbf{x}) + (1 - f(\mathbf{x})) \cdot \sigma_G(\mathbf{x})
        $$
        $$
        c_{blend}(\mathbf{x}) = f(\mathbf{x}) \cdot c_O(\mathbf{x}) + (1 - f(\mathbf{x})) \cdot c_G(\mathbf{x})
        $$
        [S1].
    *   **Final Render:** The final rendered view $x_e$ is obtained by blending the raw densities and RGB values inside and outside the ROI along each ray using standard volume rendering [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def positional_encoding(x, L=10):
    """High frequency sinusoidal positional encoding."""
    # x is shape (N, 3)
    # Returns shape (N, 3 * 2 * L)
    encodings = []
    for l in range(L):
        freq = 2**l
        encodings.append(np.cos(freq * x))
        encodings.append(np.sin(freq * x))
    return np.concatenate(encodings, axis=-1)

def volume_rendering(rays_o, rays_d, samples_t, sigma, c):
    """
    Standard volume rendering.
    rays_o: (N, 3) ray origins
    rays_d: (N, 3) ray directions
    samples_t: (N, K) sample distances along rays
    sigma: (N, K) volume densities
    c: (N, K, 3) colors
    """
    # Compute deltas
    deltas = samples_t[:, 1:] - samples_t[:, :-1]
    # Add last delta (or handle boundary) - assuming samples_t includes endpoints or we handle K-1 samples
    # For simplicity, assuming samples_t has K points, we compute K-1 intervals
    # Or if samples_t are centers, we need half-deltas. 
    # Based on NeRF standard: samples are points, deltas are distances between them.
    # Let's assume samples_t has K points, we compute K-1 intervals.
    # To match formula: sum from i=1 to N. Here N is K-1 intervals.
    
    # Transmittance T_i = exp(-sum_{j=1}^{i-1} sigma_j * delta_j)
    # alpha_i = 1 - exp(-sigma_i * delta_i)
    
    # Compute alpha
    alpha = 1.0 - np.exp(-sigma * deltas) # (N, K-1)
    
    # Compute T
    # T_1 = 1
    # T_i = T_{i-1} * (1 - alpha_{i-1})
    # Using cumulative product
    T = np.cumprod(1.0 - alpha, axis=1)
    T = np.concatenate([np.ones((T.shape[0], 1)), T[:, :-1]], axis=1) # T_i corresponds to interval i
    
    # Weighted sum
    weights = T * alpha
    color = np.sum(weights[:, :, np.newaxis] * c[:, :-1, :], axis=1)
    
    return color

def blended_nerf_training(F_theta_O, F_theta_G, B, text_prompt, num_iters, lr):
    """
    F_theta_O: Original NeRF model
    F_theta_G: Generator NeRF model (initialized with F_theta_O weights)
    B: 3D ROI box
    text_prompt: User text prompt
    """
    optimizer = Adam(F_theta_G.parameters(), lr=lr)
    
    for i in range(num_iters):
        # 1. Pose Sampling
        # Sample random camera pose around scene
        # d = e_max / (2 * tan(gamma/2))
        # r = random around d
        # theta, phi = random angles
        pose = sample_random_pose(B, gamma=60.0) # Example gamma
        
        # 2. Ray Casting
        rays_o, rays_d = cast_rays_from_pose(pose, image_resolution)
        
        # 3. Sampling
        # Sample points along rays
        # Filter points inside B
        # For points outside B, set density to 0 during training of F_theta_G
        samples_t = sample_points_along_rays(rays_o, rays_d, num_samples=192)
        
        # 4. Forward Pass F_theta_G
        # Only render inside ROI for similarity loss
        # Mask points outside B
        inside_B_mask = is_inside_box(samples_t, rays_o, rays_d, B)
        
        # Query F_theta_G
        # Encode positions
        pos_enc = positional_encoding(samples_t[inside_B_mask])
        # Query MLP
        sigma_G, c_G = F_theta_G(pos_enc, rays_d[inside_B_mask])
        
        # Set density to 0 outside B
        sigma_G_full = np.zeros_like(sigma_G)
        c_G_full = np.zeros_like(c_G)
        sigma_G_full[inside_B_mask] = sigma_G
        c_G_full[inside_B_mask] = c_G
        
        # Render I_ROI
        # Use only points inside B
        I_ROI = volume_rendering(rays_o, rays_d, samples_t, sigma_G_full, c_G_full)
        
        # 5. Loss Computation
        # Similarity Loss
        E_img = clip_image_encoder(I_ROI)
        E_txt = clip_text_encoder(text_prompt)
        L_sim = -np.dot(E_img, E_txt)
        
        # Transmittance Loss
        # Compute transmittance T for the view
        T_view = compute_transmittance(rays_o, rays_d, samples_t, sigma_G_full)
        L_T = -np.min(tau, np.mean(T_view))
        
        # Depth Loss
        # Compute disparity map D(P)
        D_P = compute_disparity_map(rays_o, rays_d, samples_t, sigma_G_full)
        L_D = -np.min(rho, np.var(D_P))
        
        # Total Loss
        L_total = L_sim + lambda_T * L_T + lambda_D * L_D
        
        # 6. Optimization
        optimizer.zero_grad()
        L_total.backward()
        optimizer.step()

def blended_nerf_rendering(F_theta_O, F_theta_G, B, pose, alpha_smooth):
    """
    Render edited scene by blending F_theta_O and F_theta_G.
    """
    rays_o, rays_d = cast_rays_from_pose(pose, image_resolution)
    samples_t = sample_points_along_rays(rays_o, rays_d, num_samples=192)
    
    # Query both models
    sigma_O, c_O = F_theta_O(samples_t, rays_d)
    sigma_G, c_G = F_theta_G(samples_t, rays_d)
    
    # Determine blend mode
    # Object Blending Mode: Query both inside box
    inside_B_mask = is_inside_box(samples_t, rays_o, rays_d, B)
    
    # Compute alpha values for each model
    # phi is non-negative activation (e.g., softplus or relu)
    alpha_O = 1.0 - np.exp(-phi(sigma_O) * deltas)
    alpha_G = 1.0 - np.exp(-phi(sigma_G) * deltas)
    
    # Blend colors
    epsilon = 1e-8
    c_blend = sigmoid((c_O * alpha_O + c_G * alpha_G) / (epsilon + alpha_O + alpha_G))
    
    # Blend densities
    # Option 1: Sum in activation
    sigma_blend = phi(sigma_O + sigma_G)
    # Option 2: Sum out activation
    # sigma_blend = phi(sigma_O) + phi(sigma_G)
    
    # Distance Smoothing for points inside B
    center_B = compute_center(B)
    diag_B = compute_diagonal(B)
    d_x = compute_distance(samples_t, center_B)
    f_x = 1.0 - np.exp(-alpha_smooth * d_x / diag_B)
    
    # Apply smoothing only inside B
    sigma_final = sigma_blend.copy()
    c_final = c_blend.copy()
    
    sigma_final[inside_B_mask] = f_x[inside_B_mask] * sigma_O[inside_B_mask] + (1 - f_x[inside_B_mask]) * sigma_G[inside_B_mask]
    c_final[inside_B_mask] = f_x[inside_B_mask] * c_O[inside_B_mask] + (1 - f_x[inside_B_mask]) * c_G[inside_B_mask]
    
    # Final Volume Rendering
    # Use standard volume rendering with blended sigma and c
    # Note: The paper says "blending the raw densities and RGB values inside and outside the ROI along each ray using eq. 1"
    # This implies we treat the blended field as the final field.
    final_color = volume_rendering(rays_o, rays_d, samples_t, sigma_final, c_final)
    
    return final_color
```

## Operating characteristics
*   **Time Complexity:** Training time depends on the number of iterations and the cost of querying two NeRF models per ray. The use of CLIP similarity loss requires rendering views and encoding them, which is computationally expensive compared to pixel-wise MSE losses.
*   **Memory Complexity:** Requires storing weights for both the original NeRF $F_{\theta_O}$ and the generator NeRF $F_{\theta_G}$.
*   **Strengths:**
    *   Can operate on any region of a real-world scene without requiring semantic decomposition or 2D masks [S1].
    *   Generates natural-looking and view-consistent results that blend with the existing scene [S1].
    *   Enables complex text-guided manipulations such as object insertion/replacement and blending [S1].
    *   Does not require learning a new feature space [S1].
*   **Limitations:**
    *   Relies on the quality of the pretrained language-image model (CLIP) for guidance [S1].
    *   May produce low-quality incoherent results if augmentations and priors (transmittance, depth, background) are not used [S1].
    *   Forward-facing scenes with limited viewing intervals may suffer from flat billboard geometry effects without depth regularization [S1].
*   **When to use:** When editing a specific region in an existing NeRF scene using text prompts and a 3D box, especially when avoiding the need for 2D masks or semantic segmentation.
*   **When not to use:** When global scene editing is required, or when high-fidelity geometric consistency is critical without the aid of strong priors.

## Background
Blended-NeRF was introduced by Ori Gordon, Omri Avrahami, and Dani Lischinski from The Hebrew University in 2023 [S1]. It builds on the principles of NeRF [42] and Dream Fields [27], leveraging pretrained language-image models like CLIP [51] for text-guided generation. It addresses the limitations of previous methods like Dream Fields, which synthesize new NeRF objects from scratch, and methods requiring semantic decomposition or 2D masks [S1].

## Sources
[S1] Blended-NeRF: Zero-Shot Object Generation and Blending in Existing Neural Radiance Fields, https://arxiv.org/abs/2306.12760v2, fetched 2026-10-26
