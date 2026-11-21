# Resonant Brane Splatting

Also called: Brane, Brane Primitive, Brane Splatting, Resonant Brane

## Overview
Resonant Brane Splatting (RBS) is a feedforward Arbitrary-Scale Super-Resolution (ASR) framework that replaces flat, single-color 2D Gaussians with 'Branes' [S1, unofficial]. Branes are expressive primitives that emit spatially varying colors by augmenting a Gaussian envelope with internal Gaussian-Hermite modes, allowing a single primitive to model local contrast, sharp edges, and complex textures without requiring dense overlapping splats [S1, unofficial]. Unlike [2DGH](link-to-2dgh.md), which uses Hermite functions to modulate spatial opacity while keeping color constant, RBS assigns an RGB-valued coefficient to each Hermite mode, making the emitted color vary spatially within the primitive footprint [S1, unofficial].

## How it works
**Inputs:** Low-resolution input image $I_{LR} \in \mathbb{R}^{H \times W \times 3}$, target scale factor $s \in \mathbb{R}^+$, and neural backbone parameters.
**Outputs:** Super-resolved image $I_{SR} \in \mathbb{R}^{H_{SR} \times W_{SR} \times 3}$, where $H_{SR} = \lfloor Hs \rfloor$ and $W_{SR} = \lfloor Ws \rfloor$.

**Data Structures:**
Each Brane primitive $k$ is defined by a state vector $\Phi_k$ predicted from a feature vector $f_k$. The layout is:
$$ \Phi_k = [\Delta x_k, \Delta y_k, \sigma_{x,k}, \sigma_{y,k}, \rho_k, \alpha_k, \mathbf{C}_{0,0,k}, \dots, \mathbf{C}_{N,M,k}] $$
Fields:
- $\Delta x_k, \Delta y_k$: Anchoring offsets (normalized).
- $\sigma_{x,k}, \sigma_{y,k}$: Anisotropic scale factors.
- $\rho_k$: Rotation angle.
- $\alpha_k$: Opacity.
- $\mathbf{C}_{n,m,k} \in \mathbb{R}^3$: Color coefficients for each mode $(n,m)$.

**Numbered Steps:**

1.  **Feature Extraction and Upsampling:**
    A neural backbone extracts a dense feature map $\hat{F} \in \mathbb{R}^{\hat{H} \times \hat{W} \times C}$ from $I_{LR}$. This map is optionally upsampled by factor $u$ to $F \in \mathbb{R}^{H' \times W' \times C}$, where $H' = u\hat{H}$ and $W' = u\hat{W}$. The number of primitives is $K = H'W'$ [S1, unofficial].

2.  **Prediction of Brane State:**
    For each feature vector $f_k$ in $F$, an MLP head predicts the full Brane state $\Phi_k$ [S1, unofficial].
    -   **Anchoring:** The $k$-th Brane is initially anchored at the center of its feature grid cell $(x_k^{\text{ref}}, y_k^{\text{ref}}) = (\frac{j + 0.5}{W'}, \frac{i + 0.5}{H'})$. An MLP head predicts continuous offsets $\Delta x_k, \Delta y_k$ for sub-pixel adaptation [S1, unofficial].
        $$ x_k = x_k^{\text{ref}} + \frac{\Delta x_k}{W'}, \quad y_k = y_k^{\text{ref}} + \frac{\Delta y_k}{H'} $$
    -   **Footprint and Opacity:** Additional heads predict anisotropic scales, rotation, and opacity [S1, unofficial]:
        $$ (\sigma_{x,k}, \sigma_{y,k}) = \text{sigmoid}(\text{MLP}_{\theta_3}(f_k)) $$
        $$ \rho_k = \pi \tanh(\text{MLP}_{\theta_4}(f_k)) $$
        $$ \alpha_k = \text{sigmoid}(\text{MLP}_{\theta_5}(f_k)) $$
    -   **Mode Colors:** Given maximum degrees $N$ and $M$, each Brane has $L = (N + 1)(M + 1)$ modes. A dedicated head predicts raw mode colors $\tilde{\mathbf{C}}_k = \text{MLP}_{\theta_2}(f_k) \in \mathbb{R}^{3L}$. Reshape into $L$ coefficients $\tilde{\mathbf{c}}_k^{n,m} \in \mathbb{R}^3$. Constrain base color to positive range and residuals to signed values [S1, unofficial]:
        $$ \mathbf{c}_k^{n,m} = \begin{cases} \text{sigmoid}(\tilde{\mathbf{c}}_k^{n,m}), & n=0, m=0 \\ \tanh(\tilde{\mathbf{c}}_k^{n,m}), & \text{otherwise} \end{cases} $$

3.  **Rasterization:**
    For each target pixel $(u, v)$ in $I_{SR}$:
    -   Project to normalized coordinates: $(u_{\text{norm}}, v_{\text{norm}}) = (\frac{u + 0.5}{W_{SR}}, \frac{v + 0.5}{H_{SR}})$ [S1, unofficial].
    -   Compute displacement from Brane center: $(\delta x_k, \delta y_k) = (u_{\text{norm}} - x_k, v_{\text{norm}} - y_k)$ [S1, unofficial].
    -   Map to local canonical coordinates via inverse rotation and scaling [S1, unofficial]:
        $$ \begin{pmatrix} x'_k \\ y'_k \end{pmatrix} = \begin{pmatrix} \frac{1}{\sigma_{x,k}} & 0 \\ 0 & \sigma_{y,k} \end{pmatrix} \begin{pmatrix} \cos \rho_k & \sin \rho_k \\ -\sin \rho_k & \cos \rho_k \end{pmatrix} \begin{pmatrix} \delta x_k \\ \delta y_k \end{pmatrix} $$
    -   Evaluate Brane emission function $\mathcal{B}_k(x'_k, y'_k)$:
        $$ \mathcal{B}_k(x'_k, y'_k) = \alpha_k E(x'_k, y'_k) \sum_{n=0}^N \sum_{m=0}^M \mathbf{c}_{n,m,k} \bar{H}_n(x'_k) \bar{H}_m(y'_k) $$
        where $E(x, y) = \exp(-\frac{x^2 + y^2}{2})$ is the Gaussian spatial envelope, and $\bar{H}_n(z)$ are normalized 1D Hermite modes [S1, unofficial].
    -   Accumulate color: $\mathbf{C}(u, v) = \sum_{k=1}^K \mathcal{B}_k(x'_k, y'_k)$ [S1, unofficial].

**Formulas:**
-   **Normalized Hermite Polynomial:** $\bar{H}_n(z) = \frac{1}{\sqrt{2^n n!}} H_n(z)$ [S1, unofficial].
-   **Physicist's Hermite Polynomial (Recursive):** $H_0(z) = 1$, $H_1(z) = 2z$, $H_n(z) = 2z H_{n-1}(z) - 2(n-1) H_{n-2}(z)$ for $n > 1$ [S1, unofficial].
-   **2D Mode:** $\Psi_{n,m}(x,y) = \bar{H}_n(x)\bar{H}_m(y)$ [S1, unofficial].

**Parameters:**
-   $N, M$: Maximum degrees of Gaussian-Hermite modes in x and y directions. Typical value: $N = M = 3$ [S1, unofficial].
-   $u$: Upsampling factor. Tested values: $\{1, 2, 4\}$ [S1, unofficial].
-   $d_{max}$: Bounding box fraction in normalized space for culling candidate Branes [S1, unofficial].
-   $R_{max}$: Canonical space boundary radius for culling, $R_{max} = \sqrt{2n_{max} + 1}$ where $n_{max} = \max(N, M)$ [S1, unofficial].

**Edge Cases and Pitfalls:**
-   **Numerical Instability:** Higher-order Hermite polynomials $H_n(z)$ grow rapidly. Using normalized Hermite polynomials $\bar{H}_n(z)$ removes dominant degree-dependent growth [S1, unofficial].
-   **Aliasing:** Training strictly on scales $s \in U(1, 4)$ fails to suppress oscillations in smooth regions at higher scales, causing aliasing artifacts [S1, unofficial].
-   **Culling:** Branes with $x'_k^2 + y'_k^2 > R_{max}^2$ are discarded as their visual impact is negligible [S1, unofficial].

## Subtopics

### Adaptive Mode Culling
A rasterization optimization that skips the evaluation of Hermite basis functions and color accumulation for Branes whose high-order mode coefficients are negligible, reducing computational load without affecting visual quality [S1, unofficial]. For each Brane, iterate through predicted Hermite coefficients $\mathbf{c}_{n,m,k}$. If the magnitude of an RGB coefficient for mode $(n, m)$ is less than $10^{-4}$, skip the Hermite polynomial evaluation and color accumulation for that mode [S1, unofficial]. This exploits sparsity induced by fine-tuning or sparsity loss, where unneeded coefficients are driven to zero [S1, unofficial].

### Sparsity Loss (Resonant Brane Splatting)
An explicit regularization term added to the training objective to enforce sparsity in the Hermite mode coefficients, preventing excessive vibration (aliasing) in smooth regions [S1, unofficial]. During training, compute the L1 norm of all Hermite coefficients where $n+m > 0$. Multiply the sum by sparsity weight $\lambda_{sparse}$ and $1/K$ (where $K$ is number of channels, RGB) [S1, unofficial]:
$$ L_{sparse} = \lambda_{sparse} \cdot \frac{1}{K} \sum_{k=1}^{K} \sum_{n+m>0} \|\mathbf{c}_{n,m,k}\|_1 $$
A tuned value of $\lambda_{sparse} = 0.0005$ yields results virtually identical to fine-tuning; an aggressive value of $0.1$ stifles details [S1, unofficial].

### Resonant Brane Splatting Regularization via Fine-Tuning
A training strategy to mitigate aliasing artifacts in smooth regions at large scales by fine-tuning the model on a wider range of scales, allowing it to naturally dampen high-order coefficients [S1, unofficial]. Fine-tune the model using a scale distribution $s \in U(1, 8)$ instead of the initial training range [S1, unofficial]. The model learns to dampen coefficients $\mathbf{c}_{n,m}$ to 0 for $n+m > 0$ in flat/smooth areas, recovering performance [S1, unofficial]. This achieves the same 'energy awareness' as explicit sparsity loss but implicitly through exposure to smoother targets [S1, unofficial].

## Operating characteristics
-   **Time Complexity:** Naive evaluation requires $O(K H_{SR} W_{SR})$ operations. Optimized via CUDA rasterizer with dual culling (bounding box $d_{max}$ and quantum turning point $R_{max}$) [S1, unofficial].
-   **Memory Complexity:** Not given in the sources.
-   **Measured Performance:** Rmax culling accelerates rendering up to 65% without quality loss [S1, unofficial]. RBS improves reconstruction quality over implicit and GS baselines, while achieving superior speed-quality trade-off than prior GS methods [S1, unofficial].
-   **Strengths:** Models high frequencies internally, reducing overlap and rasterization bottlenecks. Single primitive can model local contrast, edges, and multi-colored textures.
-   **Limitations:** Predicting excessive modes (e.g., $N=M=7$) in a single pass hampers generalization and causes smooth-region artifacts if not regularized [S1, unofficial].
-   **When to use:** Arbitrary-Scale Super-Resolution where high-frequency detail reconstruction is critical and computational efficiency is desired.
-   **When not to use:** When primitive sparsity is not an issue or when hardware lacks support for efficient CUDA rasterization with custom culling.

## Background
Resonant Brane Splatting (RBS) was introduced in 2026 as a feedforward ASR framework [S1, unofficial]. It builds on the expressive Gaussian-Hermite primitive but differs in appearance representation from 2DGH [54] by assigning RGB-valued coefficients to each Hermite mode [S1, unofficial]. It is used in ASR benchmarks, compared against implicit (Meta-SR, LIIF) and explicit (GaussianSR, GSASR) methods [S1, unofficial].

## Sources
[S1] Resonant Brane Splatting for Arbitrary-Scale Super-Resolution, https://doi.org/10.48550/arXiv.2606.29453, fetched 2026-10-30 (unofficial)
