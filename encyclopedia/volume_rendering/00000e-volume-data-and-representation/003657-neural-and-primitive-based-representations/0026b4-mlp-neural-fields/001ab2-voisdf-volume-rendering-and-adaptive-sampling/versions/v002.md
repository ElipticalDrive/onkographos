# VoISDF Volume Rendering and Adaptive Sampling

Also called: VolSDF, VolSDF Training, Volume Rendering of Neural Implicit Surfaces, Volume Rendering of Neural Implicit Surfaces training, Volume SDF

## Overview
VoISDF is a neural volume rendering method that models volume density as a transformed function of a learned Signed Distance Function (SDF) to improve geometry reconstruction and enable adaptive sampling [S3, unofficial]. Unlike prior methods that model geometry as a function of density or use random sampling, VoISDF extracts the surface as the zero level-set of the learned SDF and employs an error-bound-driven adaptive sampling algorithm to avoid salt-and-pepper artifacts without requiring a second network [S3, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of images with camera parameters. For each pixel $p$, the input is a triplet $(I_p, \mathbf{c}_p, \mathbf{v}_p)$, where $I_p \in \mathbb{R}^3$ is the intensity, $\mathbf{c}_p \in \mathbb{R}^3$ is the camera location, and $\mathbf{v}_p \in \mathbb{R}^3$ is the viewing direction [S3, unofficial].
*   **Outputs:** Rendered image intensity $I(\mathbf{c}, \mathbf{v})$ and reconstructed geometry (SDF).

**Data Structures:**
The system consists of two Multi-Layer Perceptrons (MLPs) and two scalar parameters:
1.  **Geometry Network ($f_\varphi$):** Approximates the SDF $d(\mathbf{x})$ and a global geometry feature vector $\mathbf{z}(\mathbf{x})$ of dimension 256.
    *   Input: Position $\mathbf{x} \in \mathbb{R}^3$ (with positional encoding).
    *   Output: $(d(\mathbf{x}), \mathbf{z}(\mathbf{x})) \in \mathbb{R}^{1+256}$ [S3, unofficial].
2.  **Radiance Network ($L_\psi$):** Represents the scene's radiance field.
    *   Input: Position $\mathbf{x}$, normal $\mathbf{n}$, view direction $\mathbf{v}$, and geometry feature $\mathbf{z}$.
    *   Output: Color $\mathbf{c} \in \mathbb{R}^3$ [S3, unofficial].
3.  **Parameters:** Two learnable positive scalars $\alpha, \beta \in \mathbb{R}$. In implementation, $\alpha = \beta^{-1}$ is often chosen [S3, unofficial].

**Formulas:**
*   **SDF Definition:** The SDF $d_\Omega(\mathbf{x})$ to the boundary surface $\mathcal{M}$ is defined as:
    $$d_\Omega(\mathbf{x}) = (-1)^{\mathbf{1}_\Omega(\mathbf{x})} \min_{\mathbf{y} \in \mathcal{M}} \|\mathbf{x} - \mathbf{y}\|$$
    where $\mathbf{1}_\Omega$ is the indicator function for the object space $\Omega$ [S3, unofficial].
*   **Density Conversion:** Volume density $\sigma(\mathbf{x})$ is defined using the Laplace CDF $\Psi_\beta$:
    $$\sigma(\mathbf{x}) = \alpha \Psi_\beta(-d_\Omega(\mathbf{x}))$$
    where $\Psi_\beta(s)$ is:
    $$\Psi_\beta(s) = \begin{cases} \frac{1}{2} \exp\left(-\frac{s}{\beta}\right) & \text{if } s \leq 0 \\ 1 - \frac{1}{2} \exp\left(-\frac{s}{\beta}\right) & \text{if } s > 0 \end{cases}$$
    $\alpha$ represents the constant density inside the volume, and $\beta$ controls the smoothing amount near the boundary [S3, unofficial].
*   **Rendering Integral:** The image intensity is the expected light along the ray:
    $$I(\mathbf{c}, \mathbf{v}) = \int_0^\infty L(\mathbf{x}(t), \mathbf{n}(t), \mathbf{v}) \tau(t) dt$$
    where $\mathbf{x}(t) = \mathbf{c} + t\mathbf{v}$, $\mathbf{n}(t) = \nabla_{\mathbf{x}} d_\Omega(\mathbf{x}(t))$, and $\tau(t) = \sigma(\mathbf{x}(t))T(t)$ is the PDF of opacity. Transmittance is $T(t) = \exp\left(-\int_0^t \sigma(\mathbf{x}(s)) ds\right)$ [S3, unofficial].
*   **Approximation:** The integral is approximated using the rectangle rule at discrete samples $\mathcal{S} = \{s_i\}_{i=1}^m$:
    $$\hat{I}_{\mathcal{S}}(\mathbf{c}, \mathbf{v}) = \sum_{i=1}^{m-1} \hat{\tau}_i L_i$$
    where $L_i = L(\mathbf{x}(s_i), \mathbf{n}(s_i), \mathbf{v})$ and $\hat{\tau}_i \approx \tau(s_i) \Delta s$ [S3, unofficial].

**Parameters:**
*   $\beta$: Typical value 0.001. Low $\beta$ makes SDF training easier but may reduce surface accuracy; high $\beta$ makes training harder but improves accuracy [S1, unofficial][S3, unofficial].
*   $\epsilon$: Error threshold for opacity approximation. Typical value 0.1 [S3, unofficial].
*   $n$: Initial uniform sample count. Default 128 [S3, unofficial].
*   $m$: Final sample count for rendering. Default 64 [S3, unofficial].
*   $M$: Maximum ray length [S3, unofficial].

**Algorithm: Adaptive Sampling**
The method avoids random sampling artifacts by using an error-bound-driven adaptive sampling algorithm [S3, unofficial].

1.  **Initialize:** Set uniform sampling $\mathcal{T} = \mathcal{T}_0$ with $n=128$ points: $t_k = (k-1)\frac{M}{n-1}$ for $k \in [n]$ [S3, unofficial].
2.  **Initialize Bound:** Initialize $\beta_+ > \beta$ such that the global error bound $B_{\mathcal{T},\beta_+} \leq \epsilon$. This is ensured by Lemma 2: $\beta \geq \frac{\alpha M^2}{4(n-1) \log(1 + \epsilon)}$ [S3, unofficial].
3.  **Iterate:** While $B_{\mathcal{T},\beta} > \epsilon$ and not `max_iter`:
    a.  **Upsample:** Add $n$ points to $\mathcal{T}$. The number of points added to each interval $[t_i, t_{i+1}]$ is proportional to its current error bound (Eq 14 in [S3, unofficial]) [S3, unofficial].
    b.  **Refine $\beta_+$:** If $B_{\mathcal{T},\beta_+} < \epsilon$, find $\beta_\star \in (\beta, \beta_+)$ such that $B_{\mathcal{T},\beta_\star} = \epsilon$ using bisection (max 10 iterations), then update $\beta_+ \leftarrow \beta_\star$ [S3, unofficial].
4.  **Estimate Opacity:** Estimate $\hat{O}$ using the final $\mathcal{T}$ and $\beta_+$ [S3, unofficial].
5.  **Final Samples:** Return $\mathcal{S}$ by generating $m=64$ fresh samples using inverse transform sampling with $\hat{O}^{-1}$ [S3, unofficial].

**Error Bounds:**
*   **Density Derivative Bound:** Within segment $[t_i, t_{i+1}]$:
    $$\left| \frac{d}{ds} \sigma(\mathbf{x}(s)) \right| \leq \frac{\alpha}{2\beta} \exp\left(-\frac{d_i^*}{\beta}\right)$$
    where $d_i^* = \min_{s \in [t_i, t_{i+1}], \mathbf{y} \notin B_i \cup B_{i+1}} \|\mathbf{x}(s) - \mathbf{y}\|$ and $B_i = \{\mathbf{x} \mid \|\mathbf{x} - \mathbf{x}(t_i)\| < |d_i|\}$ with $d_i = d_\Omega(\mathbf{x}(t_i))$ [S3, unofficial].
*   **Rectangle Rule Error:**
    $$|E(t)| \leq \widehat{E}(t) = \frac{\alpha}{4\beta} \left( \sum_{i=1}^{k-1} \delta_i^2 e^{-\frac{d_i^*}{\beta}} + (t - t_k)^2 e^{-\frac{d_k^*}{\beta}} \right)$$
    where $\delta_i = t_{i+1} - t_i$ and $\sigma_i = \sigma(\mathbf{x}(t_i))$ [S3, unofficial].
*   **Opacity Error Bound:**
    $$|O(t) - \widehat{O}(t)| \leq \exp\left(-\widehat{R}(t)\right) \left( \exp\left(\widehat{E}(t)\right) - 1 \right)$$
    where $\widehat{R}(t) = \sum_{i=1}^{k-1} \delta_i \sigma_i + (t - t_k) \sigma_k$ is the rectangle rule approximation of the density integral [S3, unofficial].
*   **Global Bound:**
    $$B_{\mathcal{T}, \beta} = \max_{k \in [n-1]} \left\{ \exp\left(-\widehat{R}(t_k)\right) \left( \exp(\widehat{E}(t_{k+1})) - 1 \right) \right\}$$
    with $\widehat{R}(t_0)=0$ [S3, unofficial].

**Pitfalls:**
*   Random sampling in NeRF causes salt and pepper artifacts; fixed uniform sampling causes different artifacts. VoISDF avoids these [S3, unofficial].
*   Using a second network for inverse sampling may result in density mismatch between the opacity network and the density network used for integration [S3, unofficial].
*   High $\beta$ biases the learned SDF differently than low $\beta$, affecting training ease and surface accuracy [S1, unofficial].

## Operating characteristics
*   **Complexity:** Time and memory complexity depend on the number of samples $n$ and $m$ and the MLP depth. The adaptive sampling adds overhead to compute error bounds but reduces the total samples needed for accurate rendering compared to uniform sampling.
*   **Performance:** On the BlendedMVS dataset, VolSDF improves Chamfer distance by a mean of 51.8% compared to NeRF++. VolSDF's mean PSNR is 27.08, compared to NeRF++'s 27.55 [S3, unofficial].
*   **Strengths:** Disentangles geometry and appearance better than NeRF; does not require object masks for training (unlike IDR); avoids sampling artifacts without a second network [S3, unofficial].
*   **Limitations:** Requires careful tuning of $\beta$; training can be sensitive to $\beta$ values [S1, unofficial].

## Background
VoISDF (VolSDF) was introduced in the paper "Volume Rendering of Neural Implicit Surfaces" [S3, unofficial]. It addresses the limitations of previous neural volume rendering methods that modeled geometry as a function of density, which often led to noisy reconstructions. By using an SDF-based density model, it provides a well-defined surface and an inductive bias for geometry. It is used for 3D reconstruction and novel view synthesis, particularly on datasets like DTU and BlendedMVS [S3, unofficial].

## Sources
[S1] omkarchittar/Neural_Surfaces, https://github.com/omkarchittar/Neural_Surfaces, fetched 2026-10-26 (unofficial)
[S2] learning3d/assignment3, https://github.com/learning3d/assignment3, fetched 2026-10-26
[S3] Volume Rendering of Neural Implicit Surfaces, https://proceedings.neurips.cc/paper_files/paper/2021/file/25e2a30f44898b9f3e978b1786dcd85c-Paper.pdf, fetched 2026-10-26 (unofficial)
