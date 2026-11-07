# Feature-Guided Diffusion for Inverse Rendering

Also called: Diffusion-CMA-ES Inverse Rendering, Diffusion-based CMA-ES, FGD, Feature-Guided Diffusion, Feature-Guided Diffusion for Non-Differentiable Inverse Rendering, Feature-Informed Diffusion Evolution

## Overview
Feature-Guided Diffusion for Inverse Rendering (FIDE) is a fully black-box framework for solving non-differentiable inverse rendering problems by combining a diffusion-based inverse prediction model with Covariance Matrix Adaptation Evolution Strategy (CMA-ES) [S1, unofficial]. It solves the problem of finding parameters $\mathbf{x}$ that produce a target image $\mathbf{y}^*$ without requiring gradients from the renderer, treating the rendering function $f$ as an opaque black box. Unlike scalar-loss baselines like DiBO or DiffBBO, FIDE conditions the proposal distribution on dense visual features extracted from the target image, avoiding the information bottleneck of scalar loss values and mitigating mode averaging in ill-posed inverse problems [S1, unofficial].

## How it works
FIDE operates within the standard inverse rendering loop defined by [Optimization Loops and Training Procedures](5622-optimization-loops-and-training-procedures.md), minimizing a photometric loss $\mathcal{L}$ between rendered color $\hat{C}(\mathbf{r})$ and ground-truth $C(\mathbf{r})$ [G]. The core mechanism involves maintaining a search distribution $\mathcal{N}(\boldsymbol{\mu}, \boldsymbol{\Sigma})$ over the parameter space $\mathbb{R}^d$, where $\boldsymbol{\mu}$ is the mean and $\boldsymbol{\Sigma}$ is the anisotropic covariance matrix capturing correlations between parameters [S1, unofficial].

The method alternates between an inverse prediction step (using a diffusion model conditioned on image features) and an evolution step (using CMA-ES) to solve non-differentiable inverse problems [S1, unofficial].

### Inputs and Outputs
*   **Inputs**:
    *   Black-box renderer $f: \mathbb{R}^d \rightarrow \mathcal{I}$, where $\mathcal{I}$ is the image space.
    *   Target image $\mathbf{y}^* \in \mathcal{I}$.
    *   Pretrained frozen feature extractor $E: \mathcal{I} \rightarrow \mathbb{R}^{h \times w \times c}$ (default: DINOv3).
    *   Population size $k$ (default: 500).
    *   Number of evolution cycles $N$ (default: 50).
    *   Initial standard deviation $\sigma_0$ (default: $1/6$).
*   **Outputs**:
    *   Optimized parameter vector $\mathbf{x}^*$ (the mean $\boldsymbol{\mu}$ of the final CMA-ES distribution).

### Data Structures
1.  **CMA-ES State**:
    *   Mean $\boldsymbol{\mu} \in \mathbb{R}^d$.
    *   Covariance Matrix $\boldsymbol{\Sigma} \in \mathbb{R}^{d \times d}$.
2.  **Diffusion Denoiser $\tilde{f}_{\theta}^{-1}$**:
    *   Transformer-based architecture.
    *   **Input Tokens**:
        *   Image features $\mathbf{E}(\mathbf{y})$ flattened into tokens with learned positional embeddings.
        *   Noisy parameters $\mathbf{z}$ projected into tokens via a learned linear layer.
    *   **Processing**: Four self-attention blocks on parameter tokens, followed by cross-attention with image tokens.
    *   **Conditioning**: Feature-wise Linear Modulation (FiLM) injects timestep information into each attention block.
    *   **Output**: Parameter tokens concatenated and projected back to $\mathbb{R}^d$ via a linear layer.
3.  **Training Dataset**:
    *   Pairs of $(\mathbf{x}_i, \mathbf{y}_i)$ where $\mathbf{y}_i = f(\mathbf{x}_i)$, generated each iteration.

### Parameters
*   **Feature Extractor**: DINOv3 (small) is the default due to speed; VGG-19 is an alternative. A simple learnable one-layer patch embedding (16x16) can be used but yields slower convergence [S1, unofficial].
*   **Loss Function**: LPIPS features are used for the cost function $\mathcal{L}$ to avoid pixel-level objective issues [S1, unofficial].
*   **Training Epochs**: 20 epochs per iteration (default). More epochs improve convergence but increase runtime [S1, unofficial].
*   **Optimizer**: Adam with learning rate 0.001 [S1, unofficial].
*   **Reverse ODE Steps**: 3 steps with schedule $[0.3, 0.2, 0.1]$ [S1, unofficial].
*   **Parameter Normalization**: All task parameters are normalized to $[0, 1]$ and initialized at 0.5 [S1, unofficial].

### Numbered Steps
1.  **Initialize CMA-ES**: Set mean $\boldsymbol{\mu} = \mathbf{0}_d$ and covariance $\boldsymbol{\Sigma} = \sigma_0 \mathbf{I}_{d \times d}$, where $\sigma_0 = 1/6$ [S1, unofficial]. Initialize denoiser weights $\theta$.
2.  **Loop for $i = 1$ to $N$**:
    3.  **Sampling**: Draw $k$ candidates $\{\mathbf{x}_j\}_{j=0}^{k-1}$ from the current CMA-ES distribution $\mathcal{N}(\boldsymbol{\mu}, \boldsymbol{\Sigma})$ [S1, unofficial].
    4.  **Inverse Prediction (Partial Denoising)**:
        *   Extract target features $\mathbf{e}^* = E(\mathbf{y}^*)$ [S1, unofficial].
        *   Select the first half of candidates $\{\mathbf{x}_j\}_{j=0}^{k/2-1}$.
        *   Denoise these candidates using the trained denoiser $\tilde{f}_{\theta}^{-1}$ conditioned on $\mathbf{e}^*$ and the current covariance $\boldsymbol{\Sigma}$ (used as a timestep proxy/context) [S1, unofficial].
        *   The denoiser input is the noisy parameters and the covariance $\boldsymbol{\Sigma}$. Specifically, the timestep scalar $s$ is replaced by $\boldsymbol{\Sigma}$; in practice, using the diagonal of the matrix as input is sufficient [S1, unofficial].
        *   Generate refined candidates $\{\tilde{\mathbf{x}}_j\}$ by solving the reverse-time ODE with Euler's method (3 steps) [S1, unofficial].
        *   Replace the first half of the population with these denoised candidates. The remainder $\{\mathbf{x}_j\}_{j=k/2}^{k-1}$ are passed through unmodified to preserve diversity and avoid mode collapse [S1, unofficial].
    5.  **Evaluation**: Render all $k$ candidates using the black-box function $f$ to create a dataset of parameter-image pairs $\{(\mathbf{x}_j, \mathbf{y}_j)\}_{j=0}^{k-1}$ where $\mathbf{y}_j = f(\mathbf{x}_j)$ [S1, unofficial].
    6.  **Feature Extraction**: Extract visual features for all rendered images: $\{\mathbf{e}_j\}_{j=0}^{k-1} = E(\{\mathbf{y}_j\}_{j=0}^{k-1})$ [S1, unofficial].
    7.  **Online Training**: Update the diffusion network weights $\theta$ by training on the current dataset.
        *   **Training Objective**: $\| \tilde{f}_{\theta}^{-1}(z, f(x), s) - x \|_2^2$, where $z \sim \mathcal{N}(x, \sigma(s)^2)$ [S1, unofficial].
        *   **Noise Model**: Use the anisotropic Gaussian distribution derived by CMA-ES to create training samples, replacing the usual isotropic noise model. The CMA distribution is used for both training and sampling [S1, unofficial].
        *   **Timestep**: Determine timestep for each parameter using the square root of the diagonal of the CMA covariance matrix at the end of the diffusion process [S1, unofficial].
        *   Train for the specified number of epochs (default 20) starting from the weights of the previous iteration [S1, unofficial].
    8.  **CMA-ES Update**: Evaluate the cost $c_j = \mathcal{L}(f(\mathbf{x}_j), \mathbf{y}^*)$ for each candidate. Update $\boldsymbol{\mu}$ and $\boldsymbol{\Sigma}$ by computing a weighted average of the candidates, ranked by their cost. Higher weights are assigned to top-scoring candidates to adapt the search distribution to the local loss landscape [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def feature_guided_diffusion_inverse_rendering(
    renderer: callable,
    target_image: np.ndarray,
    feature_extractor: callable,
    k: int = 500,
    N: int = 50,
    sigma_0: float = 1.0/6.0,
    d: int,
    epochs: int = 20,
    lr: float = 0.001
):
    """
    Implements Feature-Informed Diffusion Evolution (FIDE).
    
    Args:
        renderer: Black-box function f: R^d -> Image
        target_image: Target image y*
        feature_extractor: Frozen feature extractor E (e.g., DINOv3)
        k: Population size
        N: Number of evolution cycles
        sigma_0: Initial standard deviation
        d: Dimensionality of parameter space
        epochs: Training epochs per iteration
        lr: Learning rate for diffusion model
        
    Returns:
        x_opt: Optimized parameters
    """
    
    # 1. Initialize CMA-ES
    mu = np.zeros(d)
    Sigma = sigma_0 * np.eye(d)
    
    # 2. Initialize Diffusion Denoiser
    # Architecture: Transformer with cross-attention
    # Input: Noisy params z, Image features E(y), Covariance Sigma (as timestep proxy)
    # Output: Predicted clean params x
    denoiser = initialize_denoiser() 
    
    # Extract target features once
    target_features = feature_extractor(target_image)
    
    # 3. Evolution Loop
    for i in range(N):
        # --- Sampling ---
        # Draw k candidates from N(mu, Sigma)
        candidates = np.random.multivariate_normal(mu, Sigma, size=k)
        
        # --- Inverse Prediction (Partial Denoising) ---
        # Denoise only the first half of samples to preserve diversity
        half_k = k // 2
        
        # Select first half
        noisy_params = candidates[:half_k]
        
        # Condition denoiser on target features and current covariance
        # The covariance Sigma acts as the timestep context
        denoised_params = denoise_with_diffusion(
            denoiser, 
            noisy_params, 
            target_features, 
            Sigma
        )
        
        # Replace first half with denoised candidates
        candidates[:half_k] = denoised_params
        # Remainder candidates remain unchanged
        
        # --- Evaluation ---
        # Render all samples
        rendered_images = np.array([renderer(x) for x in candidates])
        
        # Extract visual features for all rendered images
        rendered_features = np.array([feature_extractor(img) for img in rendered_images])
        
        # --- Online Training ---
        # Train diffusion network on current dataset
        # Dataset: (noisy_params, rendered_features, Sigma) -> clean_params
        # Note: In training, noise is sampled using the CMA distribution
        train_diffusion_model(
            denoiser,
            noisy_params, # These are the 'clean' targets for the denoiser if we view it as predicting x from z
            rendered_features,
            Sigma,
            epochs=epochs,
            lr=lr
        )
        
        # --- CMA-ES Update ---
        # Evaluate cost using LPIPS loss against target
        costs = np.array([compute_lpips_loss(img, target_image) for img in rendered_images])
        
        # Update mu and Sigma based on ranked costs
        mu, Sigma = update_cma_es(mu, Sigma, candidates, costs)
        
    return mu

def denoise_with_diffusion(denoiser, noisy_params, target_features, Sigma):
    """
    Generates candidates from the diffusion model by using a CMA sample 
    as a starting point and solving the reverse-time ODE with Euler's method.
    """
    # Schedule for reverse ODE steps
    schedule = [0.3, 0.2, 0.1]
    
    # Start with the noisy parameters (which are samples from CMA)
    current_params = noisy_params.copy()
    
    # Solve reverse-time ODE with Euler's method
    for step_t in schedule:
        # Predict clean parameters conditioned on target features and timestep
        # The timestep is derived from Sigma (diagonal sqrt)
        timestep_input = np.sqrt(np.diag(Sigma))
        
        predicted_clean = denoiser(current_params, target_features, timestep_input)
        
        # Euler step update
        # Note: Exact reverse ODE integration details depend on the specific 
        # diffusion formulation (x-prediction vs noise prediction).
        # Here we assume the denoiser outputs the prediction of x.
        current_params = predicted_clean
        
    return current_params

def train_diffusion_model(denoiser, clean_params, rendered_features, Sigma, epochs, lr):
    """
    Trains the diffusion network online.
    Objective: || f_theta^-1(z, f(x), s) - x ||_2^2
    where z ~ N(x, sigma(s)^2)
    """
    # In practice, we generate noisy versions of the clean_params
    # using the anisotropic Gaussian from CMA
    noisy_params = np.random.multivariate_normal(clean_params, Sigma)
    
    # Timestep input
    timestep_input = np.sqrt(np.diag(Sigma))
    
    # Compute loss
    predicted_params = denoiser(noisy_params, rendered_features, timestep_input)
    loss = np.mean((predicted_params - clean_params) ** 2)
    
    # Update weights using Adam optimizer
    # (Implementation detail: standard gradient descent step)
    update_weights(denoiser, loss, lr)
```

### Edge Cases and Pitfalls
*   **Mode Averaging**: A naïve predictor trained to regress parameters directly from features returns the numerical average of several possible solutions, which is invalid. Diffusion models alleviate this by representing the inverse function as a multi-modal distribution [S1, unofficial].
*   **Mode Collapse**: Relying exclusively on the denoising network’s predictions to seed the next iteration risks the network overfitting to its current estimates and collapsing prematurely to a single solution. Partial denoising (only half the candidates) addresses this [S1, unofficial].
*   **Non-Injectivity**: Direct inverse prediction fails on non-injective functions (e.g., lighting/shading ambiguity) because it collapses multiple valid parameter sets into a single mean prediction. The diffusion-based proposer handles this by sampling distinct valid solutions [S1, unofficial].
*   **Dimensionality Limit**: CMA-ES has quadratic memory complexity, limiting the method to small-to-medium sized optimization problems (up to 100 parameters). This is reinforced by the diffusion model's architecture, which includes quadratic self-attention over the parameters [S1, unofficial].

## Operating characteristics
*   **Time Complexity**: The per-iteration runtime stays roughly constant across dimensionality settings, but the number of render calls required to converge grows non-linearly with dimensionality. Convergence is fast up to 60 parameters; reaching comparable loss at 80 parameters requires more than 50,000 render calls [S1, unofficial].
*   **Memory Complexity**: Quadratic in the parameter dimension $d$ due to CMA-ES covariance matrix storage and the diffusion model's self-attention mechanism [S1, unofficial].
*   **Measured Performance**: On the Beads task (25,000 render calls, 224x224 pixels, LPIPS loss), the method takes 6-13 minutes (depending on epochs) on a desktop computer. This is slower than differentiable baselines like Mitsuba 3 (4.5 minutes) but avoids getting stuck in poor local optima [S1, unofficial].
*   **Strengths**: Robust against local minima and strong Monte Carlo noise. Handles non-differentiable black-box renderers. Escapes local minima where gradient-based methods stall.
*   **Limitations**: Higher overhead than scalar-conditioned methods like DiBO (which takes ~4 hours on the same task but fails to converge). Limited to ~100 parameters.
*   **When to Use**: Non-differentiable black-box renderers, ill-posed inverse problems with multiple modes, or when gradient estimation is unstable or unavailable.
*   **When Not to Use**: Large-scale parameter spaces (>100 dimensions) or when differentiable gradients are available and the problem is well-posed.

## Background
Feature-Informed Diffusion Evolution (FIDE) was introduced by Andrei-Timotei Ardelean, Michael Fischer, Tim Weyrich, and Tomáš Iser [S1, unofficial]. It addresses the limitations of existing black-box optimization methods (like CMA-ES and Bayesian optimization) which rely on scalar fitness values, discarding spatial information present in rendered images. The method builds on the principles of diffusion models for generative modeling and CMA-ES for derivative-free optimization, combining them to solve inverse rendering problems without differentiability requirements [S1, unofficial].

## Sources
[S1] Feature-Guided Diffusion for Non-Differentiable Inverse Rendering, https://doi.org/10.48550/arXiv.2607.17411, fetched 2026-10-29 (unofficial)
