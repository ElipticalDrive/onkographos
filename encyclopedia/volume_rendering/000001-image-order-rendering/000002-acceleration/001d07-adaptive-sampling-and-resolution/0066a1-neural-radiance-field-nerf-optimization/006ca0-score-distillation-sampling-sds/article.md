# Score Distillation Sampling (SDS)
Also called: LSDS, LSDS gradient, SDS Gradient Derivation, Score Distillation Sampling (SDS) Gradient Derivation

## Overview
Score Distillation Sampling (SDS) is an optimization technique that leverages a frozen, pretrained 2D text-to-image diffusion model as a loss function to optimize a parameterized 3D scene representation, such as a Neural Radiance Field (NeRF) [S1, unofficial][S2]. It solves the problem of 3D generation from 2D priors by avoiding backpropagation through the diffusion model itself, instead using the model's score function to guide the gradient updates of the 3D generator [S1, unofficial]. Unlike direct minimization of the diffusion training loss, which requires expensive and unstable backpropagation through the diffusion U-Net, SDS uses a "sticking-the-landing" gradient estimator that yields lower-variance updates [S1, unofficial]. It builds on [Neural Radiance Field (NeRF) Optimization](19666-neural-radiance-field-nerf-optimization.md) by optimizing a randomly initialized 3D model via gradient descent such that its 2D renderings from random angles achieve a low loss [S1, unofficial].

## How it works
**Inputs and Outputs**
- **Inputs**: A parameterized 3D scene generator $g$ (e.g., a NeRF MLP) with parameters $\theta$ [S1, unofficial]; a frozen pretrained 2D diffusion model (U-Net) $\epsilon_\phi$ trained on images [S1, unofficial]; a text prompt $y$ [S1, unofficial]; a weighting function $w(t)$ [S1, unofficial]; a classifier-free guidance scale $\omega$ [S1, unofficial]; and a differentiable volumetric renderer [S1, unofficial].
- **Outputs**: Optimized parameters $\theta$ for the 3D scene [S1, unofficial].

**Data Structures**
- **NeRF Representation**: The scene is parameterized by a Multi-Layer Perceptron (MLP) that maps 3D coordinates and viewing directions to volumetric density $\tau$ and albedo (color) $\rho$ [S1, unofficial]. Normals $n$ are computed from gradients of the density to enable shading with random lighting directions [S1, unofficial].
- **Diffusion Model**: A frozen U-Net $\epsilon_\phi(z_t, y, t)$ that predicts noise $\epsilon$ given a noisy latent $z_t$, text embedding $y$, and timestep $t$ [S1, unofficial].
- **Diffusion Coefficients**: For each timestep $t \in [0, 1]$, coefficients $\alpha_t$ and $\sigma_t$ are retrieved, satisfying $\alpha_t^2 + \sigma_t^2 = 1$ [S1, unofficial].

**Algorithm Steps**
1. Initialize the NeRF parameters $\theta$ randomly and initialize the optimizer state [S1, unofficial].
2. Load the pretrained diffusion model $\epsilon_\phi$ and freeze its weights [S1, unofficial].
3. For each optimization iteration:
    a. Sample a random camera position and viewing angle [S1, unofficial].
    b. Render the NeRF from the current camera to obtain an image $x = g(\theta)$ [S1, unofficial].
    c. Sample a timestep $t$ uniformly from $[0, 1]$ and noise $\epsilon \sim \mathcal{N}(0, I)$ with shape matching $x$ [S1, unofficial].
    d. Compute the noisy latent $z_t$ using the diffusion coefficients:
       $$ z_t = \alpha_t x + \sigma_t \epsilon $$
    e. Compute the guided noise prediction $\hat{\epsilon}_\phi$ using classifier-free guidance (CFG) with scale $\omega$:
       $$ \hat{\epsilon}_\phi(z_t; y, t) = (1 + \omega)\epsilon_\phi(z_t; y, t) - \omega\epsilon_\phi(z_t; t) $$
       where $\epsilon_\phi(z_t; y, t)$ is the conditional noise prediction and $\epsilon_\phi(z_t; t)$ is the unconditional noise prediction (conditioned on empty text) [S1, unofficial].
    f. Compute the gradient term using the "sticking-the-landing" estimator, which treats $\epsilon$ as a control variate for $\hat{\epsilon}_\phi$:
       $$ \text{grad\_term} = \text{stopgrad}[\hat{\epsilon}_\phi(z_t; y, t) - \epsilon] $$
       The `stopgrad` operator prevents backpropagation through the diffusion model [S1, unofficial].
    g. Compute the gradient of the SDS loss with respect to $\theta$ by backpropagating the `grad_term` through the differentiable renderer $g$:
       $$ \nabla_\theta L_{SDS} = \mathbb{E}_{t,\epsilon} \left[ w(t) \cdot \text{grad\_term} \cdot \frac{\partial x}{\partial \theta} \right] $$
       This is implemented by backpropagating the weighted residual vector through the rendering process to update $\theta$ [S1, unofficial].
    h. Update $\theta$ using the optimizer (e.g., Adam) [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import time

def stop_gradient(x):
    return x  # Prevents backpropagation through this operation

def random_uniform(a, b):
    return a + (b - a) * ((math.sin(time.time() * 1000) + 1) / 2)

def random_normal(shape):
    u1 = random_uniform(0.0, 1.0)
    u2 = random_uniform(0.0, 1.0)
    z = math.sqrt(-2.0 * math.log(u1)) * math.cos(2.0 * math.pi * u2)
    return [[z for _ in range(shape[1])] for _ in range(shape[0])]

def dot_product(a, b):
    return sum(a[i][j] * b[i][j] for i in range(len(a)) for j in range(len(a[0])))

def compute_gradient(params, loss_fn):
    # Returns the gradient of loss_fn(params) with respect to params via automatic differentiation
    pass

def score_distillation_sampling(generator, diffusion_model, text_prompt, num_iterations, guidance_scale, learning_rate):
    params = generator.init_parameters()
    for step in range(num_iterations):
        camera = sample_random_camera()
        x = generator.render(params, camera)
        t = random_uniform(0.0, 1.0)
        eps = random_normal(x.shape)
        alpha_t, sigma_t = diffusion_model.get_coeffs(t)
        z_t = [[alpha_t * x[i][j] + sigma_t * eps[i][j] for j in range(len(x[0]))] for i in range(len(x))]
        eps_cond = diffusion_model.predict_noise(z_t, text_prompt, t)
        eps_uncond = diffusion_model.predict_noise(z_t, "", t)
        eps_hat = [[(1 + guidance_scale) * eps_cond[i][j] - guidance_scale * eps_uncond[i][j] for j in range(len(eps_cond[0]))] for i in range(len(eps_cond))]
        residual = stop_gradient([[eps_hat[i][j] - eps[i][j] for j in range(len(eps_hat[0]))] for i in range(len(eps_hat))])
        grad = compute_gradient(params, lambda p: dot_product(residual, generator.render(p, camera)))
        params = [[params[i][j] - learning_rate * grad[i][j] for j in range(len(params[0]))] for i in range(len(params))]
    return params

def sample_random_camera():
    return {"yaw": random_uniform(0, 2 * math.pi), "pitch": random_uniform(-math.pi / 2, math.pi / 2), "dist": random_uniform(1, 5)}
```

**Parameters**
- **Guidance Scale $\omega$**: A large value is typically used for classifier-free guidance to improve sample fidelity [S1, unofficial]. Setting $\omega > 0$ alters the score function to prefer regions where the ratio of conditional to unconditional density is large, improving quality at the cost of diversity [S1, unofficial].
- **Weighting Function $w(t)$**: A function of the timestep $t$ that is relatively robust to choice [S1, unofficial].

**Edge Cases and Pitfalls**
- **U-Net Jacobian**: Directly minimizing the diffusion training loss $L_{Diff}$ requires backpropagating through the U-Net Jacobian $\frac{\partial z_t}{\partial x}$ (or $\frac{\partial \hat{\epsilon}}{\partial x}$) [S1, unofficial]. This term is expensive to compute and poorly conditioned for small noise levels because the U-Net approximates the scaled Hessian of the marginal density [S1, unofficial]. Omitting this term (via `stopgrad`) is essential for stability and efficiency [S1, unofficial].
- **Identity DIP Failure**: Minimizing $L_{Diff}$ directly with respect to generated datapoints $x=g(\theta)$ does not produce realistic samples even with an identity DIP, due to the unstable U-Net Jacobian term [S1, unofficial].
- **Noise as Control Variate**: Using $\epsilon$ in the gradient term $\hat{\epsilon}_\phi - \epsilon$ acts as a control variate, reducing variance and speeding up optimization compared to using $\hat{\epsilon}_\phi$ alone [S1, unofficial].

## Subtopics

### [Two-stage NeRF Rendering](925-two-stage-nerf-rendering.md)
Renders NeRF in two passes to improve quality; does not use diffusion priors. Choose when computational budget allows multiple passes per pixel.

### [Adaptive sampling and reconstruction](10469-adaptive-sampling-and-reconstruction.md)
Optimizes sampling density based on error; does not use generative models. Choose when reducing ray counts per pixel is the priority.

### [SteerNeRF](17152-steernerf.md)
Uses a different approach to steer NeRF generation, potentially without diffusion distillation. Choose when specific geometric or stylistic constraints are needed without text-to-3D priors.

## Operating characteristics
- **Time Complexity**: Not given in the sources.
- **Memory Complexity**: Not given in the sources.
- **Performance**: SDS enables high-fidelity 3D generation from text prompts without 3D training data [S1, unofficial][S2]. It is robust to the choice of weighting $w(t)$ [S1, unofficial].
- **Strengths**: Bypasses the need for 3D training data; leverages powerful 2D diffusion priors; avoids unstable backpropagation through the diffusion model; relatively simple to implement [S1, unofficial][S2].
- **Limitations**: Optimization can be slow due to the iterative nature of gradient descent through the renderer; quality depends heavily on the 2D diffusion model's capabilities [S1, unofficial].
- **When to use**: Text-to-3D or image-to-3D generation where 3D data is unavailable or expensive [S2].
- **When not to use**: Not given in the sources.

## Background
Score Distillation Sampling was introduced by Ben Poole, Ajay Jain, Jonathan T. Barron, and Ben Mildenhall in the paper "DreamFusion: Text-to-3D using 2D Diffusion" (arXiv:2209.14988v1, 29 Sep 2022) [S1, unofficial]. It was popularized by DreamFusion and is central to modern generative 3D pipelines [S2]. The method is based on probability density distillation and uses score functions from pretrained 2D diffusion models to guide the optimization of 3D scene representations like NeRFs [S1, unofficial]. It differs from identity DIP (Direct Image-space Optimization) by not requiring backpropagation through the diffusion model, making it more robust than methods that rely on carefully chosen timestep schedules [S1, unofficial].

## Sources
[S1] DreamFusion: Text-to-3D using 2D Diffusion, https://doi.org/10.48550/arxiv.2209.14988, fetched 2026-10-27 (unofficial)
[S2] Previous Differentiable Rendering Differentiable rendering is a framework that formulates the image synthesis process as a differentiable function of scene parameters, enabling gradient-based optimiza, https://inferensys.com/glossary/neural-radiance-fields-and-spatial-computing/neural-scene-representations/differentiable-rendering, fetched 2026-10-29
