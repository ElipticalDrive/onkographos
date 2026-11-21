# Complexity Dropout

Also called: Complexity-aware dropout, CoD

## Overview
Complexity Dropout is a training regularization technique for Gaussian Splatting that probabilistically excludes the complexity term from the importance score calculation during densification [S1, unofficial]. It solves the problem of densification repeatedly targeting the same high-complexity regions, which prevents opacity sampling from covering remaining areas and leads to suboptimal reconstruction [S1, unofficial]. Unlike fixed dropout or opacity sampling alone, it balances structural refinement with coverage by introducing stochasticity into the primitive selection process [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: The current set of Gaussian primitives, their attributes (position, covariance, opacity, color), the training loss gradient, and the dropout rate parameter $\delta$.
- Output: A modified importance score used to select primitives for densification (splitting or cloning) and pruning.

**Data Structures:**
- Primitive List: A list of Gaussian primitives. Each primitive $i$ has an associated complexity score $C_i$ (derived from the determinant of the covariance matrix or similar metric) and an opacity score $O_i$.
- Dropout Mask: A binary mask $M \in \{0, 1\}^N$ where $N$ is the number of primitives.

**Parameters:**
- $\delta$: The dropout rate. $\delta \in [0, 1]$.
  - $\delta = 0$: Always applies the complexity term (no dropout).
  - $\delta = 1$: Reduces to pure opacity sampling (complexity term always dropped).
  - Default: $\delta = 0.2$ [S1, unofficial].

**Steps:**
1. **Calculate Importance Scores:** For each primitive $i$, calculate the base importance score $I_i$ based on the gradient of the loss with respect to the primitive's parameters.
2. **Calculate Complexity Term:** Compute the complexity score $C_i$ for each primitive. This is typically related to the spatial extent or anisotropy of the Gaussian.
3. **Generate Dropout Mask:** For each primitive $i$, generate a Bernoulli random variable $m_i$:
   $$ m_i = \begin{cases} 0 & \text{with probability } \delta \\ 1 & \text{with probability } 1 - \delta \end{cases} $$
   The mask $M = [m_1, m_2, \dots, m_N]$.
4. **Modify Importance Score:** Compute the modified importance score $I'_i$:
   $$ I'_i = I_i \cdot (1 - m_i \cdot C_i) $$
   *Note: The exact formulation may vary. If the complexity term is additive, it might be $I'_i = I_i - m_i \cdot C_i$. However, the source implies excluding the complexity term from the importance score calculation. A common interpretation in "dropout" contexts for selection is to zero out the contribution of the complexity term when the mask is active. If the importance score is a weighted sum $I_i = \alpha \cdot G_i + \beta \cdot C_i$, then $I'_i = \alpha \cdot G_i + \beta \cdot (1 - m_i) \cdot C_i$. Given the description "excludes the complexity term", we assume the complexity component is masked out.*
   *Refined Interpretation based on "excludes the complexity term from the importance score calculation":*
   Let the importance score be $S_i = f(G_i, C_i)$. If $m_i=1$ (dropout active), $S'_i = f(G_i, 0)$ or $S'_i = G_i$. If $m_i=0$, $S'_i = S_i$.
   Let's assume a linear combination for clarity: $S_i = w_g G_i + w_c C_i$.
   $$ S'_i = w_g G_i + w_c (1 - m_i) C_i $$
5. **Select Primitives:** Select primitives for densification based on the highest $S'_i$ values.
6. **Apply Densification:** Split or clone the selected primitives.

**Edge Cases and Pitfalls:**
- $\delta = 0$: Not beneficial. Falls below opacity sampling on Deep Blending (29.00 vs 29.28 AUC PSNR) [S1, unofficial].
- Large $\delta$ (0.6, 0.8): Performs worst on both datasets [S1, unofficial].
- Precision: Ensure $C_i$ is normalized or scaled appropriately relative to $G_i$ to prevent one term from dominating.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def complexity_dropout(primitives, gradients, dropout_rate, complexity_weights, gradient_weights):
    """
    Applies complexity dropout to primitive importance scores.
    
    Args:
        primitives: List of primitive objects with attributes 'complexity' (C) and 'opacity' (O).
        gradients: List of gradient magnitudes (G) for each primitive.
        dropout_rate: Float, delta (0 to 1).
        complexity_weights: Float, weight for complexity term in importance score.
        gradient_weights: Float, weight for gradient term in importance score.
        
    Returns:
        Modified importance scores.
    """
    n = len(primitives)
    if n == 0:
        return []
    
    # Extract complexity and gradient values
    complexities = np.array([p.complexity for p in primitives])
    gradients = np.array(gradients)
    
    # Generate dropout mask
    # m_i = 1 with probability (1 - delta), 0 with probability delta
    # If m_i = 0, complexity term is dropped (multiplied by 0)
    # If m_i = 1, complexity term is kept
    # Note: The source says "excludes the complexity term". 
    # If dropout_rate = delta, then with probability delta, we exclude.
    # So mask should be 0 with prob delta, 1 with prob 1-delta.
    mask = np.random.binomial(1, 1 - dropout_rate, n)
    
    # Calculate base importance scores
    # S_i = w_g * G_i + w_c * C_i
    base_scores = gradient_weights * gradients + complexity_weights * complexities
    
    # Calculate modified scores
    # If mask[i] == 0, complexity term is excluded: S'_i = w_g * G_i
    # If mask[i] == 1, complexity term is included: S'_i = w_g * G_i + w_c * C_i
    # This can be written as: S'_i = w_g * G_i + w_c * C_i * mask[i]
    modified_scores = gradient_weights * gradients + complexity_weights * complexities * mask
    
    return modified_scores
```

## Subtopics

## Operating characteristics
- **Time Complexity:** $O(N)$ per iteration for mask generation and score modification, where $N$ is the number of primitives. This is negligible compared to the densification and rendering steps.
- **Memory Complexity:** $O(N)$ for storing the mask.
- **Measured Performance:**
  - On TandT dataset with $\delta=0.2$: PSNR 24.65, SSIM 0.868, LPIPS 0.150 [S1, unofficial].
  - On DB dataset with $\delta=0.2$: PSNR 29.92, SSIM 0.911, LPIPS 0.240 [S1, unofficial].
  - These are the best or near-best results compared to other $\delta$ values (0, 0.4, 0.6, 0.8, 1) [S1, unofficial].
- **Strengths:** Prevents over-concentration on high-complexity regions, improves coverage, leads to more stable training, especially on indoor scenes [S1, unofficial].
- **Limitations:** Requires tuning of $\delta$. Poor choices (0, 0.6, 0.8) degrade performance.
- **When to use:** When densification is stuck in local optima or over-fitting to complex regions.
- **When not to use:** When computational budget is extremely tight and stability is not a concern, or when opacity sampling alone is sufficient.

## Background
Originated in the context of Gaussian Splatting optimization, specifically in the KoalaGS method [S1, unofficial]. It addresses the issue of "fragile co-adaptation" and "needle-like anisotropic structures" in sparse-view tomography [S1, unofficial]. The method is part of a broader family of regularization techniques for output noise in differentiable volume rendering.

## Sources
[S1] KoalaGS: Complexity-Aware Continuous Level of Detail for Gaussian Splatting, https://arxiv.org/abs/2610.28615, fetched 2026-10-26 (unofficial)
