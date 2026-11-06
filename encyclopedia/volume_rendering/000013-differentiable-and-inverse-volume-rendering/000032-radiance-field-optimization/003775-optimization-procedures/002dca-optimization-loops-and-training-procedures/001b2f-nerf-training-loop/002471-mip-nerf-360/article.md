# Mip-NeRF 360

Also called: Mip-NeRF 360: Unbounded Anti-Aliased Neural Radiance Fields

## Overview
Mip-NeRF 360 is a method for synthesizing photorealistic views of unbounded, real-world scenes where the camera may rotate 360 degrees and content exists at arbitrary distances. It solves the problems of aliasing, training instability, and geometric ambiguity inherent in standard NeRF when applied to large-scale environments. It sets itself apart from its parent [NeRF Training Loop](970-nerf-training-loop.md) by integrating a non-linear scene parameterization, online distillation between two MLPs, and a distortion-based regularizer, achieving a 57% reduction in mean squared error compared to standard mip-NeRF [S1].

## How it works
Mip-NeRF 360 extends the mip-NeRF framework by addressing three core challenges of unbounded scenes: scale disparity between near and far objects, the high capacity required for intricate details, and the ambiguity of geometry in open space. The method employs a two-MLP architecture trained via online distillation, a non-linear coordinate parameterization, and a continuous distortion loss.

### Inputs and Outputs
*   **Inputs:** A set of training images with known camera poses and intrinsics.
*   **Outputs:** A rendered pixel color $\mathbf{C}(\mathbf{r})$ for any query ray $\mathbf{r}$, and the trained model parameters $\Theta_{\text{NeRF}}$ and $\Theta_{\text{prop}}$.

### Data Structures
The method utilizes two Multi-Layer Perceptrons (MLPs):
1.  **Proposal MLP:** Parameterized by $\Theta_{\text{prop}}$. It outputs only volumetric density $\tau$. It is smaller and evaluated more frequently.
2.  **NeRF MLP:** Parameterized by $\Theta_{\text{NeRF}}$. It outputs density $\tau$ and color $\mathbf{c}$. It is larger (approx. 15$\times$ the capacity of a standard mip-NeRF) and evaluated less frequently.

### Numbered Steps

1.  **Scene Parameterization:**
    Map 3D scene coordinates to a bounded domain using a non-linear parameterization. This allocates more MLP capacity to nearby content and less to distant content, handling the unbounded nature of the scene [S1]. The specific parameterization function is not detailed in the provided notes, but it replaces the standard linear coordinate input to the MLP [S1].

2.  **Coarse Sampling:**
    Generate a set of "coarse" distance samples $\mathbf{t}^c$ along the ray. These are sorted samples from a uniform distribution spanning the camera's near plane $t_n$ and far plane $t_f$:
    $$ t^c \sim \mathcal{U}[t_n, t_f], \quad \mathbf{t}^c = \text{sort}(\{t^c\}) $$
    During training, this sampling is stochastic; during evaluation, samples are evenly spaced [S1].

3.  **Proposal Network Evaluation:**
    Evaluate the Proposal MLP at the coarse distances $\mathbf{t}^c$ to predict densities. These densities are used to define a histogram for resampling.

4.  **Fine Sampling (Online Distillation):**
    Resample "fine" distances $\mathbf{t}^f$ from the histogram defined by the coarse distances $\mathbf{t}^c$ and the coarse weights $\mathbf{w}^c$ (derived from the proposal MLP's densities) using inverse transform sampling:
    $$ \mathbf{t}^f \sim \text{hist}(\mathbf{t}^c, \mathbf{w}^c), \quad \mathbf{t}^f = \text{sort}(\{t^f\}) $$
    This concentrates samples around scene content, improving efficiency [S1].

5.  **NeRF Network Evaluation:**
    Evaluate the NeRF MLP at the fine distances $\mathbf{t}^f$.
    *   **Integrated Positional Encoding:** For each interval $T_i$ in $\mathbf{t}^f$, compute the mean $\boldsymbol{\mu}$ and covariance $\boldsymbol{\Sigma}$ of the conical frustum. Featurize these using integrated positional encoding $\gamma$:
        $$ \gamma(\boldsymbol{\mu}, \boldsymbol{\Sigma}) = \left\{ \begin{bmatrix} \left[ \sin(2^\ell \boldsymbol{\mu}) \exp\left(-2^{2\ell-1} \operatorname{diag}(\boldsymbol{\Sigma})\right) \right] \\ \left[ \cos(2^\ell \boldsymbol{\mu}) \exp\left(-2^{2\ell-1} \operatorname{diag}(\boldsymbol{\Sigma})\right) \right] \end{bmatrix} \right\}_{\ell=0}^{L-1} $$
        where $L$ is the number of frequency bands [S1].
    *   **MLP Forward Pass:** Input the encoded features (and view direction $\mathbf{d}$) to the NeRF MLP:
        $$ (\tau_i, \mathbf{c}_i) = \text{MLP}(\gamma(\mathbf{r}(T_i)); \Theta_{\text{NeRF}}) $$
        where $\tau_i$ is density and $\mathbf{c}_i$ is color [S1].

6.  **Volume Rendering:**
    Approximate the volume rendering integral using numerical quadrature. Compute the alpha compositing weights $w_i$:
    $$ w_i = \left(1 - e^{-\tau_i(t_{i+1}-t_i)}\right)e^{-\sum_{i'} \tau_{i'}(t_{i'+1}-t_{i'})} $$
    where $t_i$ and $t_{i+1}$ are interval boundaries, and the sum is over all intervals $i'$. The final pixel color is:
    $$ \mathbf{C}(\mathbf{r}) = \sum w_i \mathbf{c}_i $$
    The weights $w_i$ are guaranteed to sum to $\le 1$ [S1].

7.  **Loss Computation and Optimization:**
    The model is trained by minimizing a weighted combination of coarse and fine reconstruction losses. The Proposal MLP is supervised by the histogram weights generated by the NeRF MLP (online distillation), while the NeRF MLP is supervised by the image reconstruction loss.
    $$ \mathcal{L} = \sum_{\mathbf{r} \in \mathcal{R}} \frac{1}{10} \mathcal{L}_{\text{recon}}(\mathbf{C}(\mathbf{r}, \mathbf{t}^c), \mathbf{C}^*(\mathbf{r})) + \mathcal{L}_{\text{recon}}(\mathbf{C}(\mathbf{r}, \mathbf{t}^f), \mathbf{C}^*(\mathbf{r})) $$
    where $\mathcal{L}_{\text{recon}}$ is mean squared error, $\mathbf{C}^*(\mathbf{r})$ is the ground truth color, and $\mathcal{R}$ is the set of training rays [S1].
    *Note: The sources mention a "novel distortion-based regularizer" and "online distillation" as key components, but the specific formula for the distortion loss is not provided in the text excerpts. The loss equation above represents the reconstruction component explicitly defined.* [S1].

### Parameters
*   **Model Capacity:** The total model capacity is approximately 15 times larger than standard mip-NeRF [S1].
*   **Training Time:** Increases by approximately 2 times compared to mip-NeRF [S1].
*   **Acceleration:** Accelerates training by 300% compared to NeRF in Detail's learned proposer approach [S1].

### Edge Cases and Pitfalls
*   **Unbounded Ambiguity:** Standard NeRF regularizers (e.g., density injection) are insufficient for unbounded scenes. The method requires the specific distortion-based regularizer to prevent "floaters" and handle geometric ambiguity [S1].
*   **Scale Disparity:** Without non-linear parameterization, the MLP capacity saturates on intricate details while failing to model distant objects correctly [S1].

## Subtopics

### [Zip-NeRF](3030-zip-nerf.md)
Replaces the large MLP with an Instant-NGP grid structure and introduces specific anti-aliasing mechanisms (multisampling, downweighting, and interlevel loss) to handle high-frequency details, solving aliasing artifacts in spatial rendering and loss functions during online distillation.

### [Distortion Loss with Power Transformation](4560-distortion-loss-with-power-transformation.md)
Modifies the standard distortion loss by applying a non-linear power transformation to metric distance before computing the penalty, aggressively suppressing near-camera overfitting and "floaters" by curving the distance space to resemble log-distance.

## Operating characteristics
*   **Performance:** Reduces mean squared error by 57% compared to mip-NeRF [S1].
*   **Efficiency:** Training time is approximately 2$\times$ that of mip-NeRF, but 300% faster than NeRF in Detail's learned proposer approach [S1].
*   **Strengths:** Capable of producing realistic synthesized views and detailed depth maps for highly intricate, unbounded real-world scenes where the camera rotates 360 degrees [S1].
*   **Limitations:** Requires significantly larger model capacity ($\sim$15$\times$ mip-NeRF) and more samples along each ray to accurately localize surfaces in large scenes [S1].

## Background
Mip-NeRF 360 was introduced as an extension of mip-NeRF to address the challenges of rendering unbounded scenes. Standard NeRF models input infinitesimally small 3D points, causing aliasing at varying resolutions, which mip-NeRF rectified by using conical frustums. However, mip-NeRF struggled with the unbalanced detail and scale of nearby vs. distant objects, slow training times, and artifacts from scene ambiguity. Mip-NeRF 360 introduces non-linear scene parameterization, online distillation, and a distortion-based regularizer to overcome these issues. It was demonstrated on a new dataset of unbounded real-world scenes [S1].

## Sources
[S1] Mip-NeRF 360: Unbounded Anti-Aliased Neural Radiance Fields, https://doi.org/10.1109/cvpr52688.2022.00539, fetched 2026-10-26
