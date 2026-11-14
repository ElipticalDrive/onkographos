# Deformable Neural Radiance Fields (Nerfies)

Also called: Deformable NeRF, Nerfies

## Overview
Deformable Neural Radiance Fields (Nerfies) reconstructs non-rigidly deforming scenes by decomposing the scene into a static canonical template volume (a standard NeRF) and a per-observation continuous deformation field [S1]. Unlike methods that condition deformation on time or define it on discrete grids/meshes, Nerfies conditions the deformation on a learned per-frame latent code, allowing it to model complex, non-rigid motions from casually captured images [S1]. It distinguishes itself from sibling methods by introducing elastic regularization to resolve ambiguities between motion and scaling, and a coarse-to-fine optimization strategy to avoid local minima [S1].

## How it works
**Inputs:** A set of $n$ images of a non-rigidly deforming scene, captured from varying viewpoints. Each image $i$ has an associated camera pose.
**Outputs:** A canonical radiance field $F$, a deformation field $T$, and per-frame latent codes $\omega_i$ (deformation) and $\psi_i$ (appearance).

**Data Structures:**
1.  **Canonical NeRF ($F$):** An MLP mapping $(x', d, \psi_i) \to (\sigma, c)$, where $x'$ is a canonical 3D coordinate, $d$ is viewing direction, $\sigma$ is density, and $c$ is color.
2.  **Deformation MLP ($W$):** An MLP mapping $(x, \omega_i) \to (r, v)$, where $x$ is an observation-space 3D coordinate, $\omega_i$ is the deformation latent code, $r \in \mathbb{R}^3$ is the rotation vector, and $v \in \mathbb{R}^3$ is the translation vector component of the screw motion.
    *   Architecture: Depth 6, hidden size 128, skip connection at the 4th layer [S1].
    *   Activation: Softplus for density in $F$; standard activations for $W$ [S1].
3.  **Latent Codes:**
    *   $\omega_i \in \mathbb{R}^8$: Per-frame deformation latent code [S1].
    *   $\psi_i \in \mathbb{R}^8$: Per-frame appearance latent code [S1].
4.  **Positional Encoding:** $\gamma_\alpha(x)$ modulates input frequencies based on annealing parameter $\alpha$ [S1].

**Parameters:**
*   $N$: Total training iterations for annealing (e.g., 80,000) [S1].
*   $m$: Maximum number of frequency bands (e.g., 6) [S1].
*   $c$: Robust loss hyperparameter (0.03) [S1].
*   $\lambda, \mu$: Loss weighting coefficients (typically $10^{-3}$) [S1].
*   Samples per ray: 256 coarse and 256 fine samples for full HD [S1].

**Algorithm Steps:**

1.  **Initialization:**
    *   Initialize the canonical NeRF $F$ and deformation MLP $W$ [S1].
    *   Initialize the weights of the last layer of $W$ from $U(-10^{-5}, 10^{-5})$ to ensure the deformation starts near the identity transformation [S1].
    *   Initialize latent codes $\omega_i$ and $\psi_i$ via embedding layers [S1].

2.  **Deformation Field Formulation:**
    *   The deformation field $W$ outputs a screw axis $S = (r; v) \in \mathbb{R}^6$ [S1].
    *   Compute the rotation matrix $e^r \in SO(3)$ using Rodrigues' formula:
        $$ e^r = I + \frac{\sin \theta}{\theta}[r]_\times + \frac{1 - \cos \theta}{\theta^2}[r]_\times^2 $$
        where $\theta = \|r\|$, $\hat{r} = r/\|r\|$, and $[r]_\times$ is the cross-product matrix [S1].
    *   Compute the translation $p$ using the matrix $G$:
        $$ p = Gv, \quad G = I + \frac{1 - \cos \theta}{\theta^2}[r]_\times + \frac{\theta - \sin \theta}{\theta^3}[r]_\times^2 $$
    *   The transformed canonical point $x'$ is:
        $$ x' = e^S x = e^r x + p $$
        [S1]

3.  **Rendering (Forward Pass):**
    *   For each training image $i$:
        a. Cast rays in the observation frame and sample points $x$ along the rays [S1].
        b. Encode $x$ using the current positional encoding $\gamma_\alpha(x)$ [S1].
        c. Query the deformation MLP $W$ with $(\gamma_\alpha(x), \omega_i)$ to get $(r, v)$ [S1].
        d. Compute $x'$ using the formulas in Step 2 [S1].
        e. Query the canonical NeRF $F$ with $(x', d, \psi_i)$ to obtain density $\sigma$ and color $c$ [S1].
        f. Compute the rendered color $\hat{C}(\mathbf{r})$ using the standard volume rendering equation (see Parent article) [S1].

4.  **Loss Computation:**
    *   **Photometric Loss:** Mean Squared Error (MSE) between rendered color $\hat{C}$ and ground truth image color [S1].
    *   **Elastic Regularization:**
        a. Compute the Jacobian $J_T(x)$ of the deformation field $T$ with respect to $x$ using automatic differentiation [S1].
        b. Compute the Singular Value Decomposition (SVD) of $J_T(x) = U \Sigma V^T$ [S1].
        c. Compute the robust elastic loss using the Geman-McClure function:
            $$ L_{\text{elastic-r}}(x) = \rho(\|\log \Sigma\|_F, c) $$
            where $\rho(x, c) = \frac{2(x/c)^2}{(x/c)^2 + 4}$ and $c=0.03$ [S1].
            *Note: $\|\log \Sigma\|_F$ penalizes deviation of singular values from 1, treating expansion and contraction symmetrically [S1].*
        d. Weight the elastic loss at each sample by its contribution $w_i$ to the rendered view (from NeRF transmittance/density) to allow free deformation in empty space [S1].
    *   **Background Regularization (Optional):**
        If static 3D points $\{x_1, \dots, x_K\}$ are known (e.g., from SfM):
        $$ L_{\text{bg}} = \frac{1}{K} \sum_{k=1}^K \|T(x_k) - x_k\|^2 $$
        [S1]
    *   **Total Loss:**
        $$ L_{\text{total}} = L_{\text{rgb}} + \lambda L_{\text{elastic-r}} + \mu L_{\text{bg}} $$
        [S1]

5.  **Optimization:**
    *   Update parameters (NeRF weights, deformation MLP weights, latent codes) using Adam optimizer [S1].
    *   **Coarse-to-Fine Annealing:**
        a. Define the weight for frequency band $j$ as:
            $$ w_j(\alpha) = \frac{1 - \cos(\pi \cdot \text{clamp}(\alpha - j, 0, 1))}{2} $$
            where $\alpha \in [0, m]$ [S1].
        b. Linearly anneal $\alpha$ during training: $\alpha(t) = m \cdot \frac{t}{N}$, where $t$ is the current iteration and $N$ is the total iterations for annealing [S1].
        c. The positional encoding becomes $\gamma_\alpha(x) = (x, \dots, w_k(\alpha)\sin(2^k \pi x), w_k(\alpha)\cos(2^k \pi x), \dots)$ [S1].

**Pitfalls:**
*   Without regularization, deformation fields are prone to distortions, over-fitting, and local minima [S1].
*   Ambiguities between motion and scaling can lead to implausible results if not constrained [S1].
*   High-frequency positional encoding early in training can cause the model to fail to capture large motions (e.g., head rotation) by overfitting to minute details (e.g., smiles) [S1].

## Subtopics

### [Coarse-to-Fine Optimization](13448-coarse-to-fine-optimization.md)
Coarse-to-Fine Optimization is a training strategy for deformable neural radiance fields that stabilizes convergence by gradually increasing the model's capacity to represent high-frequency details. It addresses the difficulty of simultaneously optimizing complex geometry, texture, and non-rigid deformation from scratch by annealing the frequency components of the positional encoding used by the deformation network [13448-coarse-to-fine-optimization.md].

## Operating characteristics
*   **Time Complexity:** Training is computationally intensive. Full HD models require training on 8 V100 GPUs for a week, while half-resolution models take 16 hours [S1].
*   **Memory Complexity:** High memory usage due to storing intermediate states for backpropagation through the deformation field and volume rendering, typical of reverse-mode AD [44-radiance-field-optimization.md].
*   **Strengths:** Handles non-rigid deformation from casual, hand-held captures; resolves ambiguities between motion and scaling via elastic regularization; avoids local minima via coarse-to-fine annealing [S1].
*   **Limitations:** Prone to local minima without regularization; requires careful tuning of annealing parameters; deformation field can distort in empty space if not weighted properly [S1].
*   **When to use:** When reconstructing non-rigidly deforming scenes (e.g., humans) from casually captured video/images where explicit time correspondence or dense flow is unavailable [S1].
*   **When not to use:** For rigid scenes (standard NeRF is sufficient) or scenes with significant specular/transmission effects not handled by the appearance code [S1].

## Background
Nerfies was introduced by Mildenhall et al. in 2021 [S1]. It extends the original Neural Radiance Fields (NeRF) method [39] to handle dynamic, non-rigid scenes. The method was motivated by the challenge of capturing people with hand-held cameras, where non-rigidity and camera motion complicate reconstruction. It builds on the principle that a continuous deformation field can map observation-space points to a canonical template space [S1]. The approach draws inspiration from geometry processing and physical simulation, specifically elastic energy formulations, to regularize the deformation field [S1].

## Sources
[S1] Nerfies: Deformable Neural Radiance Fields, https://doi.org/10.1109/iccv48922.2021.00581, fetched 2026-10-26
