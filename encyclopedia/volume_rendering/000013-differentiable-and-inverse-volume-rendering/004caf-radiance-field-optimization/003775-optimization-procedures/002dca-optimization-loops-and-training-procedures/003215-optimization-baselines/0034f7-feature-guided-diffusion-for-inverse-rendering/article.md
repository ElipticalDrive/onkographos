# Feature-Guided Diffusion for Inverse Rendering

Also called: Diffusion-CMA-ES Inverse Rendering, Diffusion-based CMA-ES, FGD, Feature-Guided Diffusion, Feature-Guided Diffusion for Non-Differentiable Inverse Rendering, Feature-Informed Diffusion Evolution

## Overview
Feature-Guided Diffusion for Inverse Rendering (FIDE) is a fully black-box optimization framework that solves non-differentiable inverse rendering problems by combining a diffusion-based inverse predictor with a Covariance Matrix Adaptation Evolution Strategy (CMA-ES) [S1, unofficial]. Unlike scalar-loss baselines like [Optimization Baselines](6675-optimization-baselines.md) or Bayesian optimization methods such as DiBO, FIDE extracts dense visual features from the target image to guide candidate proposal, thereby avoiding the information bottleneck of single-number loss values and the mode-averaging pitfalls of direct regression [S1, unofficial]. It is specifically designed to handle the multi-modal ambiguity inherent in inverse graphics, where multiple parameter sets can produce identical images, by approximating the inverse function as a multi-modal conditional distribution [S1, unofficial].

## How it works
FIDE operates within the standard inverse rendering loop defined by [Optimization Loops and Training Procedures](5622-optimization-loops-and-training-procedures.md), minimizing a photometric loss $\mathcal{L}$ between rendered color $\hat{C}(\mathbf{r})$ and ground-truth $C(\mathbf{r})$ [G]. The core mechanism alternates between an inverse prediction step using a diffusion model conditioned on image features and an evolution step using CMA-ES to refine the search space [S1, unofficial].

### Inputs and Outputs
- **Inputs**:
  - Black-box renderer $f: \mathbb{R}^d \rightarrow \mathcal{I}$, where $\mathcal{I}$ is the image space.
  - Target image $\mathbf{y}^* \in \mathcal{I}$.
  - Pretrained frozen feature extractor $E: \mathcal{I} \rightarrow \mathbb{R}^{h \times w \times c}$ (default: DINOv3).
  - Hyperparameters: population size $k=500$, iterations $N=50$, initial standard deviation $\sigma_0 = 1/6$.
- **Outputs**:
  - Optimized parameters $\boldsymbol{\mu}$ (mean of the final CMA-ES distribution).

### Data Structures
1.  **CMA-ES State**:
    -   Mean $\boldsymbol{\mu} \in \mathbb{R}^d$, initialized to $\mathbf{0}_d$ (or $0.5$ if parameters are normalized to $[0,1]$).
    -   Covariance matrix $\boldsymbol{\Sigma} \in \mathbb{R}^{d \times d}$, initialized to $\sigma_0 \mathbf{I}_{d \times d}$.
2.  **Diffusion Model $\tilde{f}_{\theta}^{-1}$**:
    -   Transformer-based architecture.
    -   **Input Tokens**:
        -   Image features $\mathbf{F} = E(\mathbf{y})$ flattened into tokens with learned positional embeddings.
        -   Noisy parameters $\mathbf{z} \in \mathbb{R}^d$ projected into tokens via a learned linear layer.
    -   **Processing**: Four self-attention blocks on parameter tokens, followed by cross-attention with image tokens.
    -   **Conditioning**: Feature-wise Linear Modulation (FiLM) injects timestep information (derived from $\boldsymbol{\Sigma}$) into each attention block.
    -   **Output**: Parameter tokens concatenated and projected back to $\mathbb{R}^d$ via a linear layer.
3.  **Training Data**:
    -   Dataset of pairs $(\mathbf{z}_i, \mathbf{y}_i, \boldsymbol{\Sigma})$ generated from CMA-ES samples.

### Numbered Steps
1.  **Initialization**: Set $\boldsymbol{\mu} = \mathbf{0}_d$, $\boldsymbol{\Sigma} = \sigma_0 \mathbf{I}_{d \times d}$. Initialize diffusion weights $\theta$. Normalize all task parameters to $[0, 1]$ and initialize at $0.5$ [S1, unofficial].
2.  **Iteration Loop** (for $i = 1$ to $N$):
    1.  **Sampling**: Draw $k$ candidates $\{\mathbf{x}_j\}_{j=0}^{k-1} \sim \mathcal{N}(\boldsymbol{\mu}, \boldsymbol{\Sigma})$ [S1, unofficial].
    2.  **Inverse Prediction (Partial Denoising)**:
        -   Select the first $k/2$ candidates.
        -   Compute the timestep proxy $\mathbf{s} = \sqrt{\text{diag}(\boldsymbol{\Sigma})}$ [S1, unofficial].
        -   Denoise these candidates using the diffusion model: $\{\mathbf{x}'_j\}_{j=0}^{k/2-1} = \tilde{f}_{\theta}^{-1}(\{\mathbf{x}_j\}_{j=0}^{k/2-1}, E(\mathbf{y}^*), \boldsymbol{\Sigma})$ [S1, unofficial].
        -   The denoising is performed by solving the reverse-time ODE using Euler's method with 3 steps and schedule $[0.3, 0.2, 0.1]$ [S1, unofficial].
        -   The remaining $k/2$ candidates are passed through unmodified to preserve diversity and prevent mode collapse [S1, unofficial].
        -   Combine denoised and undenoised candidates to form the new population $\{\mathbf{x}_j\}_{j=0}^{k-1}$ [S1, unofficial].
    3.  **Evaluation**: Render all candidates to obtain images $\{\mathbf{y}_j\}_{j=0}^{k-1} = f(\{\mathbf{x}_j\}_{j=0}^{k-1})$ [S1, unofficial].
    4.  **Feature Extraction**: Extract features $\{\mathbf{e}_j\}_{j=0}^{k-1} = E(\{\mathbf{y}_j\}_{j=0}^{k-1})$ [S1, unofficial].
    5.  **Online Training**: Train the diffusion model $\tilde{f}_{\theta}^{-1}$ on the current dataset $\{(\mathbf{x}_j, \mathbf{e}_j, \boldsymbol{\Sigma})\}$ for $T$ epochs (default 20) using the training objective:
        $$ \mathcal{L}_{diff} = \| \tilde{f}_{\theta}^{-1}(z, \mathbf{e}, \boldsymbol{\Sigma}) - x \|_2^2 $$
        where $z \sim \mathcal{N}(x, \sigma(s)^2)$ is the noisy version of $x$ [S1, unofficial]. Use Adam optimizer with learning rate $0.001$ [S1, unofficial].
    6.  **CMA-ES Update**:
        -   Compute cost $c_j = \mathcal{L}(f(\mathbf{x}_j), \mathbf{y}^*)$ for all $j$ [S1, unofficial].
        -   Rank candidates by cost.
        -   Update $\boldsymbol{\mu}$ and $\boldsymbol{\Sigma}$ using standard CMA-ES weighted update rules based on the ranked candidates [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def fide_inverse_rendering(renderer, target_image, d, k=500, N=50, sigma_0=1.0/6.0, 
                           epochs=20, lr=0.001, feature_extractor=None):
    """
    Implements Feature-Guided Diffusion for Inverse Rendering (FIDE).
    
    Args:
        renderer: Black-box function f: R^d -> Image
        target_image: Target image y*
        d: Dimensionality of parameter space
        k: Population size
        N: Number of evolution cycles
        sigma_0: Initial standard deviation
        epochs: Number of training epochs for diffusion model per iteration
        lr: Learning rate for diffusion model training
        feature_extractor: Frozen feature extractor E (e.g., DINOv3)
        
    Returns:
        mu: Optimized parameters (mean of final distribution)
    """
    # 1. Initialization
    mu = np.zeros(d)
    Sigma = sigma_0 * np.eye(d)
    
    # Initialize diffusion model weights theta
    # Note: In practice, this is a PyTorch/TensorFlow model instance
    diffusion_model = initialize_diffusion_model(d, feature_extractor)
    optimizer = Adam(diffusion_model.parameters(), lr=lr)
    
    # Pre-compute target features
    target_features = feature_extractor(target_image)
    
    for iteration in range(N):
        # 2. Sampling: Draw k candidates from N(mu, Sigma)
        candidates = np.random.multivariate_normal(mu, Sigma, size=k)
        
        # 3. Inverse Prediction (Partial Denoising)
        # Denoise only half of the samples to preserve diversity
        half_k = k // 2
        noisy_params = candidates[:half_k]
        
        # Compute timestep proxy from covariance diagonal
        # s = sqrt(diag(Sigma))
        timestep_proxy = np.sqrt(np.diag(Sigma))
        
        # Denoise using diffusion model conditioned on target features
        # x_denoised = f_theta^-1(noisy_params, target_features, Sigma)
        # This involves solving the reverse-time ODE with Euler's method (3 steps)
        denoised_candidates = diffusion_model.reverse_ode(
            noisy_params, 
            target_features, 
            timestep_proxy, 
            steps=3, 
            schedule=[0.3, 0.2, 0.1]
        )
        
        # Combine denoised and undenoised candidates
        candidates = np.concatenate([denoised_candidates, candidates[half_k:]])
        
        # 4. Evaluation: Render all candidates
        # y_i = f(x_i)
        rendered_images = np.array([renderer(x) for x in candidates])
        
        # 5. Feature Extraction
        # e_i = E(y_i)
        image_features = feature_extractor(rendered_images)
        
        # 6. Online Training: Train diffusion model on current dataset
        # Dataset: (noisy_params, image_features, Sigma) -> clean_params
        # Note: The training objective is || f_theta^-1(z, e, Sigma) - x ||^2
        # where z is a noisy version of x. In the online setting, we use the 
        # current candidates as 'clean' x and add noise to create z.
        
        # Generate noisy versions for training
        # z ~ N(x, sigma(s)^2)
        # For simplicity, we assume the diffusion model handles the noise injection
        # during the forward process if needed, or we explicitly add noise here.
        # The source states: "During training... starts from noisy parameters... 
        # conditioned on their renderings".
        
        # Prepare training data
        # x_clean = candidates
        # y_features = image_features
        # s = timestep_proxy (broadcasted or per-parameter)
        
        # Train for 'epochs'
        for epoch in range(epochs):
            # Create noisy inputs z from clean candidates x
            # z = x + noise, where noise variance depends on s
            noise = np.random.normal(0, 1, size=candidates.shape)
            # Assuming sigma(s) is derived from timestep_proxy
            # z = candidates + noise * sigma(timestep_proxy)
            z = candidates + noise * np.sqrt(timestep_proxy) # Simplified noise model
            
            # Forward pass and loss calculation
            # predicted_x = diffusion_model(z, y_features, timestep_proxy)
            predicted_x = diffusion_model(z, image_features, timestep_proxy)
            
            loss = np.mean((predicted_x - candidates) ** 2)
            
            # Backpropagate and update
            optimizer.zero_grad()
            loss.backward()
            optimizer.step()
            
        # 7. CMA-ES Update
        # Compute costs (using LPIPS or MSE as per source)
        costs = np.array([compute_loss(rendered_images[i], target_image) for i in range(k)])
        
        # Rank candidates
        sorted_indices = np.argsort(costs)
        ranked_candidates = candidates[sorted_indices]
        
        # Update mean and covariance using CMA-ES rules
        # This step adapts the search distribution to the local loss landscape
        mu, Sigma = update_cma_es(mu, Sigma, ranked_candidates, costs)
        
    return mu

def update_cma_es(mu, Sigma, ranked_candidates, costs):
    """
    Standard CMA-ES update step.
    Computes weighted average for mean and rank-mu update for covariance.
    """
    k = ranked_candidates.shape[0]
    # Weights: higher for top candidates
    weights = np.zeros(k)
    weights[:k//10] = 2.0
    weights[k//10:] = 0.5
    weights /= weights.sum()
    
    # Update mean
    new_mu = np.dot(weights, ranked_candidates)
    
    # Update covariance
    centered = ranked_candidates - new_mu
    new_Sigma = np.sum(weights[:, None, None] * (centered[:, :, None] * centered[:, None, :]), axis=0)
    
    return new_mu, new_Sigma

def compute_loss(rendered, target):
    """
    Computes LPIPS or MSE loss.
    """
    # Placeholder for actual LPIPS or MSE calculation
    return np.mean((rendered - target) ** 2)
```

### Parameters
-   **Population Size ($k$)**: 500 [S1, unofficial].
-   **Iterations ($N$)**: 50 [S1, unofficial].
-   **Initial Standard Deviation ($\sigma_0$)**: $1/6$, chosen because the parameter space $[0, 1]$ is within $3\sigma$ of the initialization at $0.5$ [S1, unofficial].
-   **Training Epochs**: 20 (default), varies from 4 to 20 [S1, unofficial].
-   **Learning Rate**: 0.001 [S1, unofficial].
-   **Reverse ODE Steps**: 3 steps with schedule $[0.3, 0.2, 0.1]$ [S1, unofficial].
-   **Feature Extractor**: DINOv3 (small) is default; VGG-19 is an alternative [S1, unofficial].

### Edge Cases and Pitfalls
-   **Mode Averaging**: Naïve inverse predictors return the numerical average of multiple valid solutions, which is invalid. FIDE avoids this by using a diffusion model to represent the multi-modal posterior [S1, unofficial].
-   **Mode Collapse**: Relying exclusively on denoising predictions risks the network overfitting to current estimates. FIDE mitigates this by denoising only half the candidates and passing the rest through unmodified [S1, unofficial].
-   **Non-Injectivity**: Inverse rendering is often non-injective (e.g., lighting/material ambiguity). FIDE handles this by sampling distinct valid solutions rather than collapsing to a mean [S1, unofficial].
-   **Scalability**: CMA-ES has quadratic memory complexity, and the diffusion model has quadratic self-attention over parameters. This limits the method to small-to-medium problems (up to ~100 parameters) [S1, unofficial].

## Operating characteristics
-   **Time Complexity**: The per-iteration runtime is roughly constant across dimensionality, but the number of render calls required to converge grows non-linearly with dimensionality [S1, unofficial].
-   **Memory Complexity**: Quadratic in $d$ due to CMA-ES covariance matrix and diffusion self-attention [S1, unofficial].
-   **Measured Performance**: On the Beads task (25,000 render calls, 224x224 pixels, LPIPS loss), FIDE takes 6-13 minutes on a desktop computer, compared to 4 hours for DiBO [S1, unofficial].
-   **Scaling**: Convergence is fast up to 60 parameters; reaching comparable loss at 80 parameters requires more than 50,000 render calls [S1, unofficial].
-   **Strengths**: Handles noise and unfavorable initialization better than differentiable methods; escapes local minima where gradient-based methods stall; avoids information bottleneck of scalar loss [S1, unofficial].
-   **Limitations**: Higher overhead than scalar-conditioned baselines; limited to ~100 parameters [S1, unofficial].
-   **When to Use**: Non-differentiable black-box renderers; problems with multi-modal ambiguity; when gradient-based methods fail due to plateaus or noise [S1, unofficial].
-   **When Not to Use**: Large-scale problems (>100 parameters); when computational overhead is prohibitive and scalar baselines are sufficient [S1, unofficial].

## Background
FIDE was introduced by Andrei-Timotei Ardelean, Michael Fischer, Tim Weyrich, and Tomáš Iser in 2026 [S1, unofficial]. It builds on the principles of CMA-ES [Hansen 2006] and diffusion models [Li and He 2025], addressing the limitations of black-box optimization methods that rely on scalar loss values. The method was validated across diverse inverse problems including camera pose estimation, robot arm estimation, path tracing, vector splines, and Voronoi shaders [S1, unofficial].

## Sources

[S1] Feature-Guided Diffusion for Non-Differentiable Inverse Rendering, https://doi.org/10.48550/arXiv.2607.17411, fetched 2026-10-29 (unofficial)
