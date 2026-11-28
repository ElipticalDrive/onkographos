# Sigmoid Clamping
Also called: Smoothing and Clipping Function (ξ), Soft Clamping
## Overview
Sigmoid Clamping is a non-linear mapping technique that smooths and clips scalar values into the normalized range $[0, 1]$. It serves as a robust boundary handler for objective functions or transfer function inputs, preventing outliers from disrupting optimization or rendering by providing a soft transition at the boundaries rather than a hard cutoff. It differs from standard [Clamping](11530-clamping.md) by introducing a sigmoidal slope near the edges, which preserves gradient information for differentiable pipelines.

## How it works
**Inputs and Outputs:**
- Input: A scalar value $x \in \mathbb{R}$.
- Output: A scalar value $\xi(x) \in [0, 1]$.

**Parameters:**
- $\alpha_3$: A shape parameter controlling the steepness of the sigmoid transition. The standard value is $3$ [S1, unofficial].

**Formula:**
The function is defined as [S1, unofficial]:
$$
\xi(x) = \max\left(0, \min\left(1, \frac{1}{3} \left[ 1 + e^{(-\alpha_3 e x) + (\alpha_3 0.5 e)} \right]^{-1} \right)\right)
$$
Where:
- $e$ is the base of the natural logarithm ($\approx 2.71828$).
- The term $(-\alpha_3 e x) + (\alpha_3 0.5 e)$ represents the exponent argument.
- The inner term $\frac{1}{3} [\dots]^{-1}$ scales the sigmoid output. Note that the standard logistic function is $\frac{1}{1+e^{-z}}$. Here, the exponent is $z = -\alpha_3 e x + \alpha_3 0.5 e$.
- The outer $\max(0, \min(1, \dots))$ ensures the final result is strictly bounded within $[0, 1]$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def sigmoid_clamping(x: float, alpha_3: float = 3.0) -> float:
    """
    Computes the Sigmoid Clamping function xi(x).
    """
    # Calculate the exponent term: (-alpha_3 * e * x) + (alpha_3 * 0.5 * e)
    # Note: 'e' is the mathematical constant Euler's number.
    e = math.e
    exponent = (-alpha_3 * e * x) + (alpha_3 * 0.5 * e)
    
    # Calculate the sigmoid-like term: 1 / (1 + exp(exponent))
    # The source formula uses [1 + e^(...)]^-1 which is equivalent to 1 / (1 + e^(...))
    sigmoid_term = 1.0 / (1.0 + math.exp(exponent))
    
    # Scale by 1/3
    scaled = (1.0 / 3.0) * sigmoid_term
    
    # Clamp to [0, 1]
    result = max(0.0, min(1.0, scaled))
    
    return result
```

**Edge Cases and Pitfalls:**
- **Precision:** For large positive $x$, the exponent becomes large negative, $e^{\text{exponent}} \to 0$, and $\xi(x) \to 1/3$. For large negative $x$, the exponent becomes large positive, $e^{\text{exponent}} \to \infty$, and $\xi(x) \to 0$. The hard clamping at $0$ and $1$ handles any numerical overflow/underflow in the exponential.
- **Scaling Factor:** The factor $1/3$ means the unclamped sigmoid output is scaled down. This implies that even at the peak of the sigmoid (when the exponent is 0), the value is $1/3$. The function relies on the clamping and the specific parameterization to map the relevant input range to the full $[0,1]$ output effectively for its specific application (objective function smoothing) [S1, unofficial].

## Subtopics

### Truncated Exponential Activation
A variant used in deep learning contexts (specifically `ngp_pl`) where the exponential function is applied directly to the input $x$, but the pre-activation value is clamped to $[-15, 15]$ during the backward pass to prevent gradient explosion [S2, unofficial].
- **Forward:** Compute $y = \exp(x)$.
- **Backward:** Compute gradient $dL/dx = dL/dy \cdot \exp(\text{clamp}(x, -15, 15))$.
- **Parameter:** Clamp bounds $[-15, 15]$ applied to $x$ before exponentiation in the gradient calculation [S2, unofficial].
- **Implementation:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Forward pass
def forward(x):
    return torch.exp(x)

# Backward pass (conceptual)
def backward(dL_dout, x):
    # Clamp x to [-15, 15] before exponentiating for gradient stability
    x_clamped = torch.clamp(x, -15, 15)
    return dL_dout * torch.exp(x_clamped)
```
This variant is chosen when numerical stability of the gradient is critical in neural network training, differing from Sigmoid Clamping which is a fixed mathematical mapping for scalar normalization [S2, unofficial].

## Operating characteristics
- **Time Complexity:** $O(1)$ per element. Requires one exponential, one addition, one division, and two comparisons.
- **Memory Complexity:** $O(1)$ auxiliary memory.
- **Strengths:** Differentiable almost everywhere (except at the hard clamping boundaries if not smoothed, but the sigmoid part is smooth). Provides soft boundaries which are beneficial for gradient-based optimization of objective functions [S1, unofficial].
- **Limitations:** The specific scaling factor ($1/3$) and parameterization are tailored to the SOARNet objective functions; it may not be a universal transfer function clamping method for general volume rendering without re-tuning [S1, unofficial]. The hard clamping at 0 and 1 creates non-differentiable points if the input exactly hits the boundary, though this is rare in practice.
- **When to use:** When smoothing objective function scores (like ray diversity or pose consistency) before aggregation or loss calculation [S1, unofficial]. When a soft, bounded output is needed for differentiable rendering pipelines.

## Background
Sigmoid Clamping, as defined by the specific formula with $\alpha_3=3$, was introduced in the context of the SOARNet architecture for active view planning in NeRF [S1, unofficial]. It is used to normalize objective functions ($f_C, f_Q, f_D, f_T$) to the range $[0, 1]$ to facilitate efficient computation and comparison within the surrogate objective framework [S1, unofficial]. The Truncated Exponential variant is associated with the `ngp_pl` implementation, aiming to improve inference speed and stability compared to other NeRF implementations like `torch-ngp` [S2, unofficial].

## Sources
[S1] SO-NeRF: Active View Planning for NeRF using Surrogate Objectives, https://doi.org/10.48550/arXiv.2312.03266, fetched 2026-10-27 (unofficial)
[S2] kwea123/ngp_pl, https://github.com/kwea123/ngp_pl, fetched 2026-10-26 (unofficial)
