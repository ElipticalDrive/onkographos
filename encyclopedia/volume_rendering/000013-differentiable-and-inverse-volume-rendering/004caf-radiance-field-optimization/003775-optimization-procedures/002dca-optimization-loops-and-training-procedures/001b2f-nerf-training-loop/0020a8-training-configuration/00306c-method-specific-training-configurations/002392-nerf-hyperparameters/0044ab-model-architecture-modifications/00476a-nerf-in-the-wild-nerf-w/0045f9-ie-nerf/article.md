# IE-NeRF

Also called: IE-NeRF Pipeline, Image-dependent NeRF with transient mask inpainting

## Overview
IE-NeRF is a neural radiance field method that separates static and transient components in uncontrolled 'in-the-wild' scenes by using image-dependent feature encodings and a pre-trained inpainting module for supervision, rather than learning a 3D transient field [S1]. It builds on [NeRF in the Wild (NeRF-W)](11995-nerf-in-the-wild-nerf-w.md) by replacing the 3D transient field with an image-dependent 2D mask map to eliminate transient effects [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of uncalibrated or calibrated images $I_i$ with known poses.
*   **Outputs:** A static radiance field (density $\sigma$ and color $c$) and a transient mask $M_{ij}$ for each pixel $j$ in image $i$.

**Data Structures:**
*   **NeRF Network:** 8 fully connected layers with 256 channels each [S1].
    *   **MLP$\theta_1$:** Outputs density $\sigma(t)$ and intermediate feature $z_i(t)$.
    *   **MLP$\theta_2$:** Outputs image-dependent radiance $c_i(t)$ and transient mask embedding $\ell_t$.
    *   **Activation:** ReLU for density $\sigma$; Sigmoid for transient mask $M_{ij}$ (after a 128-channel layer); Sigmoid for static RGB color $c$ (after a 128-channel layer) [S1].
*   **LaMa Input:** A four-channel tensor $x' = \text{stack}(x \odot m, m)$, where $x$ is the masked image and $m$ is the binary mask [S1].

**Numbered Steps:**
1.  **Image Encoding:** Encode each input image $I_i$ into a feature latent vector $\ell_i$ using a CNN-based encoder $E$ [S1].
2.  **Static Feature Extraction:** Process the pose position $r(t) = (x, y, z)$ through $\text{MLP}_{\theta_1}$ with integrated positional encoding $\gamma_x$ to output density $\sigma(t)$ and intermediate feature $z_i(t)$ [S1]:
    $$ [\sigma(t), z_i(t)] = \text{MLP}_{\theta_1}[\gamma_x(r(t))] $$
3.  **Radiance and Transient Embedding:** Feed the directional vector $d$ (with encoding $\gamma_d$), the intermediate feature $z_i(t)$, and the original position $(x,y,z)$ into $\text{MLP}_{\theta_2}$ to predict image-dependent radiance $c_i(t)$ and transient mask embedding $\ell_t$ [S1]:
    $$ c_i(t) = \text{MLP}_{\theta_2}[\gamma_d(d), z_i(t)] $$
    *(Note: The source defines $c_i(t)$ in Eq. 7 but implies $\ell_t$ is also an output of this stage or a parallel branch of $\theta_2$ [S1].)*
4.  **Transient Mask Generation:** Generate a 2D pixel-wise transient mask probability $M_{ij}$ by mapping the 2D pixel location $l_{ij}$ and transient embedding $\ell_t$ through linear layers $E_\phi$ [S1]:
    $$ M_{ij} = E_\phi(l_{ij}, \ell_t) $$
5.  **Volumetric Rendering:** Compute the approximated pixel color $\hat{C}_i(r)$ using the static density $\sigma(t)$ and image-dependent radiance $c_i(t)$ [S1]:
    $$ \hat{C}_i(r) = \sum_{k=1}^{K} T(t_k) \cdot \alpha(\sigma(t_k)\delta_k) \cdot c_i(t_k) $$
    where $T(t_k) = \exp(-\sum_{s=1}^{k-1} \sigma(t_s)\delta_s)$ and $\alpha(\sigma\delta) = 1 - \exp(-\sigma\delta)$ [S1].
6.  **Inpainting Supervision (Image):** Apply the thresholded transient mask $M_{ij}$ to the original image $I_i$ to create a masked image, then feed it into the pre-trained LaMa inpainting network $G_\Delta$ to produce the restored static image $I_{ri}$ [S1].
7.  **Inpainting Supervision (Depth):** Render depth maps $D_{si}$ from the static NeRF and inpaint them using LaMa guided by the transient mask to produce restored depth maps $D_{ri}$ for geometry supervision [S1].
8.  **Loss Calculation:** Compute the total loss $L$ combining static, transient, perceptual, and depth losses [S1]:
    $$ L = \alpha \sum_{ij} L_s + \beta \sum_{ij} L_t + \gamma \sum_i L_{\text{LPIPS}} + \rho \sum_i L_{\text{depth}} $$

**Formulas:**
*   **Transient Loss ($L_t$):** Balances occlusion and reconstruction errors using mask $M_{ij}$ [S1]:
    $$ L_t = (1 - M_{ij}) \| C(r_{ij}) - \hat{C}_s(r_{ij}) \|^2 + \lambda M_{ij} \| C_s(r_{ij}) - \hat{C}_s(r_{ij}) \|^2 $$
    where $C(r_{ij})$ is the original image color, $\hat{C}_s(r_{ij})$ is the rendered static color, and $C_s(r_{ij})$ is the inpainted static color. $\lambda$ balances static/transient components [S1].
*   **Frequency Regularization:** Applies a linearly increasing frequency mask $\omega$ to positional encoding $\gamma_L$ [S1]:
    $$ \gamma'_L(t, T; x) = \gamma_L(x) \odot \omega(t, T, L) $$
    where $\omega^i(t, T, L)$ is defined as:
    $$ \omega^i(t, T, L) = \begin{cases} 1 & \text{if } i \le \lfloor \frac{t \cdot L}{T} \rfloor \\ 0 & \text{if } i > \lfloor \frac{t \cdot L}{T} \rfloor + 6 \end{cases} $$
    *(Note: The source provides a piecewise function with an intermediate range $t \cdot L / T + 3 < i \le t \cdot L / T + 6$ which is not fully specified in the text snippet, but the intent is a linearly increasing mask [S1].)*

**Parameters:**
*   $\lambda$: Parameter adjusting the balance between transient and static components in $L_t$ [S1].
*   $\alpha, \beta, \gamma, \rho$: Weights for static loss, transient loss, perceptual loss, and depth loss respectively [S1].
*   $L$: Hyperparameter determining the maximum encoded frequency in the positional encoding [S1].
*   $r$: Transient mask factor, defined as the ratio of transient pixels to the total image area [S1].

**Pitfalls:**
*   Inpainting modules may produce blurry results and inconsistencies across views; mitigated by using LPIPS loss inside the mask and depth supervision [S1].
*   High-frequency signals cause instability and vulnerability in transient separation; mitigated by starting training without positional encoding and incrementally boosting frequency [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# IE-NeRF Training Step
def ie_nerf_step(images, poses, encoder, mlp_theta1, mlp_theta2, 
                 encoder_phi, laema_model, mipnerf_caster, 
                 alpha, beta, gamma, rho, lambda_trans, 
                 current_iter, max_iter_freq, max_freq_L):
    
    # 1. Image Encoding
    # Encode each image I_i into latent vector l_i
    latent_vectors = []
    for img in images:
        l_i = encoder(img)
        latent_vectors.append(l_i)
        
    static_losses = []
    transient_losses = []
    lpips_losses = []
    depth_losses = []
    
    for i, (img, pose, l_i) in enumerate(zip(images, poses, latent_vectors)):
        # 2. Ray Casting (Mip-NeRF Cone Casting)
        # Cast rays r from pose, handling multiscale via IPE
        rays = mipnerf_caster(pose, img.shape)
        
        # 3. Static Feature Extraction
        # [sigma, z_i] = MLP_theta1[gamma_x(r)]
        # Note: gamma_x is Integrated Positional Encoding
        positions = rays.origin + rays.direction * rays.z_vals
        encoded_pos = integrated_pos_encoding(positions, current_iter, max_iter_freq, max_freq_L)
        sigma, z_i = mlp_theta1(encoded_pos)
        
        # 4. Radiance and Transient Embedding
        # c_i = MLP_theta2[gamma_d(d), z_i]
        # l_t is also output from this stage (parallel or combined)
        directions = rays.direction
        encoded_dir = integrated_pos_encoding(directions, current_iter, max_iter_freq, max_freq_L)
        c_i, l_t = mlp_theta2(encoded_dir, z_i)
        
        # 5. Volumetric Rendering
        # C_hat_i(r) = sum T(t_k) * alpha(sigma(t_k)*delta_k) * c_i(t_k)
        T_vals = compute_transmittance(sigma, rays.delta)
        alpha_vals = 1 - torch.exp(-sigma * rays.delta)
        weights = T_vals * alpha_vals
        C_hat_i = torch.sum(weights.unsqueeze(-1) * c_i, dim=-2)
        
        # Render Depth
        D_s_i = torch.sum(weights * rays.z_vals, dim=-1)
        
        # 6. Transient Mask Generation
        # M_ij = E_phi(l_ij, l_t)
        # l_ij is 2D pixel location
        pixel_coords = get_pixel_coords(img.shape)
        M_ij = encoder_phi(pixel_coords, l_t)
        
        # Threshold mask for inpainting
        M_binary = (M_ij > 0.5).float()
        
        # 7. Inpainting Supervision
        # Masked image x' = stack(img * (1-M), M)
        masked_img = img * (1 - M_binary)
        inpaint_input = torch.stack([masked_img, M_binary], dim=1) # 4-channel tensor
        
        # Restore static image and depth
        I_r_i = laema_model(inpaint_input)
        D_r_i = laema_model_depth(inpaint_input) # Assuming separate depth inpainter or same
        
        # 8. Loss Calculation
        
        # Static Loss: MSE on non-transient pixels
        # L_s = (1 - M_ij) * (C - C_hat_s)^2
        # Note: C_hat_s is C_hat_i (static component)
        static_loss = torch.mean((1 - M_ij) * (img - C_hat_i)**2)
        static_losses.append(static_loss)
        
        # Transient Loss
        # L_t = (1 - M_ij) * (C - C_hat_s)^2 + lambda * M_ij * (C_s - C_hat_s)^2
        # C_s is I_r_i (inpainting result)
        term1 = (1 - M_ij) * (img - C_hat_i)**2
        term2 = lambda_trans * M_ij * (I_r_i - C_hat_i)**2
        trans_loss = torch.mean(term1 + term2)
        transient_losses.append(trans_loss)
        
        # Perceptual Loss (LPIPS) inside mask
        # L_LPIPS computed on masked regions between I_r_i and img? 
        # Source says "supervised by ... repaired pixels' LPIPS inside the mask"
        # Implementation detail: Compute LPIPS on masked regions
        lpips_loss = compute_lpips(img * M_binary, I_r_i * M_binary)
        lpips_losses.append(lpips_loss)
        
        # Depth Loss
        # L_depth = mean(D_s_i - D_r_i)
        depth_loss = torch.mean(torch.abs(D_s_i - D_r_i))
        depth_losses.append(depth_loss)
        
    # Aggregate Losses
    L = (alpha * sum(static_losses) + 
         beta * sum(transient_losses) + 
         gamma * sum(lpips_losses) + 
         rho * sum(depth_losses))
         
    return L
```

## Subtopics

### NeRF in the Wild (NeRF-W)
The parent method that uses a 3D transient field. IE-NeRF differs by using a 2D mask. [Link](11995-nerf-in-the-wild-nerf-w.md)

### Mip-NeRF
Provides the cone-casting and Integrated Positional Encoding (IPE) used by IE-NeRF for multiscale representation. [Link](52-mip-nerf.md)

### LaMa
The pre-trained inpainting network used for supervision. [Link](53-lama.md)

## Operating characteristics
*   **Time and Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Evaluated on Phototourism dataset scenes: Brandenburg Gate, Sacre Coeur, Trevi Fountain, Taj Mahal [S1].
*   **Pre-filtering:** Discards images with NIMA score < 3 and images where transient objects occupy > 80% of the frame (using DeepLab v3 on Ade20k) [S1].
*   **Resolution:** Training uses 2x downsampled images, consistent with NeRF-W and HA-NeRF [S1].
*   **Strengths:** Eliminates transient effects without learning a 3D transient field; handles view-dependent appearance via image-dependent encodings [S1].
*   **Limitations:** Inpainting may cause blurriness; high-frequency instability during early training [S1].

## Background
IE-NeRF was proposed to address the challenges of reconstructing neural radiance fields from uncontrolled 'in-the-wild' photo collections, where lighting variations and transient objects (people, vehicles) degrade reconstruction quality [S1]. It extends the NeRF-W approach by introducing a 2D transient mask and leveraging pre-trained inpainting for supervision [S1].

## Sources
[S1] IE-NeRF: Exploring transient mask inpainting to enhance neural radiance fields in the wild, https://doi.org/10.1016/j.neucom.2024.129112, fetched 2026-10-27
