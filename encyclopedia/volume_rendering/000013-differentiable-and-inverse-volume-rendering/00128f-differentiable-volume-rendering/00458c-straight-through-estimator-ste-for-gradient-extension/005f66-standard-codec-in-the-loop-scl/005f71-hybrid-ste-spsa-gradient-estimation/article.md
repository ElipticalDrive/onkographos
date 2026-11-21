# Hybrid STE+SPSA Gradient Estimation

Also called: STE+SPSA

## Overview
Hybrid STE+SPSA is a gradient estimation strategy for differentiable volume rendering that combines vanilla Straight-Through Estimator (STE) with Simultaneous Perturbation Stochastic Approximation (SPSA) to handle non-differentiable codecs within a training loop. It addresses the trade-off between gradient bias and computational cost by using vanilla STE during cached inference steps and switching to SPSA only when the codec cache is refreshed. Unlike vanilla STE, which provides biased but stable gradients, this hybrid approach attempts to reduce bias during cache updates, though it incurs significantly higher variance and training time [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Current parameter vector $\mathbf{P}$ (e.g., feature planes), codec cache state, perturbation scale $\epsilon$, and objective function $\mathcal{C}_q$ (e.g., reconstruction loss).
*   **Output:** Gradient estimate $\hat{g}(\mathbf{P})$ for the optimizer.

**Data Structures:**
*   **Codec Cache:** Stores the output of the non-differentiable codec for the current parameters to avoid redundant encoding/decoding.
*   **Perturbation Vector $\Delta$:** A vector of the same dimension as $\mathbf{P}$, where each element $\Delta_i$ is sampled independently from a symmetric Bernoulli distribution: $\Delta_i \in \{-1, 1\}$ with probability $0.5$ [S1].

**Steps:**
1.  **Check Cache Status:** Determine if the codec cache is active (valid for current $\mathbf{P}$) or needs refresh.
2.  **If Cache is Active:**
    *   Use vanilla STE. The gradient is computed by backpropagating through the decoder as if the codec were the identity function. The gradient is copied directly to the pre-codec feature plane [S1].
3.  **If Cache is Refreshed:**
    *   Sample a random perturbation vector $\Delta$ where $\Delta_i = \pm 1$ with probability $0.5$ [S1].
    *   Compute the perturbed parameters: $\mathbf{P}^+ = \mathbf{P} + \epsilon \Delta$ and $\mathbf{P}^- = \mathbf{P} - \epsilon \Delta$.
    *   Evaluate the objective function $\mathcal{C}_q$ for both perturbed states. This requires running the full codec pipeline (encoding and decoding) for both $\mathbf{P}^+$ and $\mathbf{P}^-$, updating the cache in the process [S1].
    *   Compute the SPSA gradient estimate for each component $i$:
        $$ \hat{g}_i(\mathbf{P}) = \frac{\mathcal{C}_q(\mathbf{P} + \epsilon \Delta) - \mathcal{C}_q(\mathbf{P} - \epsilon \Delta)}{2 \epsilon \Delta_i} $$
    *   Use this estimate for the optimizer step [S1].

**Parameters:**
*   **$\epsilon$ (Perturbation Scale):** A small scalar value. The exact value is not specified in the sources [S1].
*   **$\Delta_i$:** Random variable, $\pm 1$ with $p=0.5$ [S1].

**Pitfalls:**
*   **Computational Cost:** Applying SPSA at every step is prohibitively expensive because it requires two additional full codec round trips (encoding/decoding) per iteration [S1].
*   **Gradient Quality:** SPSA yields the worst reconstruction quality and highest training time among tested methods. This is attributed to the high variance of zeroth-order estimates and poor signal-to-noise ratio caused by the severe non-linearity of the codec [S1].
*   **Alignment:** Although SPSA is less biased in principle, its gradient estimates can be substantially noisier for complex black-box codec round-trips, making them less aligned with the descent direction needed for radiance-field optimization compared to vanilla STE [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_gradient_hybrid_ste_spsa(P, cache, epsilon, objective_fn):
    """
    Compute gradient using Hybrid STE+SPSA.
    
    Args:
        P: Current parameter vector (e.g., feature planes).
        cache: Codec cache object.
        epsilon: Perturbation scale.
        objective_fn: Function computing loss C_q, handling codec encoding/decoding.
        
    Returns:
        gradient: Estimated gradient vector.
    """
    # Check if cache is valid for current P
    if cache.is_valid(P):
        # Step 1: Use vanilla STE for cached steps
        # Vanilla STE: Backpropagate through decoder assuming identity codec
        # Gradient is copied directly to pre-codec features
        with torch.enable_grad():
            P.requires_grad = True
            # Forward pass using cached codec output or re-encoding if needed for identity assumption
            # In practice, this often means detaching the codec and passing gradient through
            loss = objective_fn(P, use_cache=True) 
            loss.backward()
            gradient = P.grad
        return gradient
    else:
        # Step 2: Cache needs refresh, use SPSA
        # Sample random perturbation vector Delta
        # Delta_i = +/- 1 with probability 0.5
        Delta = torch.randint(low=-1, high=2, size=P.shape).float()
        
        # Compute perturbed parameters
        P_plus = P + epsilon * Delta
        P_minus = P - epsilon * Delta
        
        # Evaluate objective function for both perturbed states
        # This triggers codec encoding/decoding, updating the cache
        C_plus = objective_fn(P_plus, use_cache=False)
        C_minus = objective_fn(P_minus, use_cache=False)
        
        # Compute SPSA gradient estimate
        # g_i = (C(P + e*Delta) - C(P - e*Delta)) / (2 * e * Delta_i)
        numerator = C_plus - C_minus
        denominator = 2 * epsilon * Delta
        
        # Handle division by zero if Delta_i is 0 (though Bernoulli +/-1 avoids this)
        # In this specific Bernoulli case, Delta_i is never 0.
        gradient = numerator / denominator
        
        return gradient
```

## Subtopics

### Standard-Codec-in-the-Loop (SCL)
The training framework that inserts standard, non-differentiable codecs directly into the neural radiance field training loop using a straight-through estimator (STE) to pass gradients. [SCL](17889-standard-codec-in-the-loop-scl.md)

### Caching for SCL Training
A technique to optimize SCL training by caching codec outputs to avoid redundant computations. [Caching](17907-caching-for-scl-training.md)

## Operating characteristics
*   **Time Complexity:** Significantly higher than vanilla STE. When the cache is refreshed, SPSA requires two additional full codec round trips per iteration. Applying SPSA at every step is described as "prohibitively expensive" [S1].
*   **Memory Complexity:** Requires storing the codec cache and the perturbation vector $\Delta$. No additional major memory overhead compared to standard SCL with caching [S1].
*   **Measured Performance:**
    *   **High-Rate:** PSNR 29.63, SSIM 0.930, LPIPS 0.459, Training Time 3h 18m [S1].
    *   **Low-Rate:** PSNR 27.94, SSIM 0.909, LPIPS 0.650, Training Time 3h 18m [S1].
    *   Note: Training time is reported as identical for high and low rates in the source, which may indicate a fixed number of iterations or cache refresh frequency independent of rate [S1].
*   **Strengths:** Reduces gradient bias compared to vanilla STE during cache updates by using a zeroth-order estimator that does not rely on the identity surrogate [S1].
*   **Limitations:** Worst reconstruction quality and highest training time among compared methods (vanilla STE, mSTE, STE+SPSA) due to high variance and poor signal-to-noise ratio from codec non-linearity [S1].
*   **When to Use:** Only when cache refreshes are infrequent enough that the computational cost of SPSA is manageable, and when reducing bias during updates is critical. However, the source notes that vanilla STE is more robust and provides better overall quality, especially in low-bitrate regimes [S1].
*   **When Not to Use:** When computational efficiency is a priority, or when codec non-linearity introduces too much noise for zeroth-order estimates to be effective. Vanilla STE is preferred as the default due to robustness [S1].

## Background
The Hybrid STE+SPSA method was introduced in the context of Codec-Adaptive TriPlane Radiance Fields (CATRF) for volumetric content delivery. It was proposed as a way to improve upon vanilla STE by incorporating insights from black-box gradient approximation methods, specifically SPSA [S1]. The method was evaluated in ablation studies comparing it against vanilla STE and modified STE (mSTE) [S1]. The authors found that while SPSA is less biased in principle, its practical performance was inferior due to the specific challenges of differentiating through non-linear codecs [S1].

## Sources
[S1] CATRF: Codec-Adaptive TriPlane Radiance Fields for Volumetric Content Delivery, https://doi.org/10.1109/cvprf71909.2026.00048, fetched 2026-10-30
