# Inexact Line Search Optimization

Also called: Line Search

## Overview
Inexact Line Search Optimization is an iterative method that adapts the step size in gradient descent to achieve a reduction in the objective function while ensuring sufficiently fast progress [S1]. It solves the problem of slow convergence in standard gradient descent by dynamically selecting a step size that guarantees descent without the computational cost of finding the exact minimum along the search direction [S1]. Unlike exact line search, which may suffer from convergence deterioration, this inexact strategy yields reasonable results and descends much faster [S1]. It builds on [Gradient Descent Solver](3280-gradient-descent-solver.md).

## How it works
**Inputs:**
- $x_k$: Current point (e.g., feature opacity values) at iteration $k$.
- $d_k$: Descent direction (e.g., negative gradient).
- $f$: Objective function to minimize (e.g., visibility-weighted saliency error).
- $M$: Maximum iteration count for the line search.

**Outputs:**
- $\gamma_m$: The chosen step size.
- $x_{k+1}$: The updated point, computed as $x_k + \gamma_m d_k$.

**Data Structures:**
- Integer $m$: Current line search iteration counter, initialized to 0.
- Integer $M$: Maximum allowed line search iterations.
- Scalar $\gamma_m$: Current step size candidate, defined as $\gamma_m = 2^m$.

**Algorithm Steps:**
1. Initialize $m = 0$.
2. Set $M$ to the maximum iteration count.
3. Check if the objective function value decreases with the next candidate step size:
   $$ f(x_k + \gamma_{m+1} d_k) < f(x_k + \gamma_m d_k) $$
   where $\gamma_{m+1} = 2^{m+1}$ and $\gamma_m = 2^m$.
4. If the condition is met AND $m < M - 1$:
   - Increment $m$ by 1.
   - Repeat Step 3.
5. Otherwise (condition fails OR $m = M - 1$):
   - Terminate the line search.
   - Set $\gamma_m$ as the chosen step size.
6. Update the solution: $x_{k+1} = x_k + \gamma_m d_k$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def inexact_line_search(x_k, d_k, f, M):
    """
    Performs inexact line search to find step size gamma.
    
    Args:
        x_k: Current point (vector).
        d_k: Descent direction (vector).
        f: Objective function callable.
        M: Maximum iteration count.
        
    Returns:
        gamma: Chosen step size.
    """
    m = 0
    # Precompute current step size gamma_m = 2^m
    gamma_m = 1.0  # 2^0
    
    while m < M - 1:
        # Calculate next step size gamma_{m+1} = 2^(m+1)
        gamma_next = gamma_m * 2.0
        
        # Evaluate objective function at current and next candidate points
        f_current = f(x_k + gamma_m * d_k)
        f_next = f(x_k + gamma_next * d_k)
        
        # Check descent condition
        if f_next < f_current:
            m = m + 1
            gamma_m = gamma_next
        else:
            # Condition failed or max iterations reached (handled by loop exit)
            break
            
    return gamma_m
```

**Parameters:**
- $M$: Maximum iteration count. Typical values are not specified in the sources. [G] A small integer (e.g., 10-20) is typical for exponential search to prevent overflow, but the source does not specify.

**Edge Cases and Pitfalls:**
- **Convergence Deterioration:** Exact line search may suffer from convergence deterioration; this inexact strategy avoids this [S1].
- **Convexity:** The parallel variant has the same behavior as the sequential version if the objective function is convex [S1].
- **Objective Function Cost:** The objective function evaluation can be expensive (e.g., requiring visibility field recomputation), which justifies the use of line search [S1].

## Subtopics

### Parallel Inexact Line Search
A parallel strategy that evaluates the objective function at different candidate points in parallel along the line search direction to accelerate optimization on multi-core processors [S1]. It generates a list of $M$ step sizes $S = \{\gamma_0, \gamma_1, ..., \gamma_{M-1}\}$ where $\gamma_m = 2^m$ [S1]. It evaluates $f(x_k + \gamma_m d_k)$ in parallel for each $\gamma_m$ in $S$ using multiple threads [S1]. It finds the index $m$ of the minimum $f(x_k + \gamma_m d_k)$, and $\gamma_m$ is the chosen step size [S1]. Unlike the sequential version which terminates early if the condition is not met, the parallel version always evaluates all candidate points and picks the one with the least objective function value [S1]. The number of threads should not exceed the number of processor cores to avoid performance impact from sharing cores; parallel search is only beneficial when objective function evaluation is more expensive than thread overhead [S1].

## Operating characteristics
- **Time Complexity:** The sequential version performs up to $M$ evaluations of the objective function $f$. The parallel version performs exactly $M$ evaluations in parallel.
- **Memory Complexity:** Not given in the sources.
- **Measured Performance:** Compared to standard gradient descent, line search takes many fewer iterations to converge to a desired target [S1]. The parallel version accelerates optimization by exploiting multi-core processors [S1].
- **Scaling:** Parallel line search is beneficial only when the evaluation of the objective function is more expensive than the overhead of parallel processing [S1]. In the cited application, the objective function is particularly expensive as it requires computing visibility fields, which in turn requires a pass of slice-based volume rendering [S1].
- **Strengths:** Descends much faster than standard gradient descent; avoids convergence deterioration of exact methods; exploits multi-core processors [S1].
- **Limitations:** Parallel version has overhead from thread management; sequential version may still require multiple function evaluations per iteration [S1].
- **When to use:** When objective function evaluation is expensive (e.g., involves volume rendering passes) and convergence speed is critical [S1].
- **When not to use:** When objective function evaluation is cheap, as the overhead of line search (especially parallel) may not be justified [S1].

## Background
Originated in the context of transfer function optimization for volume rendering, specifically for optimizing visibility-weighted saliency [S1]. The method was proposed by Luo and Dingliana in 2017 [S1]. It builds on earlier work on line search methods [Vrahatis et al. 2000] and addresses issues with exact line search convergence [Zhou et al. 2006] [S1].

## Sources
[S1] Transfer function optimization based on a combined model of visibility and saliency, https://doi.org/10.1145/3154353.3154357, fetched 2026-11-01
