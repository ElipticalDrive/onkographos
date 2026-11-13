# Sigmoid Function with Jacobian

Also called: Logistic Function, Logistic Sigmoid, Sigmoid Function, Sigmoid derivative

## Overview
Computes the logistic sigmoid function $\sigma(x)$ and its derivative $\sigma'(x)$ simultaneously. It is used in volume rendering and differentiable rendering pipelines to map scalar inputs to probabilities or soft boundaries, enabling gradient-based optimization. It is distinguished from sibling methods like [Gaussian Transfer Function](4408-gaussian-transfer-function.md) by its S-shaped monotonicity and its explicit provision of the Jacobian (derivative) for backpropagation, which is critical for differentiable renderers [S1][S2, unofficial].

## How it works
**Inputs and outputs:**
- Input: A scalar value $x \in \mathbb{R}$.
- Output: A tuple $(s, ds)$, where $s = \sigma(x)$ is the sigmoid value and $ds = \sigma'(x)$ is its derivative.

**Data structures:**
- No persistent data structures. The computation is stateless and element-wise.

**Steps:**
1. Compute the exponential term $e^{-x}$.
2. Compute the sigmoid value $s = \frac{1}{1 + e^{-x}}$.
3. Compute the derivative $ds = s(1 - s)$.
4. Return $(s, ds)$.

**Formulas:**
The logistic sigmoid function is defined as:
$$ s = \sigma(x) = \frac{1}{1 + e^{-x}} $$

The derivative (Jacobian) is:
$$ \frac{ds}{dx} = s(1 - s) $$

**Parameters:**
- None. The function is standard.

**Edge cases and pitfalls:**
- **Numerical Stability:** For large positive $x$, $e^{-x}$ underflows to 0, yielding $s=1$. For large negative $x$, $e^{-x}$ overflows, causing division by zero or infinity if not handled. In practice, implementations should clamp $x$ or use a numerically stable formulation (e.g., using `log1p` or conditional branches) to avoid `inf` or `nan` in the derivative [G].
- **Derivative at Extremes:** As $|x| \to \infty$, $ds \to 0$. This can lead to vanishing gradients in deep networks or optimization loops if inputs are far from the decision boundary [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def sigmoid_with_jacobian(x):
    # Numerical stability guard: clamp x to prevent exp overflow/underflow issues
    # Standard float32 exp overflows around 88.7, underflows around -87.3
    # Clamping to [-10, 10] is often sufficient for probability mapping, 
    # but [-100, 100] is safer for general differentiable rendering.
    x_clamped = max(-100.0, min(100.0, x))
    
    # Compute sigmoid
    # Using exp(-x) is standard. For x > 0, this is stable. 
    # For x < 0, exp(-x) is large, but 1/(1+large) is small, which is stable.
    # However, direct exp(-x) for very negative x causes overflow.
    # A robust implementation checks sign:
    if x_clamped >= 0:
        exp_neg_x = math.exp(-x_clamped)
        s = 1.0 / (1.0 + exp_neg_x)
    else:
        exp_pos_x = math.exp(x_clamped)
        s = exp_pos_x / (1.0 + exp_pos_x)
        
    # Compute derivative
    ds = s * (1.0 - s)
    
    return s, ds
```

## Subtopics

## Operating characteristics
- **Time Complexity:** $O(1)$ per sample. Requires one exponential, one division, and two multiplications.
- **Memory Complexity:** $O(1)$. No auxiliary storage.
- **Performance:** Extremely fast. The derivative is computed from the sigmoid value without additional expensive operations.
- **Strengths:** Provides exact gradients required for differentiable rendering (e.g., GenDR) [S1]. Simple to implement.
- **Limitations:** Suffers from vanishing gradients for inputs far from 0. Not suitable for multi-modal distributions where [Gaussian Transfer Function](4408-gaussian-transfer-function.md) might be preferred.

## Background
The logistic sigmoid function, also known as the FL function, corresponds to the logistic distribution [S1]. It is a popular choice in differentiable renderers for soft evaluation of geometric boundaries (e.g., pixel occupancy inside a triangle) because it provides a smooth, differentiable approximation of step functions [S1]. It is widely used in Taichi-based 3D Gaussian Splatting implementations for its efficiency and differentiability [S2, unofficial].

## Sources
[S1] GenDR: A Generalized Differentiable Renderer, https://doi.org/10.1109/cvpr52688.2022.00397, fetched 2026-10-26
[S2] wanmeihuali/taichi_3d_gaussian_splatting, https://github.com/wanmeihuali/taichi_3d_gaussian_splatting, fetched 2026-10-27 (unofficial)
