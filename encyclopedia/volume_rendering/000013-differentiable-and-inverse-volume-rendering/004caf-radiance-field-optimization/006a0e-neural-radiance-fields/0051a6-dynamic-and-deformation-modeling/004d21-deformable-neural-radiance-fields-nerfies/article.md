# Deformable Neural Radiance Fields (Nerfies)

Also called: Deformable NeRF, Nerfies

## Overview
Deformable Neural Radiance Fields (Nerfies) reconstructs non-rigidly deforming scenes by decomposing the scene into a static canonical template volume (a standard NeRF) and a per-observation continuous deformation field that maps observation-space coordinates to canonical-space coordinates [S2]. Unlike methods that condition deformation on time or define it on discrete meshes/voxels, Nerfies conditions the deformation on a learned per-frame latent code and models the deformation as a continuous function using an MLP [S2]. It distinguishes itself from siblings by introducing elastic regularization to resolve ambiguities between motion and scaling, and a coarse-to-fine optimization strategy to avoid local minima [S2].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of $n$ images $\{I_i\}_{i=1}^n$ of a non-rigidly deforming scene, along with their camera poses [S2].
*   **Outputs:** A canonical NeRF $F$ (representing the template), a deformation field $T$ (parameterized by an MLP), and per-frame latent codes $\omega_i$ (deformation) and $\psi_i$ (appearance) [S2]. The system renders novel views of the deforming subject [S2].

**Data Structures:**
1.  **Canonical NeRF $F$:** An MLP mapping $(x', d, \psi_i) \to (\sigma, c)$, where $x'$ is a canonical 3D coordinate, $d$ is a viewing direction, $\sigma$ is density, and $c$ is color [S2].
2.  **Deformation MLP $W$:** An MLP mapping $(x, \omega_i) \to (r, v) \in \mathbb{R}^6$, where $x$ is an observation-space 3D coordinate and $\omega_i$ is the deformation latent code [S2]. The output $(r, v)$ parameterizes an SE(3) transformation [S2].
3.  **Latent Codes:**
    *   $\omega_i \in \mathbb{R}^8$: Per-frame learned latent deformation code encoding the scene state [S2].
    *   $\psi_i \in \mathbb{R}^8$: Per-frame learned latent appearance code modulating color output to handle exposure/white balance variations [S2].
4.  **Positional Encoding:** Input coordinates $x$ are encoded using $\gamma_\alpha(x)$, which applies frequency-dependent weights $w_k(\alpha)$ to sinusoidal basis functions [S2].

**Parameters:**
*   **Network Architecture:** Deformation network depth 6, hidden size 128, skip connection at the 4th layer [S2].
*   **Samples:** 256 coarse and 256 fine ray samples per ray for full HD (1920×1080) [S2].
*   **Annealing:** 6 frequency bands; $\alpha$ annealed from 0 to 6 over 80K iterations [S2].
*   **Loss Weights:** $\lambda = \mu = 10^{-3}$ for elastic and background regularization [S2].
*   **Robust Loss Hyperparameter:** $c = 0.03$ for the Geman-McClure function [S2].

**Step-by-Step Method:**

1.  **Initialization:**
    *   Initialize the weights of the last layer of the deformation MLP $W$ from a uniform distribution $U(-10^{-5}, 10^{-5})$ to ensure the deformation starts near the identity, since the exponential map $e^S$ is the identity when $S=0$ [S2].
    *   Initialize latent codes $\omega_i$ and $\psi_i$ via embedding layers [S2].

2.  **Ray Casting and Sampling:**
    *   For a target view, cast rays $\mathbf{r}$ through the camera [S2].
    *   Sample $N$ points $x_i$ along each ray in the observation space [S2].

3.  **Deformation (Observation to Canonical):**
    *   For each sample point $x_i$ and frame $i$, retrieve the deformation latent code $\omega_i$ [S2].
    *   Encode $x_i$ using the positional encoding $\gamma_\alpha(x_i)$, where $\alpha$ is the current annealing parameter [S2].
    *   Pass $(\gamma_\alpha(x_i), \omega_i)$ through the deformation MLP $W$ to obtain $(r, v) \in \mathbb{R}^6$ [S2].
    *   Compute the SE(3) transformation $e^S$ from the screw axis $S = (r; v)$:
        *   Compute rotation angle $\theta = \|r\|$ and axis $\hat{r} = r/\|r\|$ [S2].
        *   Compute rotation matrix $e^r$ using Rodrigues' formula:
            $$ e^r = I + \frac{\sin \theta}{\theta}[r]_\times + \frac{1 - \cos \theta}{\theta^2}[r]_\times^2 $$
            where $[r]_\times$ is the cross-product matrix of $r$ [S2].
        *   Compute translation $p$ using the matrix $G$:
            $$ G = I + \frac{1 - \cos \theta}{\theta^2}[r]_\times + \frac{\theta - \sin \theta}{\theta^3}[r]_\times^2 $$
            $$ p = Gv $$
        *   Transform the point to canonical space: $x'_i = e^r x_i + p$ [S2].

4.  **Radiance Field Evaluation:**
    *   Query the canonical NeRF $F$ with the transformed point $x'_i$, viewing direction $d$, and appearance code $\psi_i$ to obtain density $\sigma_i$ and color $c_i$ [S2].

5.  **Volume Rendering:**
    *   Compute the rendered color $\hat{C}(\mathbf{r})$ using the standard emission-absorption model:
        $$ \hat{C}(\mathbf{r}) = \sum_{i=1}^N T_i (1 - \exp(-\sigma_i \delta_i)) \mathbf{c}_i $$
        where $T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right)$ is the transmittance and $\delta_i$ is the step size [S2].

6.  **Loss Computation and Regularization:**
    *   **Photometric Loss:** Compute MSE between $\hat{C}(\mathbf{r})$ and the ground truth image color [S2].
    *   **Elastic Regularization:** Compute the Jacobian $J_T(x)$ of the deformation field with respect to $x$ using automatic differentiation. Compute the singular values $\Sigma$ of $J_T(x)$. Apply the robust elastic loss:
        $$ L_{\text{elastic-r}}(x) = \rho(\|\log \Sigma\|_F, c) $$
        where $\rho(x, c) = \frac{2(x/c)^2}{(x/c)^2 + 4}$ is the Geman-McClure robust error function with $c=0.03$ [S2].
        *   Weight this loss by the sample's contribution to the rendered view $w_i$ to allow free deformation in empty space [S2].
    *   **Background Regularization (Optional):** If static 3D points $\{x_k\}_{k=1}^K$ are known, penalize their movement:
        $$ L_{\text{bg}} = \frac{1}{K} \sum_{k=1}^K \|T(x_k) - x_k\|^2 $$
        [S2].
    *   **Total Loss:** $L_{\text{total}} = L_{\text{rgb}} + \lambda L_{\text{elastic-r}} + \mu L_{\text{bg}}$ [S2].

7.  **Optimization:**
    *   Optimize the parameters of $F$, $W$, $\omega_i$, and $\psi_i$ using Adam [S2].
    *   Anneal the positional encoding bandwidth parameter $\alpha(t)$ linearly: $\alpha(t) = m \cdot \frac{t}{N}$, where $t$ is the current iteration, $N$ is the total iterations for $\alpha$ to reach $m$, and $m$ is the max number of frequency bands (e.g., 6) [S2]. This implements coarse-to-fine optimization [S2].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn

def positional_encoding(x, alpha, num_bands):
    """
    Computes frequency-weighted positional encoding.
    x: (N, 3) observation coordinates
    alpha: scalar annealing parameter
    num_bands: int, number of frequency bands
    Returns: (N, D) encoded features
    """
    # Generate base frequencies
    freq_bands = torch.arange(num_bands, device=x.device).float()
    # Hann window weights sliding across bands
    w = 0.5 * (1 - torch.cos(torch.pi * torch.clamp(alpha - freq_bands, 0, 1)))
    # Scale frequencies by alpha
    scales = (2 ** freq_bands) * alpha
    # Expand dimensions for broadcasting
    x_exp = x.unsqueeze(-1)  # (N, 3, 1)
    scales_exp = scales.unsqueeze(0).unsqueeze(0)  # (1, 1, num_bands)
    # Compute sin/cos components
    sin_comp = torch.sin(scales_exp * torch.pi * x_exp)
    cos_comp = torch.cos(scales_exp * torch.pi * x_exp)
    # Apply weights and concatenate
    sin_w = w.unsqueeze(0).unsqueeze(0) * sin_comp
    cos_w = w.unsqueeze(0).unsqueeze(0) * cos_comp
    return torch.cat([x, sin_w.flatten(1), cos_w.flatten(1)], dim=-1)

def se3_transform(r, v):
    """
    Computes SE(3) transform from screw axis parameters.
    r: (N, 3) rotation vector
    v: (N, 3) translation vector
    Returns: R (N, 3, 3), p (N, 3)
    """
    theta = torch.norm(r, dim=-1, keepdim=True)  # (N, 1)
    r_hat = r / (theta + 1e-8)  # (N, 3)
    sin_theta = torch.sin(theta)
    cos_theta = torch.cos(theta)
    
    # Cross-product matrix [r]_x
    def cross_matrix(v):
        x, y, z = v[:, 0], v[:, 1], v[:, 2]
        zeros = torch.zeros_like(x)
        return torch.stack([
            torch.stack([zeros, -z, y], dim=-1),
            torch.stack([z, zeros, -x], dim=-1),
            torch.stack([-y, x, zeros], dim=-1)
        ], dim=-2)
    
    r_hat_x = cross_matrix(r_hat)
    
    # Rotation matrix via Rodrigues' formula
    I = torch.eye(3, device=r.device).unsqueeze(0).expand(r.size(0), -1, -1)
    R = I + (sin_theta / (theta + 1e-8)) * r_hat_x + \
        ((1 - cos_theta) / (theta**2 + 1e-8)) * torch.bmm(r_hat_x, r_hat_x)
    
    # Translation matrix G
    G = I + ((1 - cos_theta) / (theta**2 + 1e-8)) * r_hat_x + \
        ((theta - sin_theta) / (theta**3 + 1e-8)) * torch.bmm(r_hat_x, r_hat_x)
    
    p = torch.bmm(G, v.unsqueeze(-1)).squeeze(-1)
    return R, p

def nerfies_forward(x_obs, d, omega, psi, deformation_mlp, canonical_nerf, pos_encoder, alpha):
    """
    Forward pass for a batch of points along a ray.
    """
    # 1. Deformation
    x_encoded = pos_encoder(x_obs, alpha, num_bands=6)
    r_v = deformation_mlp(torch.cat([x_encoded, omega], dim=-1))
    r = r_v[:, :3]
    v = r_v[:, 3:]
    
    R, p = se3_transform(r, v)
    x_canonical = torch.bmm(R, x_obs.unsqueeze(-1)).squeeze(-1) + p
    
    # 2. Radiance Field
    sigma, color = canonical_nerf(x_canonical, d, psi)
    return sigma, color

def elastic_regularization(x_obs, omega, deformation_mlp, pos_encoder, alpha, w_i):
    """
    Computes weighted elastic regularization loss.
    """
    x_obs.requires_grad_(True)
    x_encoded = pos_encoder(x_obs, alpha, num_bands=6)
    r_v = deformation_mlp(torch.cat([x_encoded, omega], dim=-1))
    r = r_v[:, :3]
    v = r_v[:, 3:]
    
    R, p = se3_transform(r, v)
    x_prime = torch.bmm(R, x_obs.unsqueeze(-1)).squeeze(-1) + p
    
    # Jacobian J_T(x) = dx'/dx
    jacobian = torch.autograd.grad(x_prime.sum(), x_obs, create_graph=True)[0]
    
    # SVD and log singular values
    U, S, Vh = torch.linalg.svd(jacobian)
    log_S = torch.log(S + 1e-8)
    
    # Frobenius norm of log(S)
    elastic_norm = torch.norm(log_S, dim=-1) ** 2
    
    # Geman-McClure robust loss
    c = 0.03
    robust_loss = 2 * (elastic_norm / c)**2 / ((elastic_norm / c)**2 + 4)
    
    # Weight by ray contribution
    return (robust_loss * w_i).mean()

def volume_render(sigma, color, transmittance_init=1.0):
    """
    Standard volume rendering integration.
    """
    N = sigma.size(0)
    delta = 0.01  # Step size (assumed uniform for pseudocode)
    T = torch.ones(N + 1, device=sigma.device)
    T[1:] = torch.cumprod(torch.exp(-sigma * delta), dim=0)
    
    weights = T[:-1] * (1 - torch.exp(-sigma * delta))
    rendered_color = torch.sum(weights.unsqueeze(-1) * color, dim=0)
    return rendered_color, T
```

## Subtopics

### [Coarse-to-Fine Optimization](13448-coarse-to-fine-optimization.md)
Coarse-to-Fine Optimization is a training strategy for deformable neural radiance fields that stabilizes convergence by gradually increasing the model's capacity to represent high-frequency details [G]. It addresses the difficulty of simultaneously optimizing complex geometry, texture, and non-rigid deformation from scratch by annealing the frequency components of the positional encoding used by the deformation network. It starts with low-frequency bands to learn smooth, large-scale motions, then introduces higher frequencies to capture fine details, preventing local minima.

### [Template-Free NeRF (TFS-NeRF)](16781-template-free-nerf-tfs-nerf.md)
TFS-NeRF is a template-free 3D semantic NeRF method for reconstructing dynamic scenes from sparse or single-view RGB videos, specifically handling interactions between two or more entities (rigid, non-rigid, or deformable). It distinguishes itself from Nerfies by explicitly disentangling the motions of interacting entities and from LBS-based approaches like TAVA by using an Invertible Neural Network (INN) to predict Linear Blend Skinning (LBS) weights directly, thereby bypassing computationally expensive root-finding optimizations. This enables time-efficient training and semantically separable geometry reconstruction without relying on pre-defined template models (e.g., SMPL) or latent codes.

## Operating characteristics
*   **Time Complexity:** Training is computationally intensive. Full HD models require training on 8 V100 GPUs for a week, while half-resolution models take 16 hours [S2]. The deformation field evaluation adds overhead to each ray sample compared to static NeRF [S2].
*   **Memory Complexity:** High memory usage due to storing latent codes for each frame and the large MLPs for both the canonical NeRF and deformation field [S2]. The use of automatic differentiation for elastic regularization also requires storing intermediate states for the Jacobian computation [S2].
*   **Performance:** Capable of photorealistic rendering of non-rigidly deforming scenes (e.g., human faces, bodies) from casually captured mobile phone videos [S2].
*   **Strengths:** Continuous deformation field allows for smooth, high-quality reconstructions without discretization artifacts [S2]. Elastic regularization resolves ambiguities between motion and scaling [S2]. Coarse-to-fine optimization improves robustness [S2].
*   **Limitations:** Prone to local minima without coarse-to-fine optimization [S2]. Requires careful tuning of regularization weights [S2]. Performance degrades if the scene has significant non-rigid deformations that violate the "as-rigid-as-possible" prior (e.g., extreme stretching) [S2].
*   **When to use:** When reconstructing non-rigidly deforming scenes (like humans) from casual video, especially when a template is not available or when continuous deformation is preferred over discrete mesh-based methods [S2].
*   **When not to use:** For static scenes (use standard NeRF), or for scenes with severe non-rigid deformations that break the local rigidity assumption (e.g., fluid-like motions) [S2].

## Background
Deformable Neural Radiance Fields (Nerfies) was introduced by Park et al. in 2021 [S2]. It builds on the original NeRF paper by adding a continuous volumetric deformation field conditioned on per-frame latent codes [S2]. The method was motivated by the challenge of capturing people with hand-held cameras, where non-rigidity and camera motion make reconstruction difficult [S2]. It draws inspiration from geometry processing and physical simulation, specifically elastic energy formulations, to regularize the deformation field [S2]. The method was evaluated on data captured using a rig of two synchronized mobile phones [S2].

## Sources
[S1] Advances in Neural Rendering, https://doi.org/10.48550/arxiv.2111.05849, fetched 2026-10-26 (unofficial)
[S2] Nerfies: Deformable Neural Radiance Fields, https://doi.org/10.1109/iccv48922.2021.00581, fetched 2026-10-26
