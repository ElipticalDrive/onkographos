# Physical Parameter Optimization

Also called: 3DGS Volume Parameters, Participating Medium Parameters, Physical Gaussian Attributes, g Optimization, 物理化的高斯参数

## Overview
Physical Parameter Optimization replaces the standard spherical harmonic (SH) color and empirical opacity attributes of 3D Gaussian Splatting with a set of physical quantities representing a participating medium [S1, unofficial]. This allows for real-time rendering and dynamic relighting (arbitrary sun direction) by solving the radiative transfer equation through the Gaussian primitives rather than approximating lighting with SH coefficients [S1, unofficial]. It is distinct from standard 3DGS training by optimizing extinction, albedo, and scattering asymmetry instead of color and opacity [S1, unofficial].

## How it works
### Inputs and Outputs
**Inputs:**
- A dataset of images with known camera poses and sun directions (Stage 1: sun-only, black background; Stage 2: environment lighting) [S1, unofficial].
- Initial 3D Gaussian point cloud (position, scale, rotation, and initial physical parameters) [S1, unofficial].

**Outputs:**
- Optimized Gaussian attributes: position, scale, rotation, extinction coefficient ($\sigma_t$), scattering albedo ($\omega$), Henyey-Greenstein asymmetry factor ($g$), and multiple scattering octave weights ($w_n$) [S1, unofficial].
- (Stage 2 only) Optimized environment light network parameters [S1, unofficial].

### Data Structures
Each Gaussian primitive stores the following physical parameters in addition to standard geometric attributes (position $\mathbf{x}$, scale $\mathbf{s}$, rotation $\mathbf{q}$):

1.  **Extinction Coefficient ($\sigma_t$):**
    -   **Type:** Scalar.
    -   **Meaning:** Extinction cross-section, representing the volume integral of the extinction coefficient [S1, unofficial].
    -   **Activation:** `softplus`, clamped at 5.0 [S1, unofficial].
2.  **Scattering Albedo ($\omega$):**
    -   **Type:** RGB Vector (3 floats).
    -   **Meaning:** Ratio of scattering coefficient to extinction coefficient [S1, unofficial].
    -   **Activation:** `sigmoid` [S1, unofficial].
3.  **Henyey-Greenstein Asymmetry Factor ($g$):**
    -   **Type:** Scalar.
    -   **Meaning:** Anisotropy factor for the Henyey-Greenstein phase function [S1, unofficial].
    -   **Activation:** $0.8 \times \tanh(g_{raw})$, favoring forward scattering [S1, unofficial].
4.  **Multiple Scattering Octave Weights ($w_n$):**
    -   **Type:** Array of 6 learnable scalars.
    -   **Meaning:** Energy weights for the 6th-order multiple scattering octave approximation (based on Frostbite/Wrenninge) [S1, unofficial].
    -   **Activation:** `softplus` [S1, unofficial].

### Optimization Steps
The training process is divided into two stages [S1, unofficial]:

**Stage 1: Physical Parameter Training**
1.  **Initialization:** Initialize Gaussians with standard 3DGS geometry and initial physical parameters [S1, unofficial].
2.  **Rendering:** For each training iteration, render the scene using the physical volume rendering equation. The radiance $L$ is computed by integrating the extinction, albedo, and phase function along the ray [S1, unofficial].
3.  **Loss Calculation:** Compute the loss $L$ as a weighted sum of L1 and DSSIM:
    $$ L = 0.8 \cdot L1 + 0.2 \cdot DSSIM $$
    where the DSSIM weight $\lambda_{dssim} = 0.2$ [S1, unofficial].
4.  **Regularization:** Apply Soft Anisotropy Regularization if the log-ratio of scales exceeds `aniso_ratio_max`:
    $$ L_{aniso} = \lambda_{aniso} \cdot (\log(\text{ratio}))^2 $$
    with $\lambda_{aniso} = 0.001$ [S1, unofficial].
5.  **Optimization:** Update all parameters using Adam optimizer with specific learning rates (see Parameters).
6.  **Densification and Pruning:**
    -   **Interval:** Iterations 500 to 15,000, every 100 iterations [S1, unofficial].
    -   **Densification:** Use adaptive densification with `densify_top_frac=0.005` and absolute lower bound $5 \times 10^{-5}$ [S1, unofficial]. Incorporate scale gradients if $> 10^{-6}$ [S1, unofficial].
    -   **Pruning:** Remove Gaussians with mean contribution $\sum (\alpha \cdot T) < 10^{-4}$ [S1, unofficial].
    -   **Post-Densify Pruning:** Extra pruning every 1,000 iterations during densification phase [S1, unofficial].
    -   **Resurrection:** Every 3,000 iterations (during densification phase), reset the lowest contribution 5% of $\sigma_t$ Gaussians to 0.1 [S1, unofficial].
7.  **Tonemapping:** Apply fixed ACES tonemapping by default. If `tonemap_learnable` is enabled, optimize 4 ACES coefficients with a monotonicity penalty $\lambda_{tonemap\_mono} = 10^{-2}$ to prevent curve inversion in [0, 8] [S1, unofficial].

**Stage 2: Environment Light Optimization**
1.  **Freeze:** Freeze all Stage 1 geometric and physical parameters [S1, unofficial].
2.  **Train:** Optimize a global environment light network (MLP for sun direction $T_{sun}$ and SH coefficients for sky irradiance $E_{lm}$ and visibility $V_{lm}$) using the `env_lr` learning rate [S1, unofficial].

### Parameters
-   **Learning Rates (Initial / Final):**
    -   Position: $1.6 \times 10^{-4} / 1.6 \times 10^{-6}$ [S1, unofficial].
    -   $\sigma_t$: $0.025$ [S1, unofficial].
    -   $\omega$: $0.0025$ [S1, unofficial].
    -   $g$: $0.0025$ [S1, unofficial].
    -   $w_n$: $0.0025$ [S1, unofficial].
    -   Scale: $0.005$ [S1, unofficial].
    -   Rotation: $0.001$ [S1, unofficial].
    -   Environment Light (Stage 2): $10^{-3}$ [S1, unofficial].
-   **Decay:** Physical parameter learning rates decay exponentially to 1/10 of their initial value over the course of training [S1, unofficial]. Position LR decays over `position_lr_max_steps` (default 30,000) [S1, unofficial].
-   **Position LR Delay Mult:** $0.01$ [S1, unofficial].
-   **Total Iterations:** 30,000 [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def train_stage1(gaussians, dataset, iterations=30000):
    # Initialize optimizers with specific learning rates
    opt_pos = Adam(gaussians.position, lr=1.6e-4, delay_mult=0.01)
    opt_sigma_t = Adam(gaussians.sigma_t, lr=0.025)
    opt_omega = Adam(gaussians.omega, lr=0.0025)
    opt_g = Adam(gaussians.g, lr=0.0025)
    opt_w = Adam(gaussians.w, lr=0.0025)
    opt_scale = Adam(gaussians.scale, lr=0.005)
    opt_rot = Adam(gaussians.rotation, lr=0.001)
    
    for iter in range(iterations):
        # 1. Sample camera and sun direction
        cam, sun = dataset.sample()
        
        # 2. Render physical volume
        # Integrate extinction, albedo, and HG phase function
        rendered_image = render_physical_volume(gaussians, cam, sun)
        
        # 3. Compute Loss
        gt_image = dataset.get_gt(cam)
        l1_loss = L1(rendered_image, gt_image)
        dssim_loss = DSSIM(rendered_image, gt_image)
        loss = 0.8 * l1_loss + 0.2 * dssim_loss
        
        # 4. Regularization
        if check_anisotropy(gaussians):
            loss += 0.001 * anisotropy_penalty(gaussians)
            
        # 5. Backpropagate and Update
        loss.backward()
        
        opt_pos.step()
        opt_sigma_t.step()
        opt_omega.step()
        opt_g.step()
        opt_w.step()
        opt_scale.step()
        opt_rot.step()
        
        opt_pos.zero_grad()
        opt_sigma_t.zero_grad()
        opt_omega.zero_grad()
        opt_g.zero_grad()
        opt_w.zero_grad()
        opt_scale.zero_grad()
        opt_rot.zero_grad()
        
        # 6. Densification and Pruning (Iter 500-15000, step 100)
        if 500 <= iter <= 15000 and iter % 100 == 0:
            densify_gaussians(gaussians, opt_pos, opt_scale, threshold=5e-5, top_frac=0.005)
            prune_gaussians(gaussians, threshold=1e-4)
            
            # Resurrection every 3000 iterations
            if iter % 3000 == 0:
                resurrect_gaussians(gaussians, fraction=0.05, value=0.1)
                
            # Post-densify pruning every 1000 iterations
            if iter % 1000 == 0:
                prune_gaussians(gaussians, threshold=1e-4)
                
        # 7. Decay Learning Rates
        decay_learning_rates(opt_pos, opt_sigma_t, opt_omega, opt_g, opt_w, iter, max_steps=30000)

def train_stage2(gaussians, env_network, dataset, iterations=30000):
    # Freeze Stage 1 parameters
    gaussians.position.requires_grad = False
    gaussians.scale.requires_grad = False
    gaussians.rotation.requires_grad = False
    gaussians.sigma_t.requires_grad = False
    gaussians.omega.requires_grad = False
    gaussians.g.requires_grad = False
    gaussians.w.requires_grad = False
    
    opt_env = Adam(env_network.parameters(), lr=1e-3)
    
    for iter in range(iterations):
        cam, sun = dataset.sample()
        rendered_image = render_physical_volume_with_env(gaussians, env_network, cam, sun)
        gt_image = dataset.get_gt(cam)
        
        loss = 0.8 * L1(rendered_image, gt_image) + 0.2 * DSSIM(rendered_image, gt_image)
        loss.backward()
        opt_env.step()
        opt_env.zero_grad()
        
        decay_learning_rates(opt_env, iter, max_steps=30000)
```

### Edge Cases and Pitfalls
-   **Anisotropy Control:** Position LR must sync with iterations to prevent anisotropy loss of control [S1, unofficial].
-   **Resurrection Loop:** Resurrection is only active during the densification period to avoid net destruction loops [S1, unofficial].
-   **Tonemapping:** If using learnable tonemapping, ensure monotonicity penalty is applied to prevent curve inversion in [0, 8] [S1, unofficial].
-   **CUDA Compilation:** On Windows, ensure MSVC toolset version is compatible with CUDA (e.g., MSVC 14.44 for CUDA 12.8) [S1, unofficial].

## Operating characteristics
-   **Time Complexity:** Similar to standard 3DGS training, but with additional overhead for physical volume integration and multiple scattering octave calculations [S1, unofficial].
-   **Memory Complexity:** Higher than standard 3DGS due to additional physical parameters ($\sigma_t, \omega, g, w_n$) per Gaussian [S1, unofficial].
-   **Performance:** Enables real-time rendering and dynamic relighting [S1, unofficial].
-   **Strengths:** Physically plausible lighting, dynamic sun direction support, stable physical parameters [S1, unofficial].
-   **Limitations:** Complex training pipeline (two stages), sensitive to learning rate decay and densification parameters [S1, unofficial].
-   **When to Use:** When dynamic relighting and physically accurate volume rendering are required [S1, unofficial].
-   **When Not to Use:** When simple static lighting or empirical color representation is sufficient [S1, unofficial].

## Background
Physical Parameter Optimization was introduced as part of the 4DGS-Volume-Cloud project, building on the 3DGS-Volume-Cloud repository [S1, unofficial]. It aims to replace ray-marching volume clouds in game engines with 3D Gaussian Splatting for real-time rendering and dynamic relighting [S1, unofficial]. The method was developed by hsiang0117 and is documented in the 4DGS-Volume-Cloud GitHub repository [S1, unofficial].

## Sources
[S1] hsiang0117/4DGS-Volume-Cloud, https://github.com/hsiang0117/4DGS-Volume-Cloud, fetched 2026-10-26 (unofficial)
