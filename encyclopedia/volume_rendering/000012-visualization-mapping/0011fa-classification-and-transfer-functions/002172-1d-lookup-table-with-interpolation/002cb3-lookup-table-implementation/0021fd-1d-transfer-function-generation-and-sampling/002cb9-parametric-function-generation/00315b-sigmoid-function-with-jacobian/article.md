# Sigmoid Function with Jacobian

Also called: Sigmoid, Sigmoid derivative

## Overview
Computes the logistic sigmoid function and its analytical derivative (Jacobian) for a scalar input. It maps real-valued inputs to the $(0, 1)$ range, providing a smooth, differentiable activation often used in transfer functions for soft classification or as a component in more complex parametric mappings. It is distinguished from siblings like the Gaussian Transfer Function by its asymmetric S-shape and bounded output range, whereas Gaussians are symmetric and bell-shaped.

## How it works
**Inputs and outputs:**
- Input: A scalar value $x \in \mathbb{R}$.
- Outputs: The sigmoid value $s \in (0, 1)$ and its derivative $s' \in (0, 0.25]$.

**Data structures:**
- No complex data structures; operates on scalar floats.

**Formulas:**
The sigmoid function is defined as:
$$ s = \sigma(x) = \frac{1}{1 + e^{-x}} $$

The derivative (Jacobian) with respect to $x$ is:
$$ \frac{ds}{dx} = s(1 - s) $$

**Parameters:**
- None. The function is standard.

**Edge cases and pitfalls:**
- **Numerical Stability:** For large positive $x$, $e^{-x}$ underflows to 0, yielding $s=1$. For large negative $x$, $e^{-x}$ overflows, yielding $s=0$. In implementations using floating-point arithmetic, direct computation of $1/(1+e^{-x})$ can suffer from overflow if $x$ is very negative (e.g., $x < -700$ for float32). A numerically stable implementation should check the sign of $x$ and compute $e^x/(1+e^x)$ for $x < 0$. [G]
- **Derivative Range:** The maximum derivative is $0.25$ at $x=0$. This limits the gradient magnitude in backpropagation or sensitivity analysis.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sigmoid(x):
    """
    Computes the sigmoid function sigma(x) = 1 / (1 + exp(-x)).
    Handles numerical stability for large negative x.
    """
    if x >= 0:
        z = exp(-x)
        return 1.0 / (1.0 + z)
    else:
        z = exp(x)
        return z / (1.0 + z)

def sigmoid_with_jacobian(x):
    """
    Computes the sigmoid function and its derivative.
    Returns: (s, ds_dx)
    """
    s = sigmoid(x)
    ds_dx = s * (1.0 - s)
    return s, ds_dx
```

## Subtopics

### Inverse Sigmoid Function
Computes the inverse sigmoid function, also known as the logit function. It maps a value $x \in (0, 1)$ back to the real line $\mathbb{R}$. It is defined as $\text{logit}(x) = \ln\left(\frac{x}{1-x}\right)$. This is useful for converting normalized probabilities or transfer function outputs back to the original data space. Implementation requires clamping $x$ to $(0, 1)$ to avoid logarithm of zero or negative numbers. [S2, unofficial]

## Operating characteristics
- **Time Complexity:** $O(1)$. Requires one exponential and a few arithmetic operations.
- **Memory Complexity:** $O(1)$.
- **Strengths:** Smooth, differentiable, bounded output. Analytical derivative is cheap to compute from the function output.
- **Limitations:** Asymmetric; saturates at 0 and 1, leading to vanishing gradients for extreme inputs. Not suitable for symmetric highlighting of a specific range (use Gaussian for that).
- **When to use:** When a smooth, monotonic transition from 0 to 1 is needed, or when the derivative is required for optimization or sensitivity analysis.

## Background
The sigmoid function is a fundamental activation function in neural networks and statistics. In the context of volume rendering and 3D Gaussian Splatting, it is used for soft classification or blending operations. The specific implementation with Jacobian is found in Taichi-based 3D Gaussian Splatting visualizers. [S1, unofficial]

## Sources
[S1] wanmeihuali/taichi_3d_gaussian_splatting, https://github.com/wanmeihuali/taichi_3d_gaussian_splatting, fetched 2026-10-27 (unofficial)
[S2] NVlabs/InstantSplat, https://github.com/NVlabs/InstantSplat, fetched 2026-10-27 (unofficial)
