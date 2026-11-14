# Sigmoid Function with Jacobian

Also called: Logistic Function, Logistic Sigmoid, Sigmoid Function, Sigmoid derivative

## Overview
Computes the logistic sigmoid function $\sigma(x)$ and its analytical derivative $\sigma'(x)$ for a scalar input $x$. It serves as a differentiable activation function in volume rendering and differentiable rendering pipelines, mapping real-valued inputs to the $(0, 1)$ interval while providing gradient information for optimization [S1][S2, unofficial]. It is distinguished from sibling methods like the [Gaussian Transfer Function](4408-gaussian-transfer-function.md) by its S-shaped monotonicity and its specific role in soft thresholding or probabilistic aggregation rather than peak-based classification [S1].

## How it works
**Inputs and outputs:**
- Input: A scalar value $x \in \mathbb{R}$.
- Output: A tuple $(s, ds)$, where $s = \sigma(x) \in (0, 1)$ is the sigmoid value, and $ds = \sigma'(x)$ is the derivative with respect to $x$.

**Data structures:**
- No complex data structures are required. The computation operates on scalar floating-point values.

**Steps:**
1. Compute the exponential term $e^{-x}$.
2. Compute the sigmoid value $s$ using the logistic formula.
3. Compute the derivative $ds$ using the identity $\sigma'(x) = s(1-s)$.

**Formulas:**
The logistic sigmoid function is defined as [S1][S2, unofficial]:
$$ s = \sigma(x) = \frac{1}{1 + e^{-x}} $$

The derivative of the sigmoid function is [S2, unofficial]:
$$ \frac{ds}{dx} = s(1 - s) $$

**Parameters:**
- None. The function is standard and parameter-free.

**Edge cases and pitfalls:**
- **Numerical Stability:** For large positive $x$, $e^{-x}$ approaches 0, and $s$ approaches 1. For large negative $x$, $e^{-x}$ becomes very large, potentially causing overflow if computed naively as $1/(1+e^{-x})$ without care, though standard IEEE 754 floating-point handles large exponents by saturating to infinity, resulting in $s=0$. In practice, implementations should handle extreme values to avoid NaNs or infinities if downstream operations are sensitive [G].
- **Derivative at Extremes:** As $x \to \pm \infty$, $ds \to 0$. This can lead to vanishing gradients in deep optimization problems [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sigmoid_with_jacobian(x: float) -> tuple[float, float]:
    """
    Computes the logistic sigmoid function and its derivative.
    
    Args:
        x: A scalar real number.
        
    Returns:
        A tuple (s, ds) where s is the sigmoid value and ds is the derivative.
    """
    # Compute e^(-x)
    # Note: For very large negative x, exp(-x) may overflow. 
    # A robust implementation might use conditional logic or specialized libraries,
    # but standard float64 handles up to ~709 before overflow.
    exp_neg_x = math.exp(-x)
    
    # Compute sigmoid s = 1 / (1 + e^(-x))
    s = 1.0 / (1.0 + exp_neg_x)
    
    # Compute derivative ds = s * (1 - s)
    ds = s * (1.0 - s)
    
    return s, ds
```

## Subtopics

### [Inverse Sigmoid Function](7961-inverse-sigmoid-function.md)
Computes the inverse of the sigmoid function, mapping probability values in $(0, 1)$ to the real line $(-\infty, \infty)$. It serves as the canonical link function in logistic regression and is used in volume rendering to invert normalized transfer function outputs back to logit space for linear interpolation or gradient-based optimization. Unlike the parent [Sigmoid Function with Jacobian](6497-sigmoid-function-with-jacobian.md) which maps reals to probabilities, this method performs the reverse transformation.

## Operating characteristics
- **Time Complexity:** $O(1)$ per evaluation. Requires one exponential and a few arithmetic operations.
- **Memory Complexity:** $O(1)$. No auxiliary storage is needed.
- **Strengths:** Analytically differentiable, computationally inexpensive, and bounded output makes it suitable for probabilistic interpretations and soft thresholding in differentiable renderers [S1].
- **Limitations:** Suffers from vanishing gradients for large $|x|$ [G]. Not suitable for multi-modal distributions where Gaussian functions are preferred [S1].
- **When to use:** When a smooth, monotonic transition between two states is required, or when aggregating probabilistic contributions in differentiable rendering [S1].
- **When not to use:** When multi-modal classification is needed (use [Gaussian Transfer Function](4408-gaussian-transfer-function.md)) or when the inverse mapping is required (use [Inverse Sigmoid Function](7961-inverse-sigmoid-function.md)).

## Background
The logistic sigmoid function, also known as the FL function, corresponds to the logistic distribution [S1]. It has been widely used in statistics and machine learning as an activation function. In the context of differentiable rendering, it is popular for soft evaluation of geometric primitives (e.g., determining if a pixel lies inside a triangle) and for aggregating face contributions, where it provides a differentiable approximation of hard boolean operations [S1]. It is frequently referred to as "the" sigmoid in recent literature, distinguishing it from more general sigmoidal functions [S1].

## Sources
[S1] GenDR: A Generalized Differentiable Renderer, https://doi.org/10.1109/cvpr52688.2022.00397, fetched 2026-10-26
[S2] wanmeihuali/taichi_3d_gaussian_splatting, https://github.com/wanmeihuali/taichi_3d_gaussian_splatting, fetched 2026-10-27 (unofficial)
