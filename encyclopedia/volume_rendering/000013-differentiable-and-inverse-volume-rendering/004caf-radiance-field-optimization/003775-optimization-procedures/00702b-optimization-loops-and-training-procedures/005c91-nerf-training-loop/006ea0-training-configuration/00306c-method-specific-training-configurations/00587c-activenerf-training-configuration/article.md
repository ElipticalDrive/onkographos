# ActiveNeRF Training Configuration

Also called: Active Learning with Uncertainty Estimation, ActiveNeRF, ActiveNeRF optimization settings, NeRF training hyperparameters

## Overview
ActiveNeRF is a learning framework that models a 3D scene with a constrained input budget by incorporating uncertainty estimation into a NeRF model and using an active learning scheme to select new training samples based on information gain [S1]. It supplements the existing training set with newly captured samples to improve model performance upper bounds under limited training budgets [S1]. Unlike NeRF-W which models uncertainty for transient objects across images, ActiveNeRF models uncertainty for inherent noise in training data to guide active sampling [S1]. Unlike S-NeRF which uses variational inference and requires multiple identical inferences, ActiveNeRF follows the volume rendering procedure strictly and does not require multiple inferences for uncertainty [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Initial set of posed images, camera poses, and a budget for additional captures [S1].
*   **Outputs:** A trained Neural Radiance Field (NeRF) with uncertainty estimation, and a sequence of selected viewpoints for active capture [S1].

**Data Structures:**
The model consists of two parallel Multi-Layer Perceptrons (MLPs): a coarse model and a fine model [S1].
*   **Coarse Model:** Identical to vanilla NeRF [S1]. Outputs density $\sigma$ and feature vector $f$ from positional encoding $\gamma_x(x)$, and color $c$ from $f$ and direction encoding $\gamma_d(d)$ [S1].
    *   $[\sigma, f] = \text{MLP}_{\theta_1}(\gamma_x(x))$ [S1]
    *   $c = \text{MLP}_{\theta_2}(f, \gamma_d(d))$ [S1]
*   **Fine Model:** Includes an additional uncertainty branch [S1]. Outputs density $\sigma$, feature $f$, and variance $\beta^2(r(t))$ from coordinates [S1].
    *   $[\sigma, f, \beta^2(r(t))] = \text{MLP}_{\theta_1, \theta_3}(\gamma_x(r(t)))$ [S1]
    *   The mean color $\bar{c}(r(t))$ is output by a separate MLP: $\bar{c}(r(t)) = \text{MLP}_{\theta_2}(f, \gamma_d(d))$ [S1].
    *   The variance is regularized via softplus to ensure validity: $\bar{\beta}^2(r(t)) = \beta_0^2 + \log(1 + \exp(\beta^2(r(t))))$, where $\beta_0^2$ ensures a minimum variance for all locations [S1].

**Numbered Steps:**
1.  **Train ActiveNeRF Model:** Train the coarse and fine models using the initial observation data [S1]. The optimization minimizes the negative log-likelihood of rays in a batch $B$, combined with a regularization term for sparser density [S1].
    *   Loss: $\mathcal{L}^{uct} = \frac{1}{N}\sum_{i=1}^N \left( \frac{\|C(r_i) - \bar{C}(r_i)\|_2^2}{2\bar{\beta}^2(r_i)} + \frac{\log \bar{\beta}^2(r_i)}{2} + \frac{\lambda}{N_s}\sum_{j=1}^{N_s} \sigma_i(r_i(t_j)) \right)$ [S1]
    *   Total Loss: $\mathcal{L}^{uct}(C(r), \bar{C}^f(r)) + \frac{1}{N}\sum_{i=1}^N \|C(r_i) - \hat{C}^c(r_i)\|_2^2$ [S1].
2.  **Render Novel Views:** Render novel views from the trained model to estimate uncertainty [S1]. The rendered color $C(r)$ follows a Gaussian distribution $\mathcal{N}(\bar{C}(r), \bar{\beta}^2(r))$ where $\bar{C}(r) = \sum_{i=1}^{N_s} \alpha_i \bar{c}(r(t_i))$ and $\bar{\beta}^2(r) = \sum_{i=1}^{N_s} \alpha_i^2 \bar{\beta}^2(r(t_i))$ [S1].
3.  **Estimate New Viewpoint:** Evaluate candidate views to find the one that most reduces uncertainty [S1].
    *   Sample candidate views from a spherical space [S1].
    *   For efficiency, evaluate new inputs with lower resolution by sampling $H/r \times W/r$ rays instead of full $H \times W$ rays, approximating the influence of the whole image with $1/r^2$ time consumption [S1].
    *   Calculate the information gain as the subtraction of the variance from the prior to posterior distribution [S1].
    *   The acquisition function $\mathcal{A}(D_2)$ is defined as the sum of variance reductions for all sampled locations across all rays in an image:
        $$ \mathcal{A}(D_2) = \sum_{r_i \in D_2} \sum_{j=1}^{N_s} \left( \text{Var}^{(\text{pri})}(r_i(t_j)) - \text{Var}^{(\text{post})}(r_i(t_j)) \right) $$ [S1]
    *   The posterior variance for a sampled location $r_2(t_k)$ given a new ray $r_2$ is:
        $$ \text{Var}^{(\text{post})}(r_2(t_k)) = \left( \frac{1}{\bar{\beta}^2(r_2(t_k))} + \frac{\alpha_k^2}{\bar{\beta}^2(r_2)} \right)^{-1} = \frac{\bar{\beta}^2(r_2(t_k))\bar{\beta}^2(r_2)}{\alpha_k^2\bar{\beta}^2(r_2(t_k)) + \bar{\beta}^2(r_2)} $$ [S1]
    *   This is always less than the prior variance $\bar{\beta}^2(r_2(t_k))$ [S1].
    *   Choose the top-$k$ candidates that score highest in the acquisition function as the supplementary of the current training set [S1].
4.  **Capture and Add:** Capture a new perception and add it to the training set [S1]. Two approaches exist for incorporation:
    *   **Bayesian Estimation (BE):** Compute the posterior distribution of scene locations using ground-truth values from new inputs, avoiding collateral training [S1]. The posterior mean is used as the estimate:
        $$ \text{Mean} = \gamma \frac{C(r_2) - b(t_k)}{\alpha_k} + (1 - \gamma) \bar{c}(r_2(t_k)) $$
        where $\gamma = \frac{\alpha_k^2 \bar{\beta}^2(r_2(t_k))}{\alpha_k^2 \bar{\beta}^2(r_2(t_k)) + \bar{\beta}^2(r_2)}$ and $b(t_k)$ represents the contribution of unrelated sampled locations in the ray [S1].
    *   **Continuous Learning (CL):** Add captured inputs to the training set and tune the model based on the current one [S1]. The fraction of training rays from new images can be controlled to optimize newly observed regions [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import numpy as np

def softplus(x):
    return np.log(1 + np.exp(x))

class MLP:
    def __init__(self, layers):
        self.weights = [np.random.randn(layers[i], layers[i+1]) for i in range(len(layers)-1)]
        self.biases = [np.zeros(layers[i+1]) for i in range(len(layers)-1)]
    def forward(self, x):
        for w, b in zip(self.weights, self.biases):
            x = np.dot(x, w) + b
            x = np.maximum(0, x) # ReLU
        return x

def train_active_nerf(initial_images, num_iterations=200000, active_interval=40000, 
                      lr_init=5e-4, lr_min=5e-5, batch_size=1024, 
                      coarse_samples=64, fine_samples=128, lambda_reg=0.1):
    # Initialize models
    coarse_model = MLP([32, 256, 256, 2]) # Outputs sigma, f
    fine_model = MLP([32, 256, 256, 3])   # Outputs sigma, f, beta^2
    color_mlp = MLP([256, 256, 3])        # Outputs mean color c_bar
    
    # Optimizer state
    lr = lr_init
    coarse_grads = [np.zeros_like(w) for w in coarse_model.weights]
    fine_grads = [np.zeros_like(w) for w in fine_model.weights]
    color_grads = [np.zeros_like(w) for w in color_mlp.weights]
    
    training_set = list(initial_images)
    
    for iteration in range(1, num_iterations + 1):
        # Learning rate decay
        lr = max(lr_min, lr * 0.999)
        
        # Sample batch of rays
        rays = np.random.choice(training_set, batch_size, replace=False)
        
        # Forward pass: Coarse model
        coarse_outputs = coarse_model.forward(rays)
        coarse_loss = np.mean((rays.colors - coarse_outputs)**2)
        
        # Forward pass: Fine model with uncertainty
        fine_raw = fine_model.forward(rays)
        sigma = fine_raw[:, 0]
        f = fine_raw[:, 1]
        beta_sq_raw = fine_raw[:, 2]
        beta_sq = beta_sq_raw + np.log(1 + np.exp(beta_sq_raw)) # Softplus regularization
        
        c_bar = color_mlp.forward(f)
        
        # Rendered color and variance (simplified volume rendering step)
        alpha = 1 - np.exp(-sigma)
        C_bar = np.sum(alpha * c_bar, axis=0)
        beta_bar_sq = np.sum(alpha**2 * beta_sq, axis=0)
        
        # Uncertainty loss: NLL + density regularization
        nll = np.mean((rays.colors - C_bar)**2 / (2 * beta_bar_sq) + np.log(beta_bar_sq) / 2)
        reg = np.mean(np.sum(sigma, axis=1)) / fine_samples
        uncertainty_loss = nll + lambda_reg * reg
        
        total_loss = uncertainty_loss + coarse_loss
        
        # Backward pass (simplified gradient update)
        # In practice, automatic differentiation computes gradients here
        coarse_grads = [g + np.random.randn(*g.shape) * 0.01 for g in coarse_grads] # Placeholder for actual grad
        fine_grads = [g + np.random.randn(*g.shape) * 0.01 for g in fine_grads]
        color_grads = [g + np.random.randn(*g.shape) * 0.01 for g in color_grads]
        
        coarse_model.weights = [w - lr * g for w, g in zip(coarse_model.weights, coarse_grads)]
        fine_model.weights = [w - lr * g for w, g in zip(fine_model.weights, fine_grads)]
        color_mlp.weights = [w - lr * g for w, g in zip(color_mlp.weights, color_grads)]
        
        # Active learning step
        if iteration % active_interval == 0 and iteration > 0:
            # 1. Sample candidate views from spherical space
            candidates = np.random.uniform(-1, 1, size=(100, 3))
            
            gains = []
            for view in candidates:
                # 2. Render low-res rays for efficiency
                low_res_rays = render_rays(view, resolution_factor=2)
                
                # 3. Compute prior variance
                prior_var = compute_prior_variance(low_res_rays, fine_model, color_mlp)
                
                # 4. Compute posterior variance assuming new observation
                posterior_var = compute_posterior_variance(prior_var, low_res_rays)
                
                # 5. Information gain = prior - posterior
                gain = np.sum(prior_var - posterior_var)
                gains.append((view, gain))
            
            # 6. Select top-k candidates
            gains.sort(key=lambda x: x[1], reverse=True)
            selected_views = [g[0] for g in gains[:4]]
            
            # 7. Add to training set
            training_set.extend(selected_views)
            
    return fine_model, color_mlp

def render_rays(view, resolution_factor=2):
    # Simulates ray generation at reduced resolution
    return np.random.randn(100, 3) # Placeholder for actual ray geometry

def compute_prior_variance(rays, fine_model, color_mlp):
    raw = fine_model.forward(rays)
    sigma = raw[:, 0]
    f = raw[:, 1]
    beta_sq = raw[:, 2] + np.log(1 + np.exp(raw[:, 2]))
    alpha = 1 - np.exp(-sigma)
    return np.sum(alpha**2 * beta_sq, axis=0)

def compute_posterior_variance(prior_var, rays):
    # Posterior variance formula: (1/prior + alpha^2/prior_ray)^-1
    # Simplified for batch: assumes single new ray observation per location
    alpha = 1 - np.exp(-rays[:, 0]) # Placeholder for actual alpha
    return 1.0 / (1.0 / prior_var + alpha**2 / prior_var)
```

**Parameters:**
*   **Initial training set size:** Empirically adjusted (e.g., 4 or 2 initial observations) [S1].
*   **Active iteration interval:** New perceptions are captured every 40K iterations (e.g., at 40K, 80K, 120K, 160K) [S1].
*   **Sampling points:** 64 points per ray for the coarse model, 128 for the fine model [S1].
*   **Optimizer:** Adam with initial learning rate $5\times10^{-4}$, decaying exponentially to $5\times10^{-5}$ [S1].
*   **Batch size:** 1024 rays [S1].
*   **Hardware:** Single RTX2080Ti GPU [S1].
*   **$\lambda$:** Hyper-parameter controlling the strength of the regularization term in the uncertainty loss function [S1].

**Edge Cases and Pitfalls:**
*   Simply minimizing the negative log-likelihood objective leads to a suboptimal solution where weights $\alpha_i$ for different samples in a ray are driven closer [S1]. This results in an unexpectedly large fraction of non-zero density in the whole scene, causing blurs on the object’s surface [S1]. The regularization term is necessary to force sparser volume density [S1].
*   NeRF fails to generalize well from one or few input views and may collapse to trivial solutions in unobserved regions [S1]. The uncertainty estimation ensures robustness by providing larger variances in unobserved regions instead of collapsing [S1].

## Subtopics

### Uncertainty-Guided Novel View Selection Policy
A view selection policy that uses U-NeRF's color/depth uncertainties and E-SCRNet's epistemic uncertainties to select samples with the most information gain, improving data efficiency [S2]. It generates a novel view dataset $S_{new}$ by synthesizing more views within the scene boundary using U-NeRF, prunes invalid views from $S_{new}$ based on U-NeRF’s uncertainties (color and depth reliability), and selects views with high information gain ($S_{high}$) based on E-SCRNet’s epistemic uncertainties from the pruned set [S2]. Unlike random view synthesis which may contain large redundancy, this policy selects samples with the most information gain [S2].

## Operating characteristics
**Time and Memory Complexity:**
*   **Bayesian Estimation (BE):** Saves 75% of time consumption compared to Continuous Learning but shows inferior performance to CL, though it is competitive with heuristic approaches under CL [S1].
*   **Continuous Learning (CL):** Requires re-training or fine-tuning the model on the expanded dataset, which is computationally more expensive [S1].
*   **Acquisition Function:** Evaluating candidate views involves rendering and variance calculation [S1]. Using lower resolution ($H/r \times W/r$) reduces time consumption by a factor of $1/r^2$ [S1].

**Measured Performance:**
*   **Setting I (20 total observations, synthetic scenes):** ActiveNeRF-BE achieved 25.67 PSNR in 30 minutes, while ActiveNeRF-CL achieved 26.24 PSNR in 2.2 hours [S1].
*   **Setting II (10 total observations, synthetic scenes):** ActiveNeRF-BE achieved 18.25 PSNR in 16 minutes, while ActiveNeRF-CL achieved 20.01 PSNR in 1.1 hours [S1].
*   ActiveNeRF performs superior to or on par with the original NeRF in all settings, especially with scarce training data [S1].

**Strengths and Limitations:**
*   **Strengths:** Improves the upper bound of model performances by actively selecting new viewpoints, addressing blind spots in previous works [S1]. Provides robustness under few observations [S1]. BE allows for instant offline inference on resource-constrained edge devices [S1].
*   **Limitations:** BE shows inferior performance to CL, though it is competitive with heuristic approaches under CL [S1]. Requires an initial training phase before active selection can begin [S1].

**When to Use:**
*   Use when the training data budget is constrained and the goal is to maximize performance with minimal additional resources [S1].
*   Use BE when time and computation resources are bottlenecks or for edge devices [S1].
*   Use CL when time and computation resources are not bottlenecks and higher quality is desired [S1].

## Background
ActiveNeRF is presented in a paper by X. Pan et al., building on the NeRF framework and Bayesian neural network concepts for uncertainty estimation [S1]. It addresses the limitation of NeRF failing to generalize well to novel views with sparse observations [S1]. The work incorporates insights from active learning to supplement training data efficiently [S1].

## Sources
[S1] ActiveNeRF: Learning Where to See with Uncertainty Estimation, https://link.springer.com/chapter/10.1007/978-3-031-19827-4_14, fetched 2026-10-28
[S2] Leveraging Neural Radiance Fields for Uncertainty-Aware Visual Localization, https://doi.org/10.1109/icra57147.2024.10610126, fetched 2026-10-26
