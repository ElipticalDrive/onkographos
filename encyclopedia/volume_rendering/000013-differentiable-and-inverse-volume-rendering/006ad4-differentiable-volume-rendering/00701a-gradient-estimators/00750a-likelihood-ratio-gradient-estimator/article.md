# Likelihood-Ratio Gradient Estimator
Also called: REINFORCE, Score Function Estimator
## Overview
The Likelihood-Ratio Gradient Estimator provides an unbiased gradient estimate for objectives involving expectations over discrete or stochastic variables by applying the likelihood-ratio identity [S1, unofficial]. It differs from Reparameterization Estimators [6499-reparameterization-estimators.md] by not requiring the stochastic variable to be differentiable with respect to parameters, making it suitable for discrete sampling or visibility terms, though it typically exhibits higher variance.
## How it works
**Inputs and Outputs:**
- Input: Parameters $\alpha$ defining a probability distribution $p(f; \alpha)$ over a discrete set of outcomes $\mathcal{F}$, and a loss function $L(\hat{C})$ dependent on the outcome.
- Output: An unbiased estimate of $\nabla_\alpha \mathbb{E}_{p(f)} [L(\hat{C})]$.

**Data Structures:**
- $\mathcal{F}$: The finite set of possible configurations (e.g., primitive visibility states).
- $p(f)$: Probability mass function for $f \in \mathcal{F}$.
- $L(\hat{C})$: Scalar loss value.

**Steps:**
1. Compute the expected loss:
   $$ \hat{L} = \mathbb{E}_{p(f)} [L(\hat{C})] = \sum_{f \in \mathcal{F}} L(\hat{C}) p(f) $$
2. Differentiate the expectation with respect to parameter $\alpha_{f'}$:
   $$ \frac{\partial \hat{L}}{\partial \alpha_{f'}} = \sum_{f \in \mathcal{F}} \frac{\partial L(\hat{C})}{\partial \alpha_{f'}} p(f) + \sum_{f \in \mathcal{F}} L(\hat{C}) \frac{\partial p(f)}{\partial \alpha_{f'}} $$
3. Apply the likelihood-ratio identity $\frac{\partial p(f)}{\partial \alpha_{f'}} = p(f) \frac{\partial \ln p(f)}{\partial \alpha_{f'}}$ to the second term:
   $$ \frac{\partial \hat{L}}{\partial \alpha_{f'}} = \mathbb{E}_{p(f)} \left[ \frac{\partial L(\hat{C})}{\partial \alpha_{f'}} \right] + \mathbb{E}_{p(f)} \left[ L(\hat{C}) \frac{\partial \ln p(f)}{\partial \alpha_{f'}} \right] $$
4. Combine into a single expectation:
   $$ \frac{\partial \hat{L}}{\partial \alpha_{f'}} = \mathbb{E}_{p(f)} \left[ \frac{\partial L(\hat{C})}{\partial \alpha_{f'}} + L(\hat{C}) \frac{\partial \ln p(f)}{\partial \alpha_{f'}} \right] $$
5. Estimate via Monte Carlo sampling: Draw samples $f \sim p(f)$ and compute the gradient estimate for each sample.

**Parameters:**
- Number of samples: Not given in the sources.

**Edge Cases:**
- If $p(f) = 0$, the log-probability is undefined; ensure support covers all relevant $f$.
- High variance in $L(\hat{C})$ leads to noisy gradients.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def likelihood_ratio_gradient(loss_fn, prob_fn, log_prob_fn, params, samples):
    """
    Estimate gradient of E[L] w.r.t params using likelihood-ratio identity.
    """
    grad_estimate = 0.0
    for f in samples:
        # Compute loss for this sample
        L = loss_fn(f)
        
        # Compute gradient of loss w.r.t params (if differentiable)
        # If L does not depend on params directly, this term is zero
        dL_d_params = compute_gradient_of_loss(f, params)
        
        # Compute score function: gradient of log probability
        score = log_prob_fn(f, params)
        
        # Accumulate: (dL/dalpha + L * score)
        grad_estimate += dL_d_params + L * score
        
    # Average over samples
    grad_estimate /= len(samples)
    return grad_estimate
```
## Subtopics

### [Perturbed Optimizer Gradient Estimation (Gaussian)](23617-perturbed-optimizer-gradient-estimation-gaussian.md)
This method estimates the gradient of a differentiable renderer's output with respect to its parameters $\theta$ by using noise drawn from a standard Gaussian distribution to perturb the optimizer's solution. It serves as the primary variant of the Likelihood-Ratio Gradient Estimator for continuous noise distributions, offering a specific variance profile compared to heavy-tailed alternatives like the Cauchy variant. It builds on [Likelihood-Ratio Gradient Estimator](23160-likelihood-ratio-gradient-estimator.md).

## Operating characteristics
- **Time Complexity:** Linear in the number of samples and the cost of evaluating $L(\hat{C})$ and $\nabla_\alpha \ln p(f)$.
- **Memory Complexity:** Linear in the number of samples stored for the gradient estimate.
- **Performance:** Unbiased but high variance, especially for discrete variables or sparse probabilities. Requires many samples for stable convergence.
- **Strengths:** Applicable to non-differentiable distributions and discrete variables.
- **Limitations:** High variance; sensitive to the scale of $L(\hat{C})$.
- **When to use:** When reparameterization is not possible (e.g., discrete sampling, visibility).
- **When not to use:** When low-variance gradients are critical and reparameterization is available.
## Background
The likelihood-ratio identity is a classical result in statistics. It is applied in differentiable rendering to handle visibility and discrete primitive selection, as seen in DiffSoup [S1, unofficial].
## Sources
[S1] DiffSoup: Direct Differentiable Rasterization of Triangle Soup for Extreme Radiance Field Simplification, https://doi.org/10.48550/arXiv.2603.27151, fetched 2026-10-30 (unofficial)
