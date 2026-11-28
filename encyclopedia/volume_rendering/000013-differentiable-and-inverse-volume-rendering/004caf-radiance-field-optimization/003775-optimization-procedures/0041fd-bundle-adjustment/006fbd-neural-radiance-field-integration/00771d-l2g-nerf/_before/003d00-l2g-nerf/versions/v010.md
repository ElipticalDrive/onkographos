# L2G-NeRF

Also called: Local-to-Global Registration for Bundle-Adjusting Neural Radiance Fields

## Overview
L2G-NeRF is a bundle-adjusting method for Neural Radiance Fields that jointly optimizes neural 3D representations and camera poses by combining a local photometric alignment objective with a global geometric alignment objective [S1]. It solves the problem of suboptimal solutions caused by poor initialization, which plagues purely parametric methods, and the inability to handle large displacements, which affects purely non-parametric methods [S1]. It sets itself apart by using a two-stage process: first, a pixel-wise flexible alignment (non-parametric) to initialize the solution, followed by a frame-wise constrained parametric alignment (global) to enforce geometric consistency [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of $M$ images $\{I_i\}_{i=1}^M$, query pixel coordinates $\{x_j\}_{j=1}^N$ (for 2D) or 3D ray coordinates (for 3D), and initial camera poses (typically identity).
*   **Outputs:** Optimized neural field parameters $\Theta$, optimized camera poses $\{T_i\}_{i=1}^M$, and local transformation fields $\{T_{ji}\}$.

**Data Structures:**
1.  **Neural Field MLP ($f_R$):**
    *   **3D NeRF:** Input is 3D coordinate $z \in \mathbb{R}^3$. Output is 4D vector $[c; \sigma] \in \mathbb{R}^4$, where $c$ is RGB color and $\sigma$ is volume density.
    *   **2D Image:** Input is 2D coordinate $x \in \mathbb{R}^2$. Output is 3D vector (RGB color).
2.  **Warp Neural Field MLP ($f_W$):**
    *   Input is 2D coordinate $x \in \mathbb{R}^2$ and frame-dependent embedding $\ell_i \in \mathbb{R}^d$.
    *   Output is a Lie algebra vector (se(2) for 2D rigid/homography or sl(3) for 2D homography).
3.  **Camera Pose:** $T_i \in SE(3)$ for 3D, parametrized as $[R|t]$. $R = \exp(r)$ via exponential map from $\mathfrak{so}(3)$ to $SO(3)$, $t \in \mathbb{R}^3$.
4.  **Local Transformation:** $T_{ji} \in SE(2)$ or $SL(3)$ for 2D, modeled by $W(x_j; \ell_i, \Phi)$.

**Numbered Steps:**

1.  **Initialization:**
    *   Initialize all camera poses to identity: $T_i = I \quad \forall i$.
    *   Perturb initial poses using right multiplication to alter viewing directions and distances. Additive noise is sampled from $r \sim \mathcal{N}(0, n_r I)$ and $t \sim \mathcal{N}(0, n_t I)$, where $n_r$ and $n_t$ are base multipliers [S1].
    *   Initialize neural field parameters $\Theta$ and warp field parameters $\Phi$ [S1].

2.  **Stage 1: Local Alignment (Non-Parametric):**
    *   For each image $i$ and pixel $j$, predict a local transformation $T_{ji}$ using the warp neural field:
        $$ T_{ji} = W(x_j; \ell_i, \Phi) = \exp(f_W(x_j; \ell_i, \Phi)) $$
        where $\exp(\cdot)$ maps the Lie algebra output to the Lie group (SE(2) or SL(3)) [S1].
    *   Optimize $\Theta$, $\Phi$, and $\{\ell_i\}$ by minimizing the photometric reconstruction error between the rendered image and the target image $I_i$:
        $$ \mathcal{L}_{local} = \sum_{i=1}^M \sum_{j=1}^N \| R(T_{ji} x_j; \Theta) - I_i(x_j) \|^2 $$
    *   For 3D NeRF, the rendering function $R(T_{ji} x_j; \Theta)$ is computed via volume rendering. Given a ray defined by pixel $x$ and depth $z$, the 3D point is $z x$. The color is:
        $$ R(x) = \int_{z_{near}}^{z_{far}} T(x, z) \sigma(zx) c(zx) dz $$
        where $T(x, z) = \exp(-\int_{z_{near}}^z \sigma(z'x) dz')$. Numerically, this is discretized using $K$ points sampled along the ray at depths $\{z_1, ..., z_K\}$:
        $$ R(x) = g(y_1, ..., y_K), \quad y_k = f_R(T z_k x; \Theta) $$
        where $g$ is the differentiable compositing function [S1].

3.  **Stage 2: Global Alignment (Parametric):**
    *   Solve for the global frame-wise transformation $T^*_i$ for each image $i$ by minimizing the alignment error between the local transformations $T_{ji}$ and the global transformation $T_i$ applied to the coordinates. This is an orthogonal Procrustes problem:
        $$ T^*_i = \arg \min_{T_i} \sum_{j=1}^N \| T_{ji} x_j - T_i x_j \|^2 $$
    *   The optimal rigid transformation $T=[R|t]$ is computed in closed form using SVD. Let $\bar{x}$ and $\overline{Tx}$ be the centroids of $\{x_j\}$ and $\{T_{ji} x_j\}$ respectively. Compute the covariance matrix $H$:
        $$ H = \sum_{j=1}^N (x_j - \bar{x})(T_{ji} x_j - \overline{Tx})^T = U S V^T $$
        Then:
        $$ R = V U^T, \quad t = -R\bar{x} + \overline{Tx} $$
        (Note: For homography, a similar linear system $A_j h = 0$ is solved [S1]).
    *   Apply a soft constraint to the local alignment by adding the global objective to the loss:
        $$ \mathcal{L}_{global} = \lambda \sum_{i=1}^M \sum_{j=1}^N \| T_{ji} x_j - T^*_i x_j \|^2 $$
    *   Jointly optimize $\Theta$, $\{T_i\}$, $\Phi$, and $\{\ell_i\}$ by minimizing the combined loss:
        $$ \min_{\{T_{ji}\}, \Theta, \{T_i\}} \mathcal{L}_{local} + \mathcal{L}_{global} $$
    *   The global transformations $T^*_i$ are updated iteratively during optimization to constrain the local transformations [S1].

4.  **Convergence Schedule:**
    *   Linearly increase noise multipliers $n_r$ and $n_t$ by a common factor $\{n_c\}_{c=2}^9$ starting from base values $n_r=0.01, n_t=0.1$ to aid convergence [S1].

**Parameters:**
*   $\lambda$: Weighting factor for the global constraint. Set to $1 \times 10^2$ for synthetic/object-centric scenes. For forward-facing LLFF scenes, floated between $1 \times 10^2$ and $1 \times 10^5$ [S1].
*   $n_r$: Base rotation noise multiplier. Scene-dependent, typically 0.01 to 0.09 [S1].
*   $n_t$: Base translation noise multiplier. Scene-dependent, typically 0.3 to 0.7 [S1].
*   $K$: Number of points sampled along a ray for volume rendering discretization [S1].
*   $\Theta$: Neural field MLP weights [S1].
*   $\Phi$: Warp neural field MLP weights [S1].
*   $\ell_i$: Frame-dependent embedding for frame $i$ [S1].

**Pitfalls and Edge Cases:**
*   **Poor Initialization:** Parametric methods alone fail if initialized poorly. L2G-NeRF mitigates this via the local stage [S1].
*   **Large Displacements:** Non-parametric methods alone fail with large displacements. L2G-NeRF mitigates this via the global stage [S1].
*   **Identity Initialization for 360° Scenes:** L2G-NeRF fails to recover camera poses from identity transformation for inward-facing 360° scenes with large rotation displacements [S1].
*   **High $\lambda$:** A large $\lambda$ ($1 \times 10^5$) improves geometric accuracy but reduces robustness, potentially causing divergence (e.g., on the Flower scene) [S1].
*   **Removing Global Objective:** Removing $L_{global}$ causes divergence to suboptimal poses and ghosting artifacts [S1].

## Operating characteristics
*   **Time Complexity:** More efficient than global registration methods (like BARF) because it computes gradients w.r.t. local pose rather than global pose [S1].
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** On NVIDIA GeForce RTX 2080 Ti, L2G-NeRF takes ~4.5 hours for synthetic objects and ~8 hours for real-world scenes. BARF takes ~8 and ~10.5 hours respectively [S1].
*   **Robustness:** Tolerates larger noise perturbations before divergence than BARF (fails at $n_c=8$ vs $n_c=4$ for L2G-NeRF vs BARF) [S1].
*   **Strengths:** High-fidelity reconstruction, resolves large camera pose misalignment, robust to poor initialization [S1].
*   **Limitations:** Fails on inward-facing 360° scenes with large rotations from identity init; sensitive to $\lambda$ setting [S1].
*   **When to use:** When camera poses are unknown or poorly estimated, and standard NeRF optimization fails due to initialization issues. Suitable for both 2D neural images and 3D NeRF [S1].

## Background
L2G-NeRF was proposed in the paper "Local-to-Global Registration for Bundle-Adjusting Neural Radiance Fields" [S1]. It addresses the limitations of existing bundle-adjusting NeRF methods, which are susceptible to suboptimal solutions if poorly initialized (parametric methods) or unable to handle large displacements (non-parametric methods) [S1]. The method builds on the concept of photometric bundle adjustment [S1].

## Sources
[S1] Local-to-Global Registration for Bundle-Adjusting Neural Radiance Fields, https://doi.org/10.1109/cvpr52729.2023.00799, fetched 2026-10-29
