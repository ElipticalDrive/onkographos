# Lipschitz Bound Computation for Skeletal Primitives

Also called: Falloff Filter Function Lipschitz Bound Computation

## Overview
This method computes the local Lipschitz bound $\lambda$ for skeletal primitives by evaluating the maximum absolute value of the derivative of the falloff filter function $g(x)$ over a specific interval $[a, b]$. It solves the problem of determining safe stepping distances in enhanced sphere tracing by providing a tight bound on the gradient magnitude, which allows for larger, safe steps along the ray compared to conservative global bounds. It is distinguished from generic interval arithmetic strategies by relying on closed-form analytical solutions for specific standard falloff functions, avoiding numerical approximation errors.

## How it works
**Inputs and Outputs:**
- **Input:** The falloff filter function $g(x)$, the interval $[a, b]$ representing the normalized distance range along the ray segment.
- **Output:** The Lipschitz bound $\lambda$, defined as $\max_{x \in [a,b]} |g'(x)|$.

**Data Structures:**
No complex data structures are required. The method relies on pre-computed analytical roots and derivative formulas for specific primitive types.

**Steps:**
1. Identify the falloff function type (Wyvill, Quadric, or Blobs).
2. Compute the root $x^*_0$ of the second-order derivative $g''(x) = 0$ within the unit interval $[0, 1]$.
3. Determine the bound $\lambda$ based on the position of $x^*_0$ relative to the input interval $[a, b]$:
   - If $a \geq 1$, then $\lambda = 0$.
   - If $x^*_0 \in [a, b]$, then $\lambda = |g'(x^*_0)|$.
   - Otherwise, $\lambda = \max(|g'(a)|, |g'(b)|)$.

**Formulas:**
For all functions, $g(x)$ is defined for $x < 1$ and $0$ otherwise. The derivatives are evaluated as follows:

**Wyvill Function:**
$$g(x) = (1 - x^2)^3$$
$$g'(x) = -6x(1-x^2)^2$$
$$g''(x) = -6(1-x^2)(1-5x^2)$$
Root: $x^*_0 = 1/\sqrt{5}$.
Bound: $\lambda_0 = 96\sqrt{5}/125 \approx 1.72$.

**Quadric Function:**
$$g(x) = (1 - x^2)^2$$
$$g'(x) = -4x(1-x^2)$$
$$g''(x) = -4(1-3x^2)$$
Root: $x^*_0 = 1/\sqrt{3}$.
Bound: $\lambda_0 = 8\sqrt{3}/9 \approx 1.54$.

**Blobs Function:**
$$g(x) = -\frac{4}{9}x^6 + \frac{17}{9}x^4 - \frac{22}{9}x^2 + 1$$
$$g'(x) = -\frac{4}{9}x(x^2 - 1)(6x^2 - 11)$$
$$g''(x) = -\frac{4}{9}(30x^4 - 51x^2 + 11)$$
Root: $x^*_0 = \frac{51 - \sqrt{1281}}{2\sqrt{15}}$.
Bound: $\lambda_0 = (17\sqrt{1281} + 453)\sqrt{\frac{17}{2278125} - \frac{\sqrt{1281}}{6834375}} \approx 1.59$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def compute_lipschitz_bound_wyvill(a, b):
    """
    Computes Lipschitz bound for Wyvill falloff function on interval [a, b].
    """
    if a >= 1:
        return 0.0
    
    # Precomputed root of g''(x) = 0
    x_star = 1.0 / math.sqrt(5.0)
    
    # Precomputed max derivative value at x_star
    # lambda_0 = 96*sqrt(5)/125
    lambda_star = 96.0 * math.sqrt(5.0) / 125.0
    
    if a <= x_star <= b:
        return lambda_star
    else:
        # Evaluate |g'(a)| and |g'(b)|
        # g'(x) = -6x(1-x^2)^2
        def g_prime(x):
            return abs(-6.0 * x * (1.0 - x*x)**2)
        
        return max(g_prime(a), g_prime(b))

def compute_lipschitz_bound_quadric(a, b):
    """
    Computes Lipschitz bound for Quadric falloff function on interval [a, b].
    """
    if a >= 1:
        return 0.0
    
    # Precomputed root of g''(x) = 0
    x_star = 1.0 / math.sqrt(3.0)
    
    # Precomputed max derivative value at x_star
    # lambda_0 = 8*sqrt(3)/9
    lambda_star = 8.0 * math.sqrt(3.0) / 9.0
    
    if a <= x_star <= b:
        return lambda_star
    else:
        # Evaluate |g'(a)| and |g'(b)|
        # g'(x) = -4x(1-x^2)
        def g_prime(x):
            return abs(-4.0 * x * (1.0 - x*x))
        
        return max(g_prime(a), g_prime(b))

def compute_lipschitz_bound_blobs(a, b):
    """
    Computes Lipschitz bound for Blobs falloff function on interval [a, b].
    """
    if a >= 1:
        return 0.0
    
    # Precomputed root of g''(x) = 0
    # x_star = (51 - sqrt(1281)) / (2 * sqrt(15))
    sqrt_1281 = math.sqrt(1281.0)
    sqrt_15 = math.sqrt(15.0)
    x_star = (51.0 - sqrt_1281) / (2.0 * sqrt_15)
    
    # Precomputed max derivative value at x_star
    # lambda_0 = (17*sqrt(1281) + 453) * sqrt(17/2278125 - sqrt(1281)/6834375)
    term1 = 17.0 * sqrt_1281 + 453.0
    term2 = 17.0 / 2278125.0 - sqrt_1281 / 6834375.0
    lambda_star = term1 * math.sqrt(term2)
    
    if a <= x_star <= b:
        return lambda_star
    else:
        # Evaluate |g'(a)| and |g'(b)|
        # g'(x) = -4/9 * x * (x^2 - 1) * (6x^2 - 11)
        def g_prime(x):
            return abs((-4.0/9.0) * x * (x*x - 1.0) * (6.0*x*x - 11.0))
        
        return max(g_prime(a), g_prime(b))
```

**Parameters:**
- None. The method is deterministic based on the function type.

**Edge Cases and Pitfalls:**
- If $a \geq 1$, the bound is exactly 0, as the falloff function is 0 and its derivative is 0 outside the unit interval.
- Floating-point precision in evaluating $g'(x)$ near boundaries should be handled carefully, though the analytical forms are stable.
- The root $x^*_0$ must be checked against the interval $[a, b]$ correctly; if the interval does not contain the peak of the derivative, the maximum occurs at one of the endpoints.

## Subtopics

### [Local Lipschitz Bound Computation for Hierarchical Implicit Models](3894-local-lipschitz-bound-computation-for-hierarchical-implicit-.md)
Computes the local Lipschitz bound $\lambda(e)$ for a segment $e$ within a hierarchical construction tree. This method provides the component-wise bound for skeletal primitives used within that larger hierarchical context.

## Operating characteristics
- **Time Complexity:** $O(1)$ per primitive evaluation. The computation involves a constant number of arithmetic operations and comparisons.
- **Memory Complexity:** $O(1)$. No additional memory is required beyond the constants for the specific function type.
- **Strengths:** Provides tight, accurate bounds that enable larger stepping distances in sphere tracing, reducing the number of field function queries. It avoids the overhead of numerical root-finding or interval arithmetic propagation for these specific primitives.
- **Limitations:** Only applicable to skeletal primitives with the specific falloff functions defined (Wyvill, Quadric, Blobs). Not applicable to arbitrary implicit functions.
- **When to use:** When using enhanced sphere tracing with skeletal primitives and needing to optimize ray marching steps.
- **When not to use:** For non-skeletal primitives or when the falloff function does not match the predefined analytical forms.

## Background
The method originates from the analysis of falloff filter functions used in soft object rendering. The Wyvill function was introduced by Wyvill in 1986 [S1, Wyvill 86]. The Quadric function is a standard $C^1$ variant. The Blobs function, described in [WMW86], modifies the quadric to satisfy specific boundary conditions. The specific Lipschitz bound computation for these functions was detailed in the context of segment tracing using local Lipschitz bounds [S1].

## Sources
[S1] Segment Tracing Using Local Lipschitz Bounds, https://doi.org/10.1111/cgf.13951, fetched 2026-10-26
