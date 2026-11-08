# 3DGS Adaptive Density Control

Also called: Dynamic Gaussian Densification, Pruning and Splitting, cloning, splitting, and pruning

## Overview
3DGS Adaptive Density Control (ADC) is a dynamic optimization strategy that adjusts the number and distribution of 3D Gaussian primitives during training to prevent under-reconstruction (missing geometry) and over-reconstruction (coarse approximations) [S1][S2, unofficial]. It sets itself apart from static initialization methods by using view-space positional gradient magnitudes to trigger cloning, splitting, and pruning operations at regular intervals, ensuring the representation remains compact while capturing fine details [S1][S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of 3D Gaussians $\mathcal{G}$, each defined by position $\mu$, covariance $\Sigma$ (or scale $s$ and rotation $q$), opacity $\alpha$, and spherical harmonic coefficients $c$. Training images and their camera poses.
*   **Outputs:** An updated set of Gaussians $\mathcal{G}'$ with adjusted counts, positions, scales, and opacities.

**Data Structures:**
*   **Gaussian Attributes:** Each Gaussian $i$ stores:
    *   $\mu_i \in \mathbb{R}^3$: Position.
    *   $s_i \in \mathbb{R}^3$: Scale (non-negative).
    *   $q_i \in \mathbb{R}^4$: Rotation (unit quaternion).
    *   $\alpha_i \in [0, 1]$: Opacity.
    *   $c_{i,0}, \dots, c_{i,N}$: SH coefficients.
*   **Gradient Accumulators:** For each Gaussian, a running sum of view-space positional gradient magnitudes $\sum \|\nabla_{\mu} L\|$ and a counter $v_i$ (number of visible iterations since last densification) [S2, unofficial].

**Parameters:**
*   Warm-up period: $T_{warmup} = 600$ iterations [S2, unofficial].
*   Densification interval: $\Delta T = 100$ iterations [S2, unofficial].
*   Gradient threshold: $\tau_{grad} = 2 \times 10^{-4}$ [S2, unofficial].
*   Opacity threshold: $\tau_{op} = 0.05$ [S2, unofficial].
*   Final densification iteration: $T_{final} = 14900$ [S2, unofficial].
*   Opacity reset interval: $\Delta T_{reset} = 3000$ iterations [S2, unofficial].
*   Opacity reset clip value: $\alpha_{clip} = 0.01$ [S2, unofficial].
*   Learning rate for scale: $\eta_s = 0.005$ [S2, unofficial].
*   Learning rate for opacity: $\eta_\alpha = 0.025$ (official codebase) or $0.05$ (original paper) [S2, unofficial].

**Steps:**

1.  **Initialization:** Initialize Gaussians from sparse Structure-from-Motion (SfM) points [S1][S2, unofficial]. Set all gradient accumulators to zero. Set iteration counter $t = 0$.

2.  **Training Loop:** For each iteration $t$:
    a. Render training images using current Gaussians.
    b. Compute loss $L = 0.8 L_1 + 0.2 (1 - SSIM)$ [S2, unofficial].
    c. Compute gradients $\nabla_{\mu} L$ for all Gaussians visible in the current view (inside the viewing frustum) [S2, unofficial].
    d. For each visible Gaussian $i$:
        i. Accumulate gradient magnitude: $G_i \leftarrow G_i + \|\nabla_{\mu_i} L\|$.
        ii. Increment visibility counter: $v_i \leftarrow v_i + 1$.
    e. Update Gaussian parameters using Adam optimizer with specified learning rates [S2, unofficial].
        *   Mean: $\eta_\mu$ exponentially decayed from $1.6 \times 10^{-4}$ to $1.6 \times 10^{-6}$ [S2, unofficial].
        *   Scale: $\eta_s = 0.005$ [S2, unofficial].
        *   Rotation: $\eta_q = 0.001$ [S2, unofficial].
        *   Opacity: $\eta_\alpha = 0.025$ (official) or $0.05$ (original) [S2, unofficial].
        *   SH Coefficients: $0^{th}$ degree $\eta_{c,0} = 2.5 \times 10^{-3}$, remaining $\eta_{c,k} = 1.25 \times 10^{-4}$ [S2, unofficial].
    f. **Opacity Reset:** If $t > T_{warmup}$ and $t \le T_{final}$ and $(t - T_{warmup}) \pmod{\Delta T_{reset}} == 0$:
        i. For all Gaussians $i$: $\alpha_i \leftarrow \min(\alpha_i, \alpha_{clip})$ [S2, unofficial].

3.  **Densification Check:** If $t > T_{warmup}$ and $t \le T_{final}$ and $(t - T_{warmup}) \pmod{\Delta T} == 0$:
    a. **Selection:** Identify Gaussians for densification where average gradient $G_i / v_i > \tau_{grad}$ [S2, unofficial].
    b. **Splitting:** For selected Gaussians with scale $s_i$ (e.g., largest axis) greater than a threshold (implied "large" relative to scene, but specific threshold not given in sources, typically based on bounding box or fixed size [G]):
        i. Create two new Gaussians $j, k$ from parent $i$.
        ii. Sample positions $\mu_j, \mu_k$ from the parent Gaussian distribution $\mathcal{N}(\mu_i, \Sigma_i)$ [S2, unofficial].
        iii. Copy attributes: $\alpha_j = \alpha_k = \alpha_i / 2$, $c_j = c_k = c_i$, $q_j = q_k = q_i$.
        iv. Scale down: $s_j = s_k = s_i \times 0.5$ (or similar reduction factor, exact factor not specified in sources [G]).
        v. Remove parent $i$.
        vi. Add $j, k$ to $\mathcal{G}$.
    c. **Cloning:** For selected Gaussians that are "small" (not split):
        i. Duplicate Gaussian $i$ in place: Create $j$ with $\mu_j = \mu_i$, $s_j = s_i$, $\alpha_j = \alpha_i / 2$, $c_j = c_i$, $q_j = q_i$.
        ii. Set $\alpha_i = \alpha_i / 2$.
        iii. Add $j$ to $\mathcal{G}$.
    d. **Pruning:** For all Gaussians $i$:
        i. If $\alpha_i < \tau_{op}$, remove $i$ from $\mathcal{G}$ [S2, unofficial].
        ii. If scale $s_i$ is too large relative to scene size, remove $i$ from $\mathcal{G}$ [S2, unofficial]. (Specific threshold not given [G]).
    e. **Reset:** Reset $G_i = 0$ and $v_i = 0$ for all remaining Gaussians.

**Edge Cases and Pitfalls:**
*   **Gradient Clipping:** Colors are not clipped to $[0,1]$ during training to avoid zeroing gradients for pixels marginally above one, which can cause issues with white walls/sky [S2, unofficial]. Clipping is only applied during inference [S2, unofficial].
*   **SH Degree Progression:** Only $0^{th}$ degree SH is used initially. Higher degrees are enabled every 1000 iterations [S2, unofficial].
*   **Opacity Learning Rate:** Using $\eta_\alpha = 0.05$ (original) creates more Gaussians than $\eta_\alpha = 0.025$ (official), which yields cleaner reconstructions with fewer Gaussians [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Densification steps add overhead proportional to the number of Gaussians being split/cloned/pruned. Since densification occurs every 100 iterations, the average overhead is low. Rendering complexity remains $O(N)$ per pixel, where $N$ is the number of Gaussians.
*   **Memory Complexity:** Memory usage increases during densification and decreases during pruning. The final number of Gaussians depends on scene complexity and hyperparameters.
*   **Performance:** The method enables real-time rendering by maintaining a compact set of Gaussians that adapt to scene detail [S1].
*   **Strengths:** Automatically handles varying levels of detail; prevents over-smoothing in complex regions; removes artifacts (floaters) via pruning and opacity resets.
*   **Limitations:** Requires careful tuning of thresholds ($\tau_{grad}, \tau_{op}$); densification stops after iteration 14900, so final detail is fixed [S2, unofficial].

## Background
3DGS Adaptive Density Control was introduced by Kerbl et al. in the original 3D Gaussian Splatting paper [S2, unofficial]. It addresses the limitation of sparse SfM initialization, which is insufficient to represent complex geometry [S1]. The method has been refined in subsequent works, such as Faster-GS, which analyzed and improved the optimization process, including changes to the opacity learning rate [S2, unofficial].

## Sources
[S1] Recent Advances and Trends in Learning‐Based 3D Representations, https://doi.org/10.1111/cgf.70530, fetched 2026-10-26
[S2] Faster-GS: Analyzing and Improving Gaussian Splatting Optimization, https://arxiv.org/abs/2602.09999v1, fetched 2026-10-26 (unofficial)
