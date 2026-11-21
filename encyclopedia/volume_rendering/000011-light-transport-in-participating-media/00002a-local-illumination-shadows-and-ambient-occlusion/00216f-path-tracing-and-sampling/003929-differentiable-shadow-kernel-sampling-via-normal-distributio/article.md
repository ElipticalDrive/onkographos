# Differentiable Shadow Kernel Sampling via Normal Distribution

Also called: ICDF sampling for differentiable shadows, Normal distribution kernel for shadow offset

## Overview
This method provides an importance sampling strategy for shadow offsets in differentiable path tracing by modeling the offset distribution as a Normal distribution kernel in parameter space [S1]. It solves the problem of generating differentiable shadow samples by deriving a closed-form Inverse Cumulative Distribution Function (ICDF) that maps uniform random variables to offset values [S1]. It is distinguished from standard shadow sampling by its explicit construction of a continuous, monotonically increasing CDF over split halfspaces to ensure valid probability density and differentiability [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A uniform random variable $\uniformRandom \in (0, 1)$, and a bandwidth parameter $\bandwidth > 0$ [S1].
*   **Output:** A sampled shadow offset $\offset$ [S1].

**Data Structures:**
No complex data structures are required. The method relies on scalar arithmetic operations [S1].

**Parameters:**
*   $\bandwidth$: The bandwidth of the Normal distribution kernel [S1].
*   $\uniformRandom$: A uniform random variable used for sampling [S1].

**Method Steps:**

1.  **Define the Kernel:** Define the probability density function (PDF) as a Normal distribution $\mathcal{N}(0, \bandwidth)$ with mean 0 [S1]. The PDF is given by:
    $$
    \kernel (\offset ) = \frac {1}{\bandwidth \sqrt {2\pi }} \exp \left (-\frac {\offset ^2}{2\bandwidth ^2}\right )
    $$
    where $\kernel$ is the kernel density and $\offset$ is the offset value [S1].

2.  **Normalize for Halfspaces:** Treat the positive and negative halfspaces separately. Normalize the PDF on the positive halfspace such that it integrates to 0.5 [S1]. The normalized PDF on the positive halfspace is:
    $$
    \frac {\offset }{2\bandwidth ^2} \exp \left (\frac {-\offset ^2}{2\bandwidth ^2}\right )
    $$
    [S1].

3.  **Derive the CDF:** Integrate the normalized PDF to obtain the Cumulative Distribution Function (CDF). For the positive halfspace, the CDF is:
    $$
    -0.5 \exp \left (\frac {-\offset ^2}{2\bandwidth ^2}\right ) + \cplus
    $$
    where $\cplus$ is the integration constant [S1].

4.  **Set Integration Constants:** Determine the integration constants $C_+$ for the positive halfspace and $C_-$ for the negative halfspace. To ensure the CDF is continuous, monotonically increasing, and defined in the range $(0, 1)$, set $C_+ = 1$ and $C_- = 0$ [S1].

5.  **Invert the CDF (ICDF):** Solve for the offset $\offset$ in terms of the cumulative probability $\uniformRandom$ to obtain the Inverse CDF (ICDF). The general form for the ICDF is:
    $$
    \cdf ^{-1}(\uniformRandom ) = \begin {cases} \sqrt {2\sigma ^2\log (2(1-\uniformRandom ))} & \offset > 0 \\ \sqrt {2\sigma ^2\log (2\uniformRandom )} & \offset < 0 \,. \end {cases}
    $$
    where $\sigma$ corresponds to $\bandwidth$ [S1].

6.  **Simplify Domain Constraints:** Analyze the domain constraints for the square root and logarithm functions. The ICDF for the positive halfspace is defined for $\uniformRandom \in [0.5, 1)$, and the negative counterpart is defined for $\uniformRandom \in (0, 0.5]$ [S1]. Given a uniform random variable $\uniformRandom \in (0, 1)$, these cases can be unified into a single simplified equation:
    $$
    \cdf ^{-1}(\uniformRandom )= \sqrt {-2\bandwidth ^2\log (\uniformRandom )}
    $$
    [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def sample_shadow_offset(uniform_random: float, bandwidth: float) -> float:
    """
    Samples a shadow offset using the ICDF of a Normal distribution kernel.
    
    Args:
        uniform_random: A uniform random variable in (0, 1).
        bandwidth: The bandwidth parameter of the Normal distribution.
        
    Returns:
        The sampled shadow offset.
    """
    # Guard against log(0)
    if uniform_random <= 0.0:
        uniform_random = 1e-9  # [G] Small epsilon to prevent log(0)
    if uniform_random >= 1.0:
        uniform_random = 1.0 - 1e-9  # [G] Small epsilon to prevent log(0)

    # Simplified ICDF derived from the Normal distribution kernel
    # cdf^{-1}(u) = sqrt(-2 * bandwidth^2 * log(u))
    argument = -2.0 * (bandwidth ** 2) * math.log(uniform_random)
    
    # Guard against negative argument due to floating point errors
    if argument < 0.0:
        argument = 0.0
        
    return math.sqrt(argument)
```

**Edge Cases and Pitfalls:**
*   **Logarithm Domain:** The logarithm function $\log(\uniformRandom)$ requires $\uniformRandom > 0$. If $\uniformRandom$ is exactly 0, the result is undefined. In practice, clamp or offset the random value slightly away from 0 [G].
*   **Square Root Domain:** The argument to the square root must be non-negative. Since $\bandwidth^2 > 0$ and $-\log(\uniformRandom) \ge 0$ for $\uniformRandom \in (0, 1]$, the argument is non-negative. However, floating-point precision errors might result in a very small negative number if $\uniformRandom$ is extremely close to 1. Clamp the argument to 0 if negative [G].
*   **Bandwidth:** The bandwidth $\bandwidth$ must be positive. A zero or negative bandwidth is invalid for a Normal distribution [S1].

## Operating characteristics
**Time and Memory Complexity:**
The method involves a constant number of arithmetic operations (logarithm, multiplication, square root). Time complexity is $O(1)$. Memory complexity is $O(1)$ as no additional data structures are needed [S1].

**Measured Performance:**
Not given in the sources.

**Scaling:**
Not given in the sources.

**Strengths and Limitations:**
*   **Strengths:** Provides a closed-form solution for sampling, which is computationally efficient and differentiable, enabling gradient-based optimization in differentiable rendering [S1].
*   **Limitations:** The method relies on the assumption that the shadow offset distribution can be well-approximated by a Normal distribution kernel. The simplification to a single formula assumes a specific mapping of the uniform random variable to the halfspaces, which might need careful handling in implementation to ensure correct probability mass distribution [S1].

**When to Use:**
Use this method when performing differentiable path tracing where shadow offsets need to be sampled from a Normal distribution kernel, and computational efficiency and differentiability are required [S1].

**When Not to Use:**
Not specified in the sources.

## Background
**Origin:**
The method is described in "Plateau-Reduced Differentiable Path Tracing" [S1].

**History and Development:**
The method was developed to enable differentiable path tracing by providing a robust way to sample shadow offsets using a Normal distribution kernel [S1].

**Where it is Used:**
It is used in differentiable rendering systems, specifically for local illumination, shadows, and ambient occlusion calculations within participating media [S1].

## Sources
[S1] Plateau-Reduced Differentiable Path Tracing, https://doi.org/10.1109/cvpr52729.2023.00417, fetched 2026-10-26
