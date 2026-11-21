# Single-Stage Diffusion NeRF Training

Also called: Joint optimization of NeRF and Diffusion Prior, SSDNeRF training

## Overview
Single-Stage Diffusion NeRF Training (SSDNeRF) is a method that jointly optimizes scene codes, a diffusion prior, and a NeRF decoder in a single training stage to minimize a combined loss of rendering fidelity and diffusion prior consistency [S1]. Unlike two-stage methods that ignore the prior term during autoencoder training—effectively setting the render-to-prior weight ratio to infinity and resulting in biased and noisy scene codes—this approach constrains codes with both terms simultaneously [S1]. It builds on [Diffusion-guided Optimization](13230-diffusion-guided-optimization.md) by integrating the prior directly into the latent space optimization loop.

## How it works
**Inputs and Outputs:**
*   **Inputs:** A dataset of images $\{y_{ij}^{gt}\}$ with corresponding rays $\{r_{ij}^{gt}\}$, where $i$ indexes the scene and $j$ indexes the ray (pixel).
*   **Outputs:** Optimized scene codes $\{x_i\}$, diffusion prior parameters $\phi$, and NeRF decoder parameters $\psi$.

**Data Structures:**
*   **Scene Codes ($x_i$):** Represented as triplanes with shape $\mathbb{R}^{3 \times 6 \times 128 \times 128}$ [S1].
*   **Diffusion Prior ($\phi$):** A U-Net based Gaussian diffusion model that takes a noisy code $x_i^{(t)}$ and timestep $t$ as input, outputting a denoised code $\hat{x}_i$ [S1].
*   **NeRF Decoder ($\psi$):** A decoder that maps scene codes and ray coordinates to pixel RGB values $y_{\psi}(x, r)$ [S1].

**Formulas:**
The total loss function is defined as [S1]:
$$
\mathcal{L} = \lambda_{\text{rend}} \mathcal{L}_{\text{rend}}(\{x_i\}, \psi) + \lambda_{\text{diff}} \mathcal{L}_{\text{diff}}(\{x_i\}, \phi)
$$
where $\mathcal{L}_{\text{rend}}$ is the rendering loss against known observations and $\mathcal{L}_{\text{diff}}$ is the approximate upper bound of the diffusion negative log-likelihood (NLL) [S1].

The empirical weighting factors are calculated as [S1]:
$$
\lambda_{\text{diff}} := \frac{c_{\text{diff}}}{\text{EMA} \|x_i\|_F^2}
$$
$$
\lambda_{\text{rend}} := \frac{c_{\text{rend}} (1 - e^{-0.1 N_v})}{N_v}
$$
where $c_{\text{diff}}$ and $c_{\text{rend}}$ are constant scales, $N_v$ is the number of available views, and $\text{EMA} \|x_i\|_F^2$ is the exponential moving average of the scene codes' Frobenius norms [S1]. The rendering weight calibration prevents the loss from scaling linearly with the number of rays under the ray independence assumption [S1].

**Algorithm:**
The training uses a nested loop structure with prior gradient caching to address the computational expense of the diffusion loss [S1].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Initialization
Initialize scene codes {x_i}, diffusion prior parameters phi, and NeRF decoder parameters psi
Initialize EMA accumulator for scene codes' Frobenius norms
Set constants c_diff, c_rend, K_out, K_in

for k_out in range(1, K_out + 1):
    # Outer loop: Sample a batch of scenes
    batch_scenes = SampleBatchScenes(B_sc)
    
    # Compute gradients for diffusion loss with respect to phi and {x_i}
    # L_diff is the diffusion prior loss
    g_phi, g_x_diff = ComputeGradients(L_diff, params=[phi, batch_scenes], weights=[lambda_diff])
    
    # Update diffusion parameters using Adam
    phi = AdamUpdate(phi, g_phi)
    
    # Cache the prior gradient for reuse in inner loop
    cached_g_x_diff = g_x_diff
    
    for k_in in range(1, K_in + 1):
        # Inner loop: Sample a batch of rays
        batch_rays = SampleBatchRays(B_ray)
        
        # Compute gradients for rendering loss with respect to {x_i}
        # L_rend is the rendering fidelity loss
        g_x_rend = ComputeGradients(L_rend, params=[batch_scenes], weights=[lambda_rend])
        
        # Combine cached diffusion gradient with current rendering gradient
        g_x_total = g_x_rend + cached_g_x_diff
        
        # Update scene codes using Adam with the total gradient
        batch_scenes = AdamUpdate(batch_scenes, g_x_total)
        
        # Update EMA of scene codes' Frobenius norms for dynamic weighting
        UpdateEMA(batch_scenes)
        
        # If this is the last inner iteration, update decoder parameters
        if k_in == K_in:
            g_psi = ComputeGradients(L_rend, params=[psi], weights=[lambda_rend])
            psi = AdamUpdate(psi, g_psi)
```

**Parameters:**
*   $K_{out}$: Number of outer iterations.
*   $K_{in}$: Number of inner iterations per outer loop.
*   $c_{\text{diff}}, c_{\text{rend}}$: Constant scales for empirical weighting [S1].
*   $N_v$: Number of available views [S1].

**Edge Cases and Pitfalls:**
*   **Computational Cost:** The diffusion loss $\mathcal{L}_{\text{diff}}$ is computationally expensive to evaluate compared to the rendering loss $\mathcal{L}_{\text{rend}}$, which reduces overall efficiency [S1]. The method mitigates this via prior gradient caching, reusing $\nabla_x \lambda_{\text{diff}} \mathcal{L}_{\text{diff}}$ for multiple Adam steps while refreshing rendering gradients each step [S1].
*   **Bias in Two-Stage Methods:** If the prior term is ignored during initial training (as in two-stage methods), the resulting scene codes become biased and noisy [S1].

## Operating characteristics
*   **Time Complexity:** The method is less efficient than rendering-only optimization due to the expensive diffusion loss evaluation. However, prior gradient caching reduces the number of diffusion passes required, accelerating reconstruction [S1].
*   **Memory Complexity:** Requires storing scene codes $\{x_i\}$, diffusion parameters $\phi$, and decoder parameters $\psi$ [S1].
*   **Strengths:** Produces more generalizable and less noisy scene codes compared to two-stage methods by constraining codes with both rendering and prior terms [S1].
*   **Limitations:** Higher computational cost per iteration compared to standard NeRF training [S1].
*   **When to use:** When high-quality 3D generation and reconstruction are required, and computational resources allow for joint optimization [S1].
*   **When not to use:** When real-time performance is critical and the overhead of diffusion prior evaluation is prohibitive [S1].

## Background
SSDNeRF was introduced in the paper "Single-Stage Diffusion NeRF: A Unified Approach to 3D Generation and Reconstruction" [S1]. It addresses the limitations of previous two-stage generative neural field methods, which ignore the prior term during autoencoder training, leading to biased and noisy scene codes [S1]. The method is related to score distillation techniques [S1].

## Sources
[S1] Single-Stage Diffusion NeRF: A Unified Approach to 3D Generation and Reconstruction, https://doi.org/10.1109/iccv51070.2023.00229, fetched 2026-10-29
