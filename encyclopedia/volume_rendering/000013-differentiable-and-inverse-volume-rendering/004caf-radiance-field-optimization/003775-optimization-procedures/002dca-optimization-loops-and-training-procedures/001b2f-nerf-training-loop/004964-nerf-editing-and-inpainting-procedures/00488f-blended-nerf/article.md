# Blended-NeRF

Also called: Zero-Shot Object Generation and Blending in Existing Neural Radiance Fields

## Overview
Blended-NeRF is a framework for editing a specific region of interest (ROI) in an existing Neural Radiance Field (NeRF) scene based on text prompts and a user-defined 3D ROI box [S1]. It solves the problem of local 3D editing without requiring semantic masks or learning a new feature space, by initializing a generator NeRF with the original scene's weights and blending the synthesized content with the original scene using volumetric blending [S1]. Unlike methods that synthesize objects from scratch or require 2D masks for localization, Blended-NeRF operates on any region of a real-world scene, preserving the unedited parts while generating view-consistent, natural-looking edits guided by a language-image model [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** An existing NeRF model $F_{\theta_O}$ representing scene $x_o$, a 3D Region of Interest (ROI) box $B$ localized within the scene, and a user-provided text prompt $y$ [S1].
*   **Output:** An edited NeRF model $F_{\theta_E}$ (implicitly defined by the blending of $F_{\theta_O}$ and the trained generator $F_{\theta_G}$) that renders the edited scene $x_e$ [S1].

**Data Structures:**
*   **NeRF MLP ($F_\theta$):** Maps 3D position $\mathbf{x}$ and viewing direction $\mathbf{v}$ to density $\sigma$ and color $\mathbf{c}$. Inputs are embedded using high-frequency sinusoidal positional encoding $\gamma(\mathbf{x}) = [\cos(2^l \mathbf{x}), \sin(2^l \mathbf{x})]_{l=0}^{L-1}$ [S1].
*   **ROI Box ($B$):** A 3D axis-aligned or arbitrary box defined by its center and dimensions, used to localize the edit region [S1].
*   **Language-Image Model:** Pretrained model (e.g., CLIP) with image encoder $E_{img}$ and text encoder $E_{txt}$ [S1].

**Parameters:**
*   $\alpha$: Hyperparameter for distance smoothing blending strength. $\alpha=0$ weights entirely to the edited scene; higher values weight closer points to the original scene. Typical values: 0, 0.5, 2, 4, 10 [S1].
*   $\lambda_T, \lambda_D$: Weights for transmittance and depth losses [S1].
*   $\tau$: Max transmittance threshold for $L_T$ [S1].
*   $\rho$: Max variance allowed for depth loss $L_D$ [S1].
*   $\epsilon$: Small constant for numerical stability in blending [S1].
*   $\gamma$: Camera FOV [S1].
*   $e_{max}$: Maximum dimension of the ROI box [S1].

**Numbered Steps:**

1.  **Initialization:** Initialize a NeRF generator model $F_{\theta_G}$ with the weights of the original scene model $F_{\theta_O}$ [S1].
2.  **ROI Localization:** Localize the 3D box $B$ inside the scene using a GUI that renders the original NeRF and uses the output depth map for 3D understanding [S1].
3.  **Training Loop:** Optimize $F_{\theta_G}$ by minimizing the total objective $L_{total}$:
    a. **Pose Sampling:** Sample a random camera pose around the scene with azimuth/elevation angles $(\theta, \phi)$. Set initial distance $d = \frac{e_{max}}{2 \tan(\gamma/2)}$ and randomly sample radius $r$ around $d$ [S1].
    b. **Ray Casting & Sampling:** Cast rays $r$ through the sampled pose. Sample points $\mathbf{x}_i$ along the ray.
    c. **Density Masking (Training):** For points $\mathbf{x}_i$ outside $B$, set density $\sigma_G(\mathbf{x}_i) = 0$. For points inside $B$, query $F_{\theta_G}$ for $\sigma_G(\mathbf{x}_i)$ and $\mathbf{c}_G(\mathbf{x}_i)$ [S1].
    d. **Rendering (Training):** Render the generated content inside the ROI $I_{ROI}$ using only points $\mathbf{x}_i \in B$. The rendered color $C(r)$ is:
       $$ C(r) = \frac{\sum_{\mathbf{x}_i \in B} T_i (1 - e^{-\sigma_i \delta_i}) \mathbf{c}_i}{\sum_{\mathbf{x}_i \in B} T_i (1 - e^{-\sigma_i \delta_i})} \quad \text{if } \exists \mathbf{x}_i \in r \text{ s.t. } \mathbf{x}_i \in B, \text{ else } 0 $$
       where $T_i = \exp(-\sum_{j < i} \sigma_j \delta_i)$ and $\delta_i = t_{i+1} - t_i$ [S1].
    e. **Loss Computation:**
       i. **Similarity Loss:** $L_{sim} = -E_{img}(I_{ROI})^T E_{txt}(y)$ [S1].
       ii. **Transmittance Loss:** $L_T = -\min(\tau, \text{mean}(T(P)))$, where $T(P)$ is the average transmittance of the rendered view [S1].
       iii. **Depth Loss:** $L_D = -\min(\rho, \sigma^2(D(P)))$, where $\sigma^2(D(P))$ is the variance of the disparity map [S1].
       iv. **Total Loss:** $L_{total} = L_{sim} + \lambda_T L_T + \lambda_D L_D$. Anneal $L_T$ and $L_D$ during training to prevent transparent/amorphous scenes [S1].
    f. **Augmentations:** Use random backgrounds (Gaussian noise, checkerboard, random Fourier textures) to prevent scene diffusion [S1]. Use directional dependent prompts by concatenating view-dependent text to the original caption [S1].
    g. **Optimization:** Update $\theta_G$ using an optimizer (e.g., Adam) [S1].
4.  **Blending (Inference):** After training, render the edited scene by blending $F_{\theta_O}$ and $F_{\theta_G}$ along each view ray:
    a. **Query:** For each ray, query both $F_{\theta_O}$ and $F_{\theta_G}$ for the same set of points. Points inside $B$ are primarily influenced by $F_{\theta_G}$, points outside by $F_{\theta_O}$ [S1].
    b. **Mode Selection:**
       i. **Object Insertion/Replacement:** Consider only $F_{\theta_G}$ radiance field inside ROI during training. Blend using distance smoothing (Eq. 5) [S1].
       ii. **Object Blending:** Query both models inside the box. Compute alpha values: $\alpha_O(\mathbf{x}_i) = 1 - \exp(\phi(\sigma_O(\mathbf{x}_i)) \cdot \delta_i)$, $\alpha_G(\mathbf{x}_i) = 1 - \exp(\phi(\sigma_G(\mathbf{x}_i)) \cdot \delta_i)$, where $\phi$ is a non-negative activation [S1].
    c. **Density Blending (Object Blending Mode):** Choose one of two options for $\sigma(\mathbf{x}_i)$:
       i. Sum in activation: $\sigma(\mathbf{x}_i) = \phi(\sigma_O(\mathbf{x}_i) + \sigma_G(\mathbf{x}_i))$ (allows $F_{\theta_G}$ to remove original densities) [S1].
       ii. Sum out activation: $\sigma(\mathbf{x}_i) = \phi(\sigma_O(\mathbf{x}_i)) + \phi(\sigma_G(\mathbf{x}_i))$ (allows only addition of new densities) [S1].
    d. **Color Blending (Object Blending Mode):**
       $$ \mathbf{c}(\mathbf{x}_i) = S\left(\frac{\mathbf{c}_O(\mathbf{x}_i) \cdot \alpha_O(\mathbf{x}_i) + \mathbf{c}_G(\mathbf{x}_i) \cdot \alpha_G(\mathbf{x}_i)}{\epsilon + \alpha_O(\mathbf{x}_i) + \alpha_G(\mathbf{x}_i)}\right) $$
       where $S$ is sigmoid [S1].
    e. **Distance Smoothing (Insertion/Replacement Mode):** For points inside $B$, blend densities and colors based on distance $d(\mathbf{x})$ from ROI center and box diagonal $diag$:
       $$ f(\mathbf{x}) = 1 - \exp\left(-\alpha \frac{d(\mathbf{x})}{diag}\right) $$
       $$ \sigma_{blend}(\mathbf{x}) = f(\mathbf{x}) \cdot \sigma_O(\mathbf{x}) + (1 - f(\mathbf{x})) \cdot \sigma_G(\mathbf{x}) $$
       $$ \mathbf{c}_{blend}(\mathbf{x}) = f(\mathbf{x}) \cdot \mathbf{c}_O(\mathbf{x}) + (1 - f(\mathbf{x})) \cdot \mathbf{c}_G(\mathbf{x}) $$
       [S1].
    f. **Final Rendering:** Blend the raw densities and RGB values inside and outside the ROI along each ray using standard volume rendering (Eq. 1) to get the final view $x_e$ [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def sin_pos_encoding(x, L=10):
    """High frequency sinusoidal positional encoding."""
    # x shape: (N, 3)
    # Returns shape: (N, 3 * 2 * L)
    freq_bands = 2.0 ** np.arange(L)
    encoded = []
    for l in range(L):
        freq = freq_bands[l]
        encoded.append(np.cos(freq * x))
        encoded.append(np.sin(freq * x))
    return np.concatenate(encoded, axis=-1)

def volume_rendering(rays, model, roi_box=None, is_training=False, mode='insertion'):
    """
    Render a batch of rays using a NeRF model.
    rays: (N, 2) [origin, direction]
    model: function mapping (x, v) -> (sigma, c)
    roi_box: 3D box defining ROI
    is_training: if True, mask points outside ROI
    mode: 'insertion' or 'blending'
    """
    # Sample points along rays
    t_vals = np.linspace(0.0, 1.0, 128) # Example N_samples
    t = t_vals[:-1] + 0.5 * (t_vals[1:] - t_vals[:-1])
    x = rays[:, :3] + t[:, None] * rays[:, 3:6]
    
    # Get directions (constant along ray)
    v = rays[:, 3:6]
    
    # Encode positions
    x_enc = sin_pos_encoding(x)
    
    # Query model
    sigma, c = model(x_enc, v)
    
    if is_training and roi_box is not None:
        # Mask points outside ROI
        inside_roi = is_inside_box(x, roi_box)
        sigma = sigma * inside_roi
        c = c * inside_roi # Color doesn't matter if sigma is 0, but for consistency
        
    # Compute deltas
    deltas = t_vals[1:] - t_vals[:-1]
    
    # Compute transmittance and alpha
    # sigma is assumed to be raw output; apply activation if needed by model
    # Assuming model outputs raw sigma, we use exp(-sigma * delta)
    alpha = 1.0 - np.exp(-sigma * deltas)
    T = np.cumprod(np.concatenate([np.ones((x.shape[0], 1)), 1.0 - alpha[:, :-1]], axis=1), axis=1)
    
    # Weighted sum
    weights = T * alpha
    color = np.sum(weights[:, :, None] * c, axis=1)
    
    # For training ROI rendering, normalize by alpha sum if needed (Eq 4)
    if is_training and roi_box is not None:
        alpha_sum = np.sum(weights, axis=1, keepdims=True)
        # Avoid division by zero
        alpha_sum = np.maximum(alpha_sum, 1e-8)
        color = color / alpha_sum
        
    return color, sigma, alpha, T

def blend_models(rays, model_O, model_G, roi_box, alpha_smooth=2.0):
    """
    Blend original and generator models for inference.
    """
    # Sample points
    t_vals = np.linspace(0.0, 1.0, 128)
    t = t_vals[:-1] + 0.5 * (t_vals[1:] - t_vals[:-1])
    x = rays[:, :3] + t[:, None] * rays[:, 3:6]
    v = rays[:, 3:6]
    x_enc = sin_pos_encoding(x)
    
    # Query both models
    sigma_O, c_O = model_O(x_enc, v)
    sigma_G, c_G = model_G(x_enc, v)
    
    inside_roi = is_inside_box(x, roi_box)
    
    # Distance smoothing factor
    if roi_box is not None:
        center = get_box_center(roi_box)
        diag = get_box_diagonal(roi_box)
        dist = np.linalg.norm(x - center, axis=1)
        f = 1.0 - np.exp(-alpha_smooth * dist / diag)
    else:
        f = np.zeros_like(sigma_O)
        
    # Apply blending based on mode
    # Assuming 'insertion' mode for this pseudocode as it's the primary blending described
    sigma_blend = f * sigma_O + (1.0 - f) * sigma_G
    c_blend = f * c_O + (1.0 - f) * c_G
    
    # Outside ROI, use original
    sigma_blend = sigma_blend * inside_roi + sigma_O * (1.0 - inside_roi)
    c_blend = c_blend * inside_roi + c_O * (1.0 - inside_roi)
    
    # Standard Volume Rendering with blended values
    deltas = t_vals[1:] - t_vals[:-1]
    alpha = 1.0 - np.exp(-sigma_blend * deltas)
    T = np.cumprod(np.concatenate([np.ones((x.shape[0], 1)), 1.0 - alpha[:, :-1]], axis=1), axis=1)
    weights = T * alpha
    color = np.sum(weights[:, :, None] * c_blend, axis=1)
    
    return color

def is_inside_box(x, box):
    """Check if points are inside the 3D box."""
    # x: (N, 3), box: (min_x, min_y, min_z, max_x, max_y, max_z)
    min_pt = box[:3]
    max_pt = box[3:]
    return np.all((x >= min_pt) & (x <= max_pt), axis=1)

def get_box_center(box):
    min_pt = box[:3]
    max_pt = box[3:]
    return (min_pt + max_pt) / 2.0

def get_box_diagonal(box):
    min_pt = box[:3]
    max_pt = box[3:]
    return np.linalg.norm(max_pt - min_pt)

def train_blended_nerf(model_O, roi_box, text_prompt, num_iterations):
    """
    Train the generator model F_theta_G.
    """
    # Initialize G with O's weights
    model_G = copy_weights(model_O)
    
    # Load CLIP encoders
    E_img, E_txt = load_clip()
    
    for i in range(num_iterations):
        # 1. Pose Sampling
        theta, phi = sample_random_pose()
        e_max = get_box_max_dimension(roi_box)
        gamma = camera_fov
        d = e_max / (2 * np.tan(gamma / 2))
        r = np.random.uniform(d * 0.8, d * 1.2) # Random radius around d
        origin = sample_point_on_sphere(r, theta, phi)
        direction = -origin # Looking at center
        rays = np.stack([origin, direction], axis=1)
        
        # 2. Render ROI
        color_roi, sigma, alpha, T = volume_rendering(rays, model_G, roi_box, is_training=True)
        
        # 3. Compute Losses
        # Similarity Loss
        img_feat = E_img(color_roi)
        txt_feat = E_txt(text_prompt)
        L_sim = -np.dot(img_feat, txt_feat)
        
        # Transmittance Loss
        mean_T = np.mean(T)
        tau = 0.9 # Example max transmittance
        L_T = -np.minimum(tau, mean_T)
        
        # Depth Loss (Simplified: assume disparity variance calculation)
        # In practice, render depth map and compute variance
        disparity_map = render_disparity(rays, sigma)
        var_D = np.var(disparity_map)
        rho = 1.0 # Example max variance
        L_D = -np.minimum(rho, var_D)
        
        # Total Loss
        lambda_T = 0.1
        lambda_D = 0.1
        L_total = L_sim + lambda_T * L_T + lambda_D * L_D
        
        # 4. Update Model_G
        update_weights(model_G, L_total)
        
        # 5. Augmentations (Background, Directional Prompts)
        # Apply random background to rays if needed
        # Modify text prompt based on view direction
```

## Operating characteristics
*   **Time Complexity:** Training time depends on the number of iterations and the cost of querying the MLP and CLIP encoder per ray. It is comparable to other NeRF optimization methods but requires rendering only the ROI for the similarity loss, which can be faster than full-scene optimization if the ROI is small.
*   **Memory Complexity:** Requires storing the weights of both $F_{\theta_O}$ and $F_{\theta_G}$. Since $F_{\theta_G}$ is initialized with $F_{\theta_O}$, memory usage is roughly double that of a single NeRF.
*   **Strengths:** Does not require 2D masks or semantic decomposition; works on any region; preserves unedited parts of the scene; generates view-consistent results.
*   **Limitations:** Quality depends on the text prompt and CLIP alignment; may produce artifacts at ROI boundaries if blending parameters are not tuned; limited to editing within the specified 3D box.
*   **When to use:** When you want to insert, replace, or blend objects in a specific 3D region of an existing NeRF scene using text guidance, without access to segmentation masks.
*   **When not to use:** When global scene editing is required, or when precise semantic control outside of text prompts is needed.

## Background
Blended-NeRF was introduced by Ori Gordon, Omri Avrahami, and Dani Lischinski from The Hebrew University in 2023 [S1]. It builds on the principles of NeRF [42] and Dream Fields [27], adapting them for local editing. Unlike Dream Fields which synthesizes new NeRF objects from scratch, Blended-NeRF edits existing scenes by initializing the generator with existing weights and blending results [S1]. It addresses limitations of prior editing methods like Kobayashi [29] and Benaim et al. [8], which required semantic decomposition or 2D masks [S1].

## Sources
[S1] Blended-NeRF: Zero-Shot Object Generation and Blending in Existing Neural Radiance Fields, https://arxiv.org/abs/2306.12760v2, fetched 2026-10-26
