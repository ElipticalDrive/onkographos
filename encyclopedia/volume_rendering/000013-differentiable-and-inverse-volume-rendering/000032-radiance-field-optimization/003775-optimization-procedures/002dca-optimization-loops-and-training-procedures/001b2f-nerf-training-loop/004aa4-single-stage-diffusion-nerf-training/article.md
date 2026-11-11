# Single-Stage Diffusion NeRF Training

Also called: Joint optimization of NeRF and Diffusion Prior, SSDNeRF training

## Overview
Single-Stage Diffusion NeRF Training (SSDNeRF) is a method that jointly optimizes scene codes, a diffusion prior, and a NeRF decoder in a single training loop to minimize a combined loss of rendering fidelity and diffusion prior consistency [S1]. Unlike two-stage methods that ignore the prior term during autoencoder training—effectively setting the render-to-prior weight ratio to infinity and resulting in biased, noisy scene codes—this approach constrains codes with both terms simultaneously to prevent such bias [S1].

## How it works
**Inputs and Outputs**
*   **Inputs:** Ground truth observations $\{y_{ij}^{gt}\}$ and corresponding ray directions $\{r_{ij}^{gt}\}$ for a batch of scenes $i$ and rays $j$ [S1].
*   **Outputs:** Optimized scene codes $\{x_i\}$, diffusion prior parameters $\phi$, and NeRF decoder parameters $\psi$ [S1].

**Data Structures**
*   **Scene Codes ($x_i$):** Represented as triplanes with shape $\mathbb{R}^{3 \times 6 \times 128 \times 128}$ [S1].
*   **Diffusion Prior ($p_\phi$):** A Gaussian diffusion model (e.g., U-Net) that takes a noisy code $x_i^{(t)}$ and timestep $t$ as input, outputting a denoised code $\hat{x}_i$ [S1].
*   **NeRF Decoder ($p_\psi$):** A decoder that maps scene code $x_i$ and ray $r_{ij}$ to predicted pixel RGB values $y_{\psi}(x_i, r_{ij})$ [S1].

**Formulas**
The total training objective is defined as [S1]:
$$
L = \lambda_{\text{rend}} L_{\text{rend}}(\{x_i\}, \psi) + \lambda_{\text{diff}} L_{\text{diff}}(\{x_i\}, \phi)
$$
where $L_{\text{rend}}$ is the rendering loss against known observations and $L_{\text{diff}}$ is the approximate upper bound of the diffusion negative log-likelihood (NLL), also referred to as score distillation [S1].

The empirical weighting factors are calculated as [S1]:
$$
\lambda_{\text{diff}} = \frac{c_{\text{diff}}}{\text{EMA} \|x_i\|_F^2}
$$
$$
\lambda_{\text{rend}} = \frac{c_{\text{rend}} (1 - e^{-0.1 N_v})}{N_v}
$$
where $c_{\text{diff}}$ and $c_{\text{rend}}$ are constant scales, $N_v$ is the number of available views, and $\text{EMA} \|x_i\|_F^2$ is the exponential moving average of the scene codes' Frobenius norms [S1]. The rendering weight calibration prevents the loss from scaling linearly with the number of rays under the assumption of ray independence [S1].

**Parameters**
*   $K_{\text{out}}$: Number of outer iterations [S1].
*   $K_{\text{in}}$: Number of inner iterations per outer loop [S1].
*   $B_{\text{sc}}$: Batch of scenes [S1].
*   $B_{\text{ray}}$: Batch of rays [S1].
*   $c_{\text{rend}}, c_{\text{diff}}$: Constant scales for empirical weighting [S1].

**Pitfalls**
*   **Computational Cost:** The diffusion loss $L_{\text{diff}}$ is significantly more expensive to evaluate than the rendering loss $L_{\text{rend}}$, reducing overall efficiency [S1].
*   **Gradient Caching:** To mitigate cost, prior gradients are cached and reused across multiple Adam steps, while rendering gradients are refreshed every step [S1].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Algorithm 1: Single-stage diffusion NeRF training [S1]
# Input: Ground truth observations {y_ij_gt}, ray directions {r_ij_gt}
# Initialize scene codes {x_i}, diffusion prior parameters phi, NeRF decoder parameters psi

for kout in range(1, K_out + 1):
    # Sample a batch of scenes
    batch_scenes = SampleBatch(B_sc)
    
    # Compute gradients for diffusion loss with respect to phi and {x_i}
    # L_diff is the approximate upper bound of diffusion NLL [S1]
    grad_phi, grad_x_diff = ComputeGradients(
        variables=[phi, batch_scenes],
        loss=lambda_diff * L_diff(batch_scenes, phi)
    )
    
    # Update diffusion parameters phi using Adam
    phi = AdamUpdate(phi, grad_phi)
    
    # Cache the diffusion gradient for scene codes to reuse in inner loop
    cached_grad_x_diff = grad_x_diff
    
    for kin in range(1, K_in + 1):
        # Sample a batch of rays
        batch_rays = SampleBatch(B_ray)
        
        # Compute gradients for rendering loss with respect to {x_i}
        # L_rend is the rendering loss against known observations [S1]
        grad_x_rend = ComputeGradients(
            variables=[batch_scenes],
            loss=lambda_rend * L_rend(batch_scenes, psi, batch_rays)
        )
        
        # Combine cached diffusion gradient with current rendering gradient
        total_grad_x = grad_x_rend + cached_grad_x_diff
        
        # Update scene codes {x_i} using Adam optimizer with total gradient
        batch_scenes = AdamUpdate(batch_scenes, total_grad_x)
        
        # If this is the last inner iteration, update decoder parameters
        if kin == K_in:
            grad_psi = ComputeGradients(
                variables=[psi],
                loss=lambda_rend * L_rend(batch_scenes, psi, batch_rays)
            )
            psi = AdamUpdate(psi, grad_psi)
```

## Operating characteristics
*   **Efficiency:** The method is computationally expensive due to the diffusion loss evaluation. It employs prior gradient caching to accelerate reconstruction by refreshing rendering gradients every step while reusing prior gradients [S1].
*   **Quality:** Joint optimization prevents the biased and noisy scene codes produced by two-stage methods that ignore the prior during autoencoder training [S1].
*   **Generalization:** The empirical weighting mechanism adapts to the number of available views $N_v$, making it suitable for generalizable test-time NeRF reconstruction covering a wide spectrum from single-view to dense observations [S1].

## Background
Single-Stage Diffusion NeRF was introduced in the paper "Single-Stage Diffusion NeRF: A Unified Approach to 3D Generation and Reconstruction" [S1]. It addresses limitations in previous two-stage generative neural field methods [2, 13, 32, 47] and finetuning approaches [8, 61] by unifying the training of the diffusion prior and the NeRF decoder.

## Sources
[S1] Single-Stage Diffusion NeRF: A Unified Approach to 3D Generation and Reconstruction, https://doi.org/10.1109/iccv51070.2023.00229, fetched 2026-10-29
