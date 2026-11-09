# Differentiable Shadow Kernel Sampling via Normal Distribution

Also called: ICDF sampling for differentiable shadows, Normal distribution kernel for shadow offset

## Overview
This method enables differentiable path tracing by importance sampling shadow offsets using a Normal distribution kernel in parameter space [S1]. It allows for the computation of gradients with respect to scene parameters by treating shadowing as a convolution with a smooth kernel, rather than a hard binary visibility test [S1]. Unlike standard binary shadow mapping or hard-edged shadow volumes, this approach provides continuous visibility derivatives, which is essential for optimization tasks such as material parameter recovery or geometry refinement [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A uniform random variable $\xi \in (0, 1)$, a bandwidth parameter $\bandwidth > 0$, and the current parameter vector $\theta$ [S1].
*   **Output:** A sampled offset $\offset$ and the updated parameter vector $\theta' = \theta - \offset$ [S1].

**Data Structures:**
No specific complex data structures are required beyond standard scalar variables for the random number, bandwidth, and parameters. The method operates on scalar offsets applied to parameter vectors.

**Steps:**
1.  **Define the Kernel:** The shadow offset is modeled as a Normal distribution $\mathcal{N}(0, \bandwidth)$ centered at 0 [S1]. The Probability Density Function (PDF) is defined as:
    $$ \kernel (\offset ) = \frac {1}{\bandwidth \sqrt {2\pi }} \exp \left (-\frac {\offset ^2}{2\bandwidth ^2}\right ) $$
    where $\kernel$ is the kernel density, $\offset$ is the offset value, and $\bandwidth$ controls the spread of the shadow blur [S1].

2.  **Normalize for Halfspaces:** To handle the domain constraints and ensure the CDF is well-defined, the PDF is treated separately for positive and negative halfspaces. The PDF on the positive halfspace is normalized to integrate to 0.5 [S1]. The normalized PDF for the positive halfspace is given by:
    $$ \text{PDF}_+(\offset) = \frac {\offset }{2\bandwidth ^2} \exp \left (\frac {-\offset ^2}{2\bandwidth ^2}\right ) $$
    *Note: The source text presents this specific form for the normalized PDF on the positive halfspace [S1].*

3.  **Determine Integration Constants:** The Cumulative Distribution Function (CDF) is derived by integrating the PDF. To ensure the CDF is continuous, monotonically increasing, and defined in the range $(0, 1)$, specific integration constants are chosen [S1]:
    *   For the positive halfspace ($\offset > 0$), the integration constant is $C_+ = 1$.
    *   For the negative halfspace ($\offset < 0$), the integration constant is $C_- = 0$.
    
    The resulting CDF expression involves the term:
    $$ \text{CDF}(\offset) = -0.5 \exp \left (\frac {-\offset ^2}{2\bandwidth ^2}\right ) + C $$
    where $C$ is $1$ for $\offset > 0$ and $0$ for $\offset < 0$ [S1].

4.  **Invert the CDF (ICDF):** To enable importance sampling, the CDF is inverted to obtain the Inverse CDF (ICDF). The general form of the ICDF is:
    $$ \cdf ^{-1}(\xi) = \begin {cases} \sqrt {2\bandwidth ^2\log (2(1-\xi ))} & \xi \in [0.5, 1) \\ \sqrt {2\bandwidth ^2\log (2\xi )} & \xi \in (0, 0.5] \end {cases} $$
    Here, $\xi$ is the uniform random variable [S1].

5.  **Simplify Domain Constraints:** By analyzing the domain constraints for the square root and logarithm functions, the piecewise definition can be simplified. The source states that for $\xi \in (0, 1)$, the final simplified equation for the ICDF is:
    $$ \cdf ^{-1}(\xi) = \sqrt {-2\bandwidth ^2\log (\xi)} $$
    This formula generates the offset $\offset$ directly from a single uniform random sample $\xi$ [S1].

6.  **Apply Offset:** The sampled offset $\offset$ is subtracted from the current parameters $\theta$ to obtain the new parameters $\theta'$ for the differentiable path tracing step [S1]:
    $$ \theta' = \theta - \offset $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def sample_shadow_offset(xi: float, bandwidth: float) -> float:
    """
    Samples an offset from the differentiable shadow kernel using the ICDF.
    
    Args:
        xi: A uniform random variable in (0, 1).
        bandwidth: The bandwidth parameter sigma for the Normal distribution.
        
    Returns:
        The sampled offset value.
    """
    # Guard against log(0)
    if xi <= 0.0:
        xi = 1e-9
    if xi >= 1.0:
        xi = 1.0 - 1e-9
        
    # Final simplified ICDF equation from source [S1]
    # cdf^-1(xi) = sqrt(-2 * bandwidth^2 * log(xi))
    argument = -2.0 * (bandwidth ** 2) * math.log(xi)
    
    # Ensure argument is non-negative for sqrt, though mathematically 
    # -log(xi) is positive for xi in (0,1).
    if argument < 0:
        argument = 0.0
        
    offset = math.sqrt(argument)
    return offset

def update_parameters(theta, xi, bandwidth):
    """
    Updates parameters using the sampled shadow offset.
    
    Args:
        theta: Current parameter vector.
        xi: Uniform random variable.
        bandwidth: Bandwidth parameter.
        
    Returns:
        Updated parameter vector theta'.
    """
    offset = sample_shadow_offset(xi, bandwidth)
    theta_prime = theta - offset
    return theta_prime
```

**Parameters:**
*   $\xi$: A uniform random variable, typically in $(0, 1)$ [S1].
*   $\bandwidth$: The bandwidth parameter of the Normal distribution, controlling the scale of the shadow blur [S1].

**Edge Cases and Pitfalls:**
*   **Logarithm Domain:** The ICDF involves $\log(\xi)$. If $\xi = 0$, the logarithm is undefined. In practice, $\xi$ must be clamped away from 0 (e.g., $\xi \ge \epsilon$) to avoid numerical errors [S1, G].
*   **Square Root Domain:** The argument to the square root is $-2\bandwidth^2 \log(\xi)$. Since $\xi \in (0, 1)$, $\log(\xi)$ is negative, making the argument positive. However, if $\xi$ is clamped incorrectly or if $\bandwidth$ is complex, this could fail [S1].
*   **Continuity:** The source emphasizes that the CDF must be continuous and monotonically increasing. The choice of $C_+=1$ and $C_-=0$ ensures this behavior across the halfspace boundary [S1].

## Subtopics

## Operating characteristics
**Time and Memory Complexity:**
The method involves a constant number of arithmetic operations (logarithm, multiplication, square root) per sample. Time complexity is $O(1)$ per shadow offset sample. Memory complexity is $O(1)$ as no large data structures are required.

**Measured Performance:**
Not given in the sources.

**Scaling:**
Not given in the sources.

**Strengths and Limitations:**
*   **Strengths:** Enables differentiable path tracing by providing smooth gradients for shadow visibility. Allows for optimization of scene parameters [S1].
*   **Limitations:** The method relies on a specific kernel shape (Normal distribution) which may not be optimal for all shadow geometries. The simplification of the ICDF assumes specific domain constraints that must be carefully handled [S1].

**When to use it and when not:**
Use this method when differentiable gradients with respect to shadowing are required, such as in inverse rendering or optimization tasks [S1]. Do not use it if non-differentiable, hard shadows are desired, or if computational efficiency is the primary concern without the need for gradients.

## Background
**Origin:**
The method originates from the paper "Plateau-Reduced Differentiable Path Tracing" [S1].

**History and Development:**
The technique was developed to address the issue of plateaus in differentiable rendering, where hard visibility transitions cause zero gradients. By convolving the visibility with a Normal distribution kernel, the method provides non-zero gradients over a wider range of parameter changes [S1].

**Where it is used:**
It is used in differentiable path tracing renderers for tasks such as material parameter estimation, geometry optimization, and lighting estimation [S1].

## Sources
[S1] Plateau-Reduced Differentiable Path Tracing, https://doi.org/10.1109/cvpr52729.2023.00417, fetched 2026-10-26
