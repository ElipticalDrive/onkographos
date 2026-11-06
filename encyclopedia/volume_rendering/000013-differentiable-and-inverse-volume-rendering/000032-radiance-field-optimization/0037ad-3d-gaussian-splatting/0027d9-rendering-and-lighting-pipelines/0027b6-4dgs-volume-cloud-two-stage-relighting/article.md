# 4DGS-Volume-Cloud Two-Stage Relighting

Also called: Frozen Geometry + Environment Network, Stage 1 + Stage 2

## Overview
A two-stage rendering pipeline for physically parameterized volume clouds using 3D Gaussians, designed for real-time rendering and dynamic relighting with arbitrary sun directions [S1, unofficial]. It separates the optimization of static cloud geometry and physical scattering properties (Stage 1) from the learning of global environment lighting (Stage 2), allowing the cloud structure to remain frozen while the environment network adapts to new illumination conditions [S1, unofficial]. Unlike sibling methods that may bake lighting into the primitives or use differentiable volume rendering for both geometry and lighting simultaneously, this method explicitly decouples the geometric occlusion and physical parameters from the environmental inscattering, enabling efficient arbitrary-direction relighting without re-optimizing the cloud itself [S1, unofficial].

## How it works
The method operates in two distinct phases. Stage 1 optimizes the 3D Gaussian representation of the cloud under a single, fixed sun direction with a black background to learn stable physical parameters. Stage 2 freezes these parameters and trains a global environment network to model sky atmospheric scattering, enabling relighting for any sun direction [S1, unofficial].

### Inputs and Outputs
**Inputs:**
*   **Stage 1:** A dataset of images captured under pure sun illumination with a black background (UE SkyAtmosphere brightness factor = 0) [S1, unofficial]. Camera poses and sun directions are provided.
*   **Stage 2:** The trained Stage 1 model (frozen) and a dataset with the same camera poses and sun directions but with the sky atmosphere enabled (providing ambient sky light) [S1, unofficial].

**Outputs:**
*   **Stage 1:** A set of 3D Gaussians with optimized physical parameters (position, scale, rotation, extinction, albedo, scattering phase function, and multiple scattering weights) [S1, unofficial].
*   **Stage 2:** A global environment network (MLP) and precomputed sky visibility transfer vectors, saved as sidecar files alongside the Gaussian point cloud [S1, unofficial].

### Data Structures
The 3D Gaussians store physical parameters instead of standard SH colors and opacity [S1, unofficial]:

*   **Position:** 3 floats ($x, y, z$).
*   **Scale:** 3 floats (log-scale or linear, applied to covariance).
*   **Rotation:** 4 floats (quaternion $w, x, y, z$).
*   **Extinction Coefficient ($\sigma_t$):** 1 float. Represented as the volume integral of the extinction coefficient. Activated via `softplus` and clamped to a minimum of 5 [S1, unofficial].
*   **Albedo ($\omega$):** 3 floats (RGB). Activated via `sigmoid` to ensure values in $[0, 1]$ [S1, unofficial].
*   **HG g parameter:** 1 float. The Henyey-Greenstein phase function anisotropy factor. Computed as $0.8 \cdot \tanh(\text{learnable})$ to enforce forward scattering [S1, unofficial].
*   **Multiple Scattering Weights ($w_n$):** 6 floats. Octave energy weights for a 6-order multiple scattering approximation (based on Frostbite/Wrenninge). Activated via `softplus` [S1, unofficial].

**Stage 2 Sidecar Files:**
*   `env_net.pt`: PyTorch weights for the global environment network (MLP) [S1, unofficial].
*   `sky_transfer.npy`: NumPy array containing the sky visibility SH transfer vectors $V_{lm}(x)$ for each Gaussian [S1, unofficial].
*   `env.json`: Configuration file for the environment network [S1, unofficial].

### Numbered Steps

#### Stage 1: Physical Parameter Optimization
1.  **Initialize:** Create 3D Gaussians from an initial point cloud (e.g., from photogrammetry or volume sampling).
2.  **Rasterization Setup:** Configure the rasterizer to use analytic optical thickness computation (`use_analytic_tau`). This computes the per-pixel accumulated optical thickness $\tau$ along the ray, where opacity is derived physically as $\alpha = 1 - \exp(-\tau)$ [S1, unofficial].
3.  **Lighting Pass (T_light):** Compute the self-shadowing transmittance $T_{light}$ for each Gaussian along the sun direction. This is done via a separate "lighting space" rasterization pass:
    *   Use a narrow FOV perspective camera to simulate the directional sun.
    *   For each Gaussian, record the front-facing accumulated optical thickness $\tau$ using a CUDA kernel (`record_front_tau`).
    *   Compute the weighted mean of $\alpha \cdot T$ over the sun-facing footprint of the Gaussian [S1, unofficial].
4.  **Radiance Computation:** For each pixel, accumulate radiance from front-to-back sorted Gaussians. The radiance contribution of a Gaussian is:
    $$ L = \omega \cdot L_{sun} \cdot \sum_{n} w_n \cdot T_{light}^{b^n} \cdot P(v_i, v_o; g \cdot c^n) $$
    where $L_{sun}$ is the sun irradiance, $P$ is the Henyey-Greenstein phase function, $v_i$ is the incident direction ($-v_l$), and $v_o$ is the view direction [S1, unofficial].
5.  **Tonemapping:** Apply a fixed Narkowicz ACES tonemapping curve to convert linear HDR radiance to LDR for comparison with ground truth images. Optionally, use learnable ACES coefficients if the ground truth uses a different filmic curve [S1, unofficial].
6.  **Optimization:** Minimize the difference between rendered and ground truth images using an optimizer (e.g., Adam). Optimize position, scale, rotation, $\sigma_t$, $\omega$, $g$, and $w_n$ [S1, unofficial].

#### Stage 2: Environment Network Training
1.  **Freeze Geometry:** Load the Stage 1 model and freeze all Gaussian parameters (position, scale, rotation, $\sigma_t$, $\omega$, $g$, $w_n$) [S1, unofficial].
2.  **Precompute Sky Visibility ($V_{lm}$):**
    *   For each Gaussian, compute the sky visibility transfer vectors $V_{lm}(x)$ in Spherical Harmonics (SH).
    *   Scan the hemisphere in $N$ directions using the `compute_T_light_raster` function to calculate geometric occlusion from the cloud itself.
    *   $V_{lm}$ is colorless and purely geometric, computed once on the frozen point set [S1, unofficial].
    *   Default SH order is 2 (9 coefficients), configurable via `--env_sh_order` [S1, unofficial].
3.  **Train Environment Network:**
    *   Train a global MLP (the environment network) to predict the sky irradiance field $E_{lm}(v_l)$ and the sun transmittance parameters.
    *   The network takes the sun direction $v_l$ as input.
    *   Use a learning rate of $1e-3$ with an independent optimizer, decaying to $0.1\times$ [S1, unofficial].
4.  **Relighting Rendering:**
    *   For any new sun direction $v_l$, compute the final radiance $L$ as:
        $$ L = T_{sun}(v_l) \odot L_{stage1} + \omega \cdot \sum_{lm} E_{lm}(v_l) \cdot V_{lm}(x) $$
    *   $T_{sun}(v_l)$ is the multiplicative sun-atmosphere transmittance (see Formula section).
    *   The second term is the additive sky inscattering fill [S1, unofficial].
5.  **Save Outputs:** Save `env_net.pt`, `sky_transfer.npy`, and `env.json` alongside the PLY file [S1, unofficial].

### Formulas
**Sun Transmittance ($T_{sun}$):**
Modeled as a 3-parameter analytic expression:
$$ T_{sun}(v_l) = \exp(-m(\theta) \cdot \tau_{rgb}) $$
where:
*   $m(\theta)$ is the Kasten-Young air mass, a fixed geometry function of sun elevation $\theta$ [S1, unofficial].
*   $\tau_{rgb} = (\tau_R, \tau_G, \tau_B)$ are the learnable zenith optical thicknesses for Red, Green, and Blue channels [S1, unofficial].
*   Note: $\tau_B > \tau_R$ due to Rayleigh scattering ($\propto \lambda^{-4}$), causing darkening and reddening at low sun angles [S1, unofficial].
*   If the sun is below the horizon, a `smoothstep` gate extinguishes the direct sun term [S1, unofficial].

**Final Radiance ($L$):**
$$ L = T_{sun}(v_l) \odot L_{stage1} + \omega \cdot \sum_{lm} E_{lm}(v_l) \cdot V_{lm}(x) $$
*   $L_{stage1}$: The radiance computed in Stage 1 (sun term only).
*   $E_{lm}(v_l)$: Sky irradiance field represented by low-order SH, predicted by the global MLP [S1, unofficial].
*   $V_{lm}(x)$: Precomputed sky visibility transfer vectors for Gaussian at $x$ [S1, unofficial].
*   $\omega$: Frozen albedo [S1, unofficial].

### Parameters
*   **SH Order:** 2 (9 coefficients) for $E_{lm}$ and $V_{lm}$ by default [S1, unofficial].
*   **Stage 2 Learning Rate:** $1e-3$, decaying to $0.1\times$ [S1, unofficial].
*   **Tonemapping:** Fixed Narkowicz ACES by default. Optional learnable ACES coefficients [S1, unofficial].
*   **T_light Computation:** Rasterization-based (`use_analytic_tau`) by default. Voxel-based alternative available via `--tlight_voxel` [S1, unofficial].

### Edge Cases and Pitfalls
*   **Sun Below Horizon:** The direct sun term must be multiplicative with $T_{sun} \le 1$. A purely additive term cannot express the darkening (attenuation) caused by atmospheric path length. A `smoothstep` gate is used to extinguish the sun term when the sun is below the horizon [S1, unofficial].
*   **Data Bias:** The sun direction coverage in the training dataset must be uniform to ensure healthy geometric shadow gradients. Biased direction data can cause vertical extensions to escape supervision [S1, unofficial].
*   **Tonemapping Mismatch:** If the ground truth is true linear HDR (no tonemapping), the tonemapping must be disabled (`--no-tonemap_aces`). Using learnable ACES is inappropriate as it cannot represent the identity function [S1, unofficial].
*   **Gradient Continuity:** The lighting pass (`T_light`) supports full continuous gradients for $\tau$, weights, normalization, and Gaussian transforms. However, sun camera framing, sorting, coverage, and threshold branches are treated as constants during backpropagation [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import numpy as np

def stage1_train(gaussians, dataset, optimizer):
    """
    Stage 1: Train physical parameters under single sun direction.
    """
    for images, poses, sun_directions in dataset:
        # 1. Rasterize Gaussians with analytic tau
        # Compute per-pixel optical thickness tau
        tau = rasterize_analytic_tau(gaussians, poses)
        
        # 2. Compute opacity from tau
        alpha = 1.0 - torch.exp(-tau)
        
        # 3. Compute T_light (self-shadowing)
        # Use a separate light-space rasterization pass
        T_light = compute_light_space_transmittance(gaussians, sun_directions)
        
        # 4. Compute Radiance
        # L = omega * L_sun * sum(w_n * T_light^b^n * P(v_i, v_o))
        L_sun = get_sun_irradiance(sun_directions)
        phase_func = heney_greenstein(gaussians.g, sun_directions, poses.view_directions)
        
        # Multiple scattering octaves
        L = torch.zeros_like(images)
        for n in range(6):
            weight = gaussians.w_n[n]
            L += weight * (T_light ** (2**n)) * phase_func
            
        L = gaussians.omega * L_sun * L
        
        # 5. Tonemapping
        L_ldr = apply_aces_tonemap(L)
        
        # 6. Loss and Optimization
        loss = mse_loss(L_ldr, images)
        optimizer.zero_grad()
        loss.backward()
        optimizer.step()

def stage2_precompute(gaussians, num_hemisphere_samples=60):
    """
    Stage 2 Precomputation: Compute sky visibility SH transfer vectors.
    """
    V_lm = []
    for gaussian in gaussians:
        # Scan hemisphere in N directions
        visibility_coeffs = []
        for i in range(num_hemisphere_samples):
            direction = fibonacci_hemisphere_sample(i)
            # Compute geometric occlusion using rasterization
            T_light = compute_T_light_raster(gaussian, direction)
            # Convert to SH coefficients (order 2)
            sh_coeffs = convert_to_sh(T_light, order=2)
            visibility_coeffs.append(sh_coeffs)
        
        # Average or fit SH to the sampled visibility
        V_lm.append(fit_sh_from_samples(visibility_coeffs, order=2))
    
    return torch.stack(V_lm) # Shape: (N_gaussians, 9)

def stage2_train(gaussians, V_lm, env_dataset, env_network, optimizer):
    """
    Stage 2: Train environment network with frozen geometry.
    """
    # Freeze Gaussians
    for param in gaussians.parameters():
        param.requires_grad = False
        
    for images, poses, sun_directions in env_dataset:
        # 1. Get Sky Irradiance from MLP
        E_lm = env_network(sun_directions) # Shape: (N_gaussians, 9)
        
        # 2. Compute Inscaattering Term
        # Sum over SH coefficients: omega * sum(E_lm * V_lm)
        inscattering = torch.zeros_like(images)
        for gaussian_idx, gaussian in enumerate(gaussians):
            omega = gaussian.omega
            V = V_lm[gaussian_idx]
            E = E_lm[gaussian_idx]
            inscattering += omega * torch.sum(E * V)
            
        # 3. Compute Sun Transmittance
        # T_sun = exp(-m(theta) * tau_rgb)
        theta = get_sun_elevation(sun_directions)
        m_theta = kasten_young_air_mass(theta)
        tau_rgb = env_network.get_zenith_optical_thickness() # Learnable params
        T_sun = torch.exp(-m_theta * tau_rgb)
        
        # 4. Get Stage 1 Radiance (Sun Term Only)
        # Re-render Stage 1 radiance with current sun direction
        L_stage1 = render_stage1_radiance(gaussians, sun_directions)
        
        # 5. Final Radiance
        # L = T_sun * L_stage1 + inscattering
        L = T_sun * L_stage1 + inscattering
        
        # 6. Tonemapping and Loss
        L_ldr = apply_aces_tonemap(L)
        loss = mse_loss(L_ldr, images)
        
        optimizer.zero_grad()
        loss.backward()
        optimizer.step()

def render_relighted(gaussians, V_lm, env_network, sun_direction):
    """
    Real-time relighting inference.
    """
    # 1. Get Sky Irradiance
    E_lm = env_network(sun_direction)
    
    # 2. Compute Inscaattering
    inscattering = 0.0
    for gaussian_idx, gaussian in enumerate(gaussians):
        omega = gaussian.omega
        V = V_lm[gaussian_idx]
        E = E_lm[gaussian_idx]
        inscattering += omega * torch.sum(E * V)
        
    # 3. Compute Sun Transmittance
    theta = get_sun_elevation(sun_direction)
    m_theta = kasten_young_air_mass(theta)
    tau_rgb = env_network.get_zenith_optical_thickness()
    T_sun = torch.exp(-m_theta * tau_rgb)
    
    # 4. Get Stage 1 Radiance
    L_stage1 = render_stage1_radiance(gaussians, sun_direction)
    
    # 5. Final Radiance
    L = T_sun * L_stage1 + inscattering
    
    return apply_aces_tonemap(L)
```

## Operating characteristics
*   **Time Complexity:** Stage 1 training is comparable to standard 3DGS but with additional overhead for the lighting pass and physical parameter optimization. Stage 2 precomputation is $O(N \cdot H)$ where $N$ is the number of Gaussians and $H$ is the number of hemisphere samples. Stage 2 training is fast as it only optimizes a small MLP. Inference is real-time, dominated by the rasterization of Gaussians and a simple MLP evaluation per pixel [S1, unofficial].
*   **Memory Complexity:** Stores physical parameters per Gaussian (extinction, albedo, HG g, 6 scattering weights) plus sidecar files for the environment network and visibility vectors. The sidecar files are relatively small compared to the point cloud [S1, unofficial].
*   **Strengths:** Enables arbitrary-direction relighting without retraining the cloud geometry. Physically based parameters allow for realistic atmospheric effects (reddening at low sun angles). Real-time rendering capability [S1, unofficial].
*   **Limitations:** Assumes static cloud geometry (time dimension not implemented in the initial version). Relies on uniform sun direction coverage in the training dataset for robust relighting. The multiplicative sun term requires careful handling of sun below the horizon [S1, unofficial].
*   **When to Use:** When dynamic relighting of volume clouds is required in real-time applications, and the cloud geometry is static or can be approximated as such. Suitable for virtual production and interactive visualization [S1, unofficial].
*   **When Not to Use:** If the cloud geometry is highly dynamic and cannot be captured by a static Gaussian set. If the training data lacks sufficient sun direction diversity, relighting quality will degrade [S1, unofficial].

## Background
The method is based on 3D Gaussian Splatting (Kerbl et al., 2023) and the 3DGS-Volume-Cloud project. It was developed as an independent 4D implementation workspace, with an initial code baseline at commit `69fea2c8d5a13d6c0e41ac12ca45c80b71bd3403` [S1, unofficial]. The project aims to replace ray-marching volume clouds in game engines with physically parameterized 3D Gaussians for real-time rendering and dynamic relighting. It is used in the 4DGS-Volume-Cloud project for rendering volume clouds based on the WDAS Cloud dataset [S1, unofficial].

## Sources
[S1] hsiang0117/4DGS-Volume-Cloud, https://github.com/hsiang0117/4DGS-Volume-Cloud, fetched 2026-10-26 (unofficial)
