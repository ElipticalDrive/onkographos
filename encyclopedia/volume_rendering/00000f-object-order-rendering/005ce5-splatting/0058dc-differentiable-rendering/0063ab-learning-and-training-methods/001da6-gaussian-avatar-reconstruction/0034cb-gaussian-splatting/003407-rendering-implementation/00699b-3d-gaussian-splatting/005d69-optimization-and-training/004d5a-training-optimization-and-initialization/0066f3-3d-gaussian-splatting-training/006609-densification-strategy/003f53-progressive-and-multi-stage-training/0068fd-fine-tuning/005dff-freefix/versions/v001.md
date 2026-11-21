# FreeFix

Also called: FreeFix: Boosting 3D Gaussian Splatting via Fine-Tuning-Free Diffusion Models

## Overview
FreeFix is a fine-tuning-free method to improve the rendering quality of extrapolated views in 3D Gaussian Splatting (3DGS) by leveraging pretrained 2D image diffusion models (IDMs) [S1]. It solves the problem of artifacts in rendered RGBs and inconsistency across frames that arise when using Video Diffusion Models (VDMs) or naive 2D IDMs, avoiding the computational burden of temporal attention and the loss of per-pixel guidance due to 3D VAE down-sampling [S1]. It sets itself apart from siblings like [ViewExtrapolator](11917-mcmc-based-3dgs-sgld-for-3dgs.md) or [Difix3D+](11917-mcmc-based-3dgs-sgld-for-3dgs.md) by using a 2D–3D interleaved refinement strategy with per-pixel confidence masks derived from Fisher information, rather than relying on rendered opacity or warp masks which fail to identify artifact regions [S1].

## How it works
**Inputs and Outputs:**
- **Input:** An initial 3DGS model $G_{init}$ reconstructed from a sparse or partial training set $S_{train} = \{(V^t_j, I^t_j)\}$, and a continuous trajectory of $m$ extrapolated viewpoints $T_{ext} = \{V^e_0, V^e_1, ..., V^e_{m-1}\}$ [S1].
- **Output:** A refined 3DGS model $G_{refined}$ and a set of refined images for the extrapolated views [S1].

**Data Structures:**
- **3DGS Model $G$:** Contains $N$ Gaussians. Each Gaussian $i$ has position $\mu_i$, covariance $\Sigma_i = R_i S_i S_i^T R_i^T$ (where $R_i \in SO(3)$ and $S_i \in \mathbb{R}^{3 \times 3}$), opacity $\eta_i$, and color $c_i$ [S1].
- **Fixed Image Set $F_{i-1}$:** A set of previously refined images $\{(V^e_0, \tilde{I}^e_{0,f}), ..., (V^e_{i-1}, \tilde{I}^e_{i-1,f})\}$ [S1].
- **Affine Matrices:** For each generated view, two optimizable matrices $A_f \in \mathbb{R}^{3 \times 3}$ and $A_b \in \mathbb{R}^{3 \times 1}$ are maintained to correct color bias accumulation [S1].

**Numbered Steps:**

1.  **Initialization:** Start with $G_0 = G_{init}$ and an empty fixed image set $F_{-1} = \emptyset$ [S1].
2.  **Iterate through Extrapolated Views:** For each view index $i$ from $0$ to $m-1$:
    a.  **2D Refinement:**
        i.  Render the current view $V^e_i$ using the current 3DGS model $G_{i-1}$ to get the artifact-ridden image $\tilde{I}^e_i = \pi(V^e_i; G_{i-1})$ and the opacity map $M_\alpha = \pi(V^e_i; (G_{i-1}, \alpha))$ [S1].
        ii. Compute the uncertainty map $\bar{C}_{V^e_i; G_{i-1}}$ as the Hessian approximation: $\bar{C}_{V^e_i; G_{i-1}} = \nabla_G \pi(V^e_i; G_{i-1})^T \nabla_G \pi(V^e_i; G_{i-1})$ [S1].
        iii. Compute the certainty map $C^{c}_{V^e_i; G_{i-1}} = \exp[-\gamma_c \bar{C}_{V^e_i; G_{i-1}}]$ where $\gamma_c$ is a hyperparameter [S1].
        iv. Compute the confidence map $M^{c}_{V^e_i; G_{i-1}} = \pi(V^e_i; (G_{i-1}, C^{c}_{V^e_i; G_{i-1}})) \odot M_\alpha$ [S1].
        v. Encode $\tilde{I}^e_i$ to latent $x^r_0$. Resize $M^{c}_{V^e_i; G_{i-1}}$ to the latent shape to get $M_c$ [S1].
        vi. Perform diffusion denoising steps. For each step $t$, calculate the guided prediction $x^{t,g}_0$:
            - If in early stages (overall guidance active): $x^{t,g}_0 = M_c \odot x^r_0 + (1 - M_c) \odot (\beta M_\alpha x^r_0 + (1 - \beta M_\alpha) x^t_0)$ [S1].
            - Otherwise: $x^{t,g}_0 = M_c \odot x^r_0 + (1 - M_c) \odot x^t_0$ [S1].
            - Here $x^t_0$ is the standard denoiser prediction $x_t - \sigma_t F_\theta(x_t, t)$ [S1].
        vii. Derive the next step input $x_{t-1}$ from $x^{t,g}_0$ and $x_t$: $x_{t-1} = \frac{\sigma_{t-1} - \sigma_t}{\sigma_t} x_t - \frac{\sigma_{t-1}}{\sigma_t} x^{t,g}_0$ [S1].
        viii. Complete denoising to obtain the fixed image $\tilde{I}^e_{i,f}$ [S1].
    b.  **3D Refinement:**
        i.  Update $G_{i-1}$ to $G_i$ by optimizing on $S_{train}$, the previous fixed image set $F_{i-1}$, and the current pair $(V^e_i, \tilde{I}^e_{i,f})$ [S1].
        ii. For the current view, apply affine correction to the rendered image $\hat{I}^e_i$ before computing loss: $\hat{I}'^e_i = A_f \times \hat{I}^e_i + A_b$ [S1].
        iii. Compute loss $L = (1 - \lambda_s) ||\hat{I}'^e_i - \tilde{I}^e_{i,f}||_1 + \lambda_s \text{SSIM}(\hat{I}'^e_i, \tilde{I}^e_{i,f})$ [S1]. Use a smaller weight for generated views compared to $S_{train}$ to prevent distortion [S1].
        iv. Optimize $G_i$ and the affine matrices $A_f, A_b$ for this view [S1].
    c.  **Update State:** Add $(V^e_i, \tilde{I}^e_{i,f})$ to $F_{i-1}$ to form $F_i$ [S1].
3.  **Final Output:** Return $G_m$ as $G_{refined}$ [S1].

**Formulas:**
- **3DGS Rendering:** $\pi(V; G) = \sum_{i=1}^{N} \alpha_i c_i \prod_{j=1}^{i-1} (1 - \alpha_i)$, where $\alpha_i = \eta_i \exp[-\frac{1}{2} (p - \mu_i)^T \Sigma_i^{-1} (p - \mu_i)]$ [S1].
- **Uncertainty:** $\bar{C}_{V;G} = \nabla_G \pi(V; G)^T \nabla_G \pi(V; G)$ [S1].
- **Certainty:** $C^{c}_{V;G} = \exp[-\gamma_c \bar{C}_{V;G}]$ [S1].
- **Confidence Map:** $M^{c}_{V;G} = \pi(V; (G, C^{c}_{V;G})) \odot \pi(V; (G, \alpha))$ [S1].
- **Guided Prediction (Early Stage):** $x^{t,g}_0 = M_c \odot x^r_0 + (1 - M_c) \odot (\beta M_\alpha x^r_0 + (1 - \beta M_\alpha) x^t_0)$ [S1].
- **Guided Prediction (Later Stage):** $x^{t,g}_0 = M_c \odot x^r_0 + (1 - M_c) \odot x^t_0$ [S1].
- **Next Step Input:** $x_{t-1} = \frac{\sigma_{t-1} - \sigma_t}{\sigma_t} x_t - \frac{\sigma_{t-1}}{\sigma_t} x^{t,g}_0$ [S1].

**Parameters:**
- $\gamma_c$: Hyperparameter controlling sensitivity to artifacts in confidence map rendering. Range $[0, 1]$. Typical values: 0.001, 0.01, 0.1 [S1].
- $\beta$: Hyperparameter controlling the strength of overall guidance [S1].
- $\lambda_s$: Weight for SSIM term in the 3D refinement loss [S1].

**Pitfalls:**
- Using rendered opacity as guidance is ineffective because high-opacity regions can still contain severe artifacts [S1].
- Using uncertainty maps directly is numerically unstable ($[0, +\infty)$) and can be overwhelmed by low-opacity Gaussians with large uncertainties [S1].
- Direct application of IDMs without interleaved refinement causes blurriness and inconsistency, especially in low-confidence areas like sky/ground [S1].
- Color bias accumulation in generated views can lead to a blurry/over-gray effect if not corrected by affine matrices [S1].

## Operating characteristics
- **Time Complexity:** Quadratic in sequence length if using VDMs, but FreeFix uses IDMs which are linear per frame, though the interleaved 3D refinement adds optimization cost per view [S1].
- **Memory Complexity:** Requires storing the fixed image set $F_{i-1}$ and maintaining affine matrices for each generated view [S1].
- **Strengths:** Fine-tuning-free, preserves generalization, handles extrapolated views better than naive 2D DMs, avoids VDM computational burden [S1].
- **Limitations:** Relies on the quality of the initial 3DGS, iterative refinement is slower than single-pass methods [S1].
- **When to use:** When refining 3DGS from sparse/partial inputs for extrapolated views without access to fine-tuning resources or VDMs [S1].

## Background
FreeFix was introduced by Hongyu Zhou et al. from Zhejiang University, University of Maryland, and Huawei [S1]. It addresses the trade-off between generalization and fidelity in diffusion-based 3DGS refinement [S1]. The method was evaluated on LLFF, Mip-NeRF 360, and Waymo datasets using SDXL and Flux as backbone IDMs [S1].

## Sources
[S1] FreeFix: Boosting 3D Gaussian Splatting via Fine-Tuning-Free Diffusion Models, https://doi.org/10.1109/3dv69130.2026.00109, fetched 2026-10-30
