# Generative Latent Optimization


## Overview
Generative Latent Optimization (GLO) is a training procedure for generative radiance fields where a set of latent codes is jointly optimized with the generator network parameters to minimize a reconstruction loss against input images [S1]. It is distinguished from standard Generative Adversarial Networks (GANs) by the absence of a discriminator, effectively functioning as a discriminator-less GAN [S1]. Unlike methods that use latent codes merely for conditioning (e.g., NeRF-W), GLO explicitly treats the latent codes as learnable parameters to account for scene variability or to enable inversion [S1].

## How it works
**Inputs and Outputs:**
- **Inputs:** A dataset of $N$ images $\{I_1, \dots, I_N\}$ and a generator network $G$ (which maps latent codes and optional auxiliary inputs to radiance field predictions or rendered images).
- **Outputs:** Optimized generator parameters $\theta_G$ and a set of latent codes $\{z_1, \dots, z_N\}$.

**Data Structures:**
- **Latent Codes:** A set of vectors $Z = \{z_1, \dots, z_N\}$. Each $z_i$ is typically sampled from a prior distribution (e.g., standard normal $\mathcal{N}(0, I)$) at initialization [S1].
- **Generator Parameters:** $\theta_G$, representing the weights of the neural network (e.g., MLPs for density and color).

**Algorithm:**
The optimization minimizes the reconstruction error between the generated output and the target image for each pair. The objective function is:

$$
\min_{G, z_1, \dots, z_N} \sum_{i=1}^{N} \mathcal{L}(G(z_i, u_i), I_i)
$$

where:
- $\mathcal{L}$ is a reconstruction loss, such as $L_2$ or $L_1$ [S1].
- $G(z_i, u_i)$ is the output of the generator given latent code $z_i$ and auxiliary inputs $u_i$ [S1].
- $u_i$ represents other inputs not optimized over (e.g., camera poses, viewing directions, or positional encodings required by the specific radiance field architecture) [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generative_latent_optimization(images, generator, loss_fn, optimizer_config):
    """
    Jointly optimizes generator parameters and latent codes.
    
    Args:
        images: List of N target images {I_1, ..., I_N}.
        generator: Function G(z, u) that generates output from latent z and inputs u.
        loss_fn: Reconstruction loss function (e.g., L2).
        optimizer_config: Configuration for the optimizer (e.g., Adam).
        
    Returns:
        generator_params: Optimized parameters of G.
        latent_codes: Optimized latent codes {z_1, ..., z_N}.
    """
    N = len(images)
    
    # Initialize latent codes from a prior distribution (e.g., standard normal)
    # [G] Standard practice for GLO initialization.
    latent_codes = [torch.randn_like(generator.latent_dim) for _ in range(N)]
    
    # Initialize generator parameters
    generator_params = generator.get_parameters()
    
    # Create optimizer that updates both generator parameters and latent codes
    # [G] The optimizer must treat latent_codes as parameters to be updated.
    optimizer = Adam([generator_params] + latent_codes, **optimizer_config)
    
    # Optimization loop
    for step in range(max_steps):
        optimizer.zero_grad()
        
        total_loss = 0.0
        for i in range(N):
            # Get auxiliary inputs for image i (e.g., camera pose, viewing direction)
            # [G] u_i is specific to the scene representation (e.g., NeRF requires ray origins/directions).
            u_i = get_auxiliary_inputs(images[i])
            
            # Generate output
            generated_output = generator(latent_codes[i], u_i)
            
            # Compute reconstruction loss
            loss = loss_fn(generated_output, images[i])
            total_loss += loss
            
        # Backpropagate and update
        total_loss.backward()
        optimizer.step()
        
    return generator_params, latent_codes
```

**Parameters:**
- **Loss Function:** Typically $L_2$ or $L_1$ [S1].
- **Optimizer:** Adam is commonly used [S1].
- **Initialization:** Latent codes are usually sampled from a normal distribution [S1].

**Edge Cases and Pitfalls:**
- **Auxiliary Inputs:** In NeRF-based GLO, $u_i$ must include ray sampling information (positions and directions) [S1]. Failure to provide these leads to incorrect rendering.
- **Optimization Stability:** Without a discriminator, the model may collapse to a single mode if not regularized. Joint optimization can be sensitive to learning rates for $z_i$ vs. $\theta_G$ [S1].

## Subtopics

### Patch-based Latent Code Initialization
An optimization-based initialization procedure used to find a good starting latent code for finetuning a generative radiance field [S2]. It solves the problem of poor convergence by initializing latent codes via patch reconstruction rather than random sampling [S2].
- **Definition:** Optimizes a latent code to reconstruct random image patches from input views [S2].
- **Steps:**
    1. Sample 8 random patches of size $32 \times 32$ from all available input views [S2].
    2. Initialize the new latent code vector with zeros [S2].
    3. Optimize the latent code using Adam ($\beta_1=0.9, \beta_2=0.999$, learning rate $0.001$) for 1,500 steps [S2].
- **When to choose:** When finetuning a pre-trained prior model (e.g., for few-shot face synthesis) to ensure the latent code is in a valid region of the latent space [S2].

## Operating characteristics
- **Time Complexity:** $O(N \cdot T \cdot C)$, where $N$ is the number of images, $T$ is the number of optimization steps, and $C$ is the cost of one forward/backward pass through the generator [S1, S2].
- **Memory Complexity:** $O(N \cdot D + M)$, where $D$ is the latent dimension and $M$ is the memory for generator parameters [S1].
- **Performance:** GLO can be computationally expensive due to the joint optimization of $N$ latent codes [S1]. For example, inversion on a prior model may take 10 minutes on 4 TPUs for 1,500 steps [S2].
- **Strengths:** Simple to implement (no discriminator); effective for view synthesis and scene editing when latent codes are explicitly optimized [S1].
- **Limitations:** Lacks the stability guarantees of GANs; may suffer from mode collapse or poor generalization if not regularized [S1].

## Background
Generative Latent Optimization was introduced by Salimans et al. [S1]. It is described as a "Discriminator-less GAN" [S1]. In the context of NeRF, GLO has been used in models like NeRF-W and Edit-NeRF to optimize latent codes for appearance and scene editing [S1]. The method is particularly relevant in the 2020–2023 era for models that use latent codes to account for changeable aspects of a scene [S1].

## Sources
[S1] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
[S2] Preface: A Data-driven Volumetric Prior for Few-shot Ultra High-resolution Face Synthesis, https://doi.org/10.1109/iccv51070.2023.00315, fetched 2026-10-27
