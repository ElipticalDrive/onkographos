# DDPM Posterior Sampling with Adaptive Step Size

Also called: Joint Denoising and Material Optimization, Posterior Score Sampling

## Overview
This method performs inverse rendering by jointly denoising an environment map using a Diffusion Posterior Sampling (DPS) prior while simultaneously optimizing material parameters (base color, roughness, metallic) to fit observed input images [S1]. It differs from standard denoising by interleaving the diffusion sampling steps with gradient-based material optimization, using an adaptive step size parameter $\rho$ to balance the trade-off between the realism of the generative prior and the fidelity to the input observations [S1].

## How it works
**Inputs:**
*   Input images $\hat{y}_i$ for $i=1 \dots m$.
*   Corresponding camera poses $\mathbf{c}_i$.
*   A pre-trained DDPM model for seamless spherical environment maps (architecture from [Nichol and Dhariwal 2021]) [S1].
*   A differentiable renderer $R(\cdot)$ capable of path tracing environment maps [S1].

**Outputs:**
*   Optimized material parameters $\mathbf{k}_{arm}$ (base color, roughness, metallic texture maps).
*   A clean, seamless environment map $x_0$.

**Data Structures:**
*   Environment map $x_t$: A spherical texture representing the illumination at diffusion step $t$. Initialized via Mitsuba3 reconstruction [S1].
*   Material parameters $\mathbf{k}_{arm}$: Texture maps for base color, roughness, and metallic.
*   Optimizer: Adam [S1].

**Parameters:**
*   Learning rate: $1 \times 10^{-2}$ for all parameters during initialization [S1].
*   Denoising steps: 1000 [S1].
*   Refinement iterations: 200 [S1].
*   Hyper-parameter $\rho$: Balances generation realism vs. observation faithfulness.
    *   For $t > 500$: $\rho = 0.1$ [S1].
    *   For $t < 500$: $\rho$ varies between $0.1$ and $1$ [S1].

**Algorithm:**

1.  **Initialization:**
    *   Initialize environment map $x$ using Mitsuba3 reconstruction [S1].
    *   Initialize material parameters $\mathbf{k}_{arm}$ (base color, roughness, metallic) [S1].
    *   Run a preliminary optimization using the loss:
        $$ L_{init}(\mathbf{k}_{arm}, x) = \sum_{i=1}^{m} || \hat{y}_i - R(x, \mathbf{k}_{arm}, \mathbf{c}_i)||^2 $$
        using Adam with learning rate $1 \times 10^{-2}$ for metallic, roughness, environment map, and base color [S1].
    *   **Reset:** Keep the optimized roughness and metallic texture maps. Reset base color to zero. This serves as the initialization for the joint optimization [S1].

2.  **Joint Denoising and Material Optimization:**
    *   Iterate $t$ from $T$ down to $1$ (1000 steps) [S1].
    *   In each step, perform one step of DDPM posterior sampling on $x_t$ to obtain $x_{t-1}$, using the posterior score function defined in Eq. 13 of [S1]. The step size $\rho$ is selected based on $t$ [S1].
    *   Simultaneously, optimize $\mathbf{k}_{arm}$ (base color, roughness, metallic) by minimizing the denoising loss:
        $$ L_{denoise}(\mathbf{k}_{arm}, t) = \sum_{i=1}^{m} || \hat{y}_i - R(\hat{x}_t, \mathbf{k}_{arm}, \mathbf{c}_i)||^2 $$
        where $\hat{x}_t$ is the current environment map estimate at step $t$ [S1].
    *   *Note:* The sources describe this as "interleaved" optimization. The pseudocode below reflects the joint update per step [S1].

3.  **Refinement:**
    *   After generating the clean environment map $x_0$, refine the material parameters $\mathbf{k}_{arm}$ for 200 additional iterations using the refinement loss:
        $$ L_{refine}(\mathbf{k}_{arm}) = \sum_{i=1}^{m} || \hat{y}_i - R(x_0, \mathbf{k}_{arm}, \mathbf{c}_i)||^2 $$
        [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ddpm_posterior_sampling_adaptive(input_images, camera_poses, renderer, ddpm_model, num_steps=1000, num_refine=200):
    """
    Jointly denoise environment map and optimize materials using DDPM posterior sampling with adaptive step size.
    """
    # 1. Initialization
    # Initialize environment map x using Mitsuba3 reconstruction
    x = initialize_environment_map_mitsuba3()
    
    # Initialize material parameters (base color, roughness, metallic)
    k_arm = initialize_material_parameters()
    
    # Preliminary optimization to initialize roughness/metallic
    optimizer_init = Adam([x, k_arm], lr=1e-2)
    for _ in range(initialization_iterations): # Not specified, run until convergence
        loss_init = sum(norm(input_images[i] - renderer(x, k_arm, camera_poses[i]))**2 for i in range(len(input_images)))
        optimizer_init.zero_grad()
        loss_init.backward()
        optimizer_init.step()
        
    # Reset base color to zero, keep optimized roughness and metallic
    k_arm.base_color = 0.0
    # x remains as initialized by Mitsuba3 or the result of the init step
    
    # 2. Joint Denoising and Material Optimization
    optimizer_joint = Adam([x, k_arm], lr=1e-2) # Learning rate not explicitly specified for joint step, assume similar or default
    
    for t in range(num_steps, 0, -1):
        # Determine adaptive step size rho
        if t > 500:
            rho = 0.1
        else:
            # rho varies between 0.1 and 1 for t < 500
            # Exact schedule not given, assume linear interpolation or fixed value from context
            rho = interpolate_rho(t, start=0.1, end=1.0, total_steps=500) 
            
        # Perform one step of DDPM posterior sampling on x
        # This uses the posterior score function from Eq. 13 in [S1]
        # x is updated to x_{t-1}
        x = ddpm_posterior_step(x, t, ddpm_model, rho)
        
        # Optimize material parameters jointly with the denoising loss
        # Note: The source says "in every denoising step, we optimize... jointly with the loss"
        # This implies an inner optimization loop or a single gradient step per diffusion step.
        # Given the timing (630ms for material opt vs 290ms for denoising), it likely involves multiple inner steps or a full optimization pass.
        # We assume a single optimization step per diffusion step for the pseudocode structure, 
        # but in practice, this might be an inner loop.
        
        optimizer_joint.zero_grad()
        loss_denoise = sum(norm(input_images[i] - renderer(x, k_arm, camera_poses[i]))**2 for i in range(len(input_images)))
        loss_denoise.backward()
        optimizer_joint.step()
        
    # 3. Refinement
    # After generating clean environment map x_0 (which is x after the loop)
    optimizer_refine = Adam([k_arm], lr=1e-2) # Only optimize materials
    for _ in range(num_refine):
        optimizer_refine.zero_grad()
        loss_refine = sum(norm(input_images[i] - renderer(x, k_arm, camera_poses[i]))**2 for i in range(len(input_images)))
        loss_refine.backward()
        optimizer_refine.step()
        
    return x, k_arm

def interpolate_rho(t, start, end, total_steps):
    # t goes from 500 down to 1
    # Map t to [0, 1]
    progress = (500 - t) / 499.0
    return start + progress * (end - start)
```

**Edge Cases and Pitfalls:**
*   **Choice of $\rho$:** If $\rho$ is too small, sampling ignores input observations in favor of realism. If $\rho$ is too large, results are faithful to observations but have inferior realism [S1].
*   **Local Minima:** The adaptive schedule (low $\rho$ early, high $\rho$ late) prevents the material optimization from getting stuck in local minima by using the realistic prior to guide final noise reduction [S1].
*   **Initialization:** Resetting base color to zero while keeping optimized roughness/metallic is critical for faster convergence [S1].

## Operating characteristics
*   **Time Complexity:** Obtaining a single sample takes approximately 20 minutes [S1].
*   **Breakdown:** This includes 1000 iterations of interleaved denoising (290 ms per iteration) and material optimization (630 ms per iteration) [S1].
*   **Hardware:** Tested on four A100 GPUs for training the generative model; inference timing reported without specific hardware, but implied to be GPU-accelerated [S1].
*   **Strengths:** Produces realistic and seamless environment maps while fitting observations; the adaptive $\rho$ balances realism and fidelity [S1].
*   **Limitations:** Computationally expensive (20 mins per sample); requires a pre-trained DDPM model [S1].

## Background
*   **Origin:** Proposed in "Diffusion Posterior Illumination for Ambiguity-Aware Inverse Rendering" (2023) by Linjie Lyu et al. [S1].
*   **Context:** Part of a pipeline for inverse rendering that uses generative models to resolve ambiguities in material and illumination estimation [S1].
*   **Development:** Builds on Diffusion Posterior Sampling (DPS) techniques and uses a DDPM architecture from [Nichol and Dhariwal 2021] [S1].

## Sources
[S1] Diffusion Posterior Illumination for Ambiguity-Aware Inverse Rendering, https://doi.org/10.1145/3618357, fetched 2026-10-26
