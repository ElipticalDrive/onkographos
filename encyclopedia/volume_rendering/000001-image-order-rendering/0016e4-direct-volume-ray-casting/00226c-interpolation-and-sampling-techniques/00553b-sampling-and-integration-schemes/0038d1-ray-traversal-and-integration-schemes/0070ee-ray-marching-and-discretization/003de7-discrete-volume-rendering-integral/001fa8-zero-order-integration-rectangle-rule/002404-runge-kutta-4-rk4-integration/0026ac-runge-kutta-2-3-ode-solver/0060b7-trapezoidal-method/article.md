# Trapezoidal Method

Also called: Euler-Trapezoidal, Predictor-corrector Trapezoidal, Quadrature Warp, V^q

## Overview
The Trapezoidal Method is a second-order implicit predictor-corrector technique used for ray marching and integration in volume rendering [S2]. It solves the ordinary differential equation (ODE) governing light transport by combining the speed of an explicit Euler predictor with the accuracy of an implicit trapezoidal corrector [S2]. Unlike explicit methods, it can overcome black hole singularities caused by point lights, though it may fail to converge on non-stiff equations with large step sizes [S2].

## How it works
**Inputs and Outputs:**
- Input: Current state $y_i$ (accumulated color/transmittance), current position $t_i$, step size $s$, and derivative function $f(y, t)$ (typically density or absorption coefficient) [S2].
- Output: Next state $y_{i+1}$ [S2].

**Data Structures:**
- Scalar values for $y_i, y_{i+1}, t_i, t_{i+1}, s$ [S2].
- Function $f$ evaluating the integrand at a given position and state [S2].

**Steps:**
1. **Predictor Step:** Estimate $y_{i+1}$ using Euler's method to provide a fast initial guess [S2].
   $$ y_{i+1}^{(0)} = y_i + s \cdot f(y_i, t_i) $$
2. **Corrector Step (Fixed-Point Iteration):** Solve the implicit trapezoidal equation using the predictor as the initial value for iteration [S2].
   $$ y_{i+1}^{(k+1)} = y_i + \frac{s}{2} \left( f(y_i, t_i) + f(y_{i+1}^{(k)}, t_i + s) \right) $$
   Iterate until convergence or a maximum number of iterations is reached [S2].

**Formula:**
The core implicit equation is:
$$ y_{i+1} = y_i + \frac{s}{2} (f(y_i, t_i) + f(y_{i+1}, t_i + s)) $$
Where:
- $y_{i+1}$ is the unknown next state [S2].
- $y_i$ is the current state [S2].
- $s$ is the step size [S2].
- $f$ is the derivative function (e.g., density) [S2].
- $t_i$ is the current position along the ray [S2].

**Parameters:**
- Step size $s$: Determined by adaptive step control or fixed marching [S2].
- Convergence tolerance: Not specified in sources [S2].

**Edge Cases and Pitfalls:**
- **Convergence Failure:** The method may not converge, particularly when the equation is not stiff or when step sizes are too large relative to the tolerance [S2].
- **Singularities:** Implicit nature allows it to handle singularities (e.g., point lights) that break explicit methods [S2].
- **Bias:** In the context of differentiable rendering, trapezoidal quadrature is a biased estimator of the integral, though it becomes consistent at silhouettes [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def trapezoidal_method(y_i, t_i, s, f, max_iter=10, tol=1e-6):
    """
    Perform one step of the Euler-Trapezoidal method.
    
    Args:
        y_i: Current accumulated value (scalar or vector).
        t_i: Current position along the ray.
        s: Step size.
        f: Function returning the derivative (e.g., density) at (y, t).
        max_iter: Maximum fixed-point iterations.
        tol: Convergence tolerance.
        
    Returns:
        y_i+1: The updated accumulated value.
    """
    # Predictor: Euler's method
    y_pred = y_i + s * f(y_i, t_i)
    
    # Corrector: Fixed-point iteration
    y_next = y_pred
    for k in range(max_iter):
        # Evaluate derivative at current estimate
        f_next = f(y_next, t_i + s)
        f_current = f(y_i, t_i)
        
        # Update estimate using trapezoidal rule
        y_new = y_i + (s / 2.0) * (f_current + f_next)
        
        # Check convergence
        if abs(y_new - y_next) < tol:
            return y_new
        y_next = y_new
        
    # Return last estimate if not converged
    return y_next
```

## Subtopics

### Adaptive Step Control
Estimates error by comparing solutions of different orders to adjust step size [S2].

### Quadrature Warp ($\mathcal{V}^q$)
A specific application in differentiable rendering where trapezoidal quadrature is used to define a continuous warp function that satisfies boundary consistency at silhouettes [S1].

## Operating characteristics
- **Time Complexity:** Slower than explicit methods due to fixed-point iteration, but faster than pure implicit methods because the Euler predictor provides a good initial guess [S2].
- **Memory Complexity:** Low, requiring storage for current and next state values [S2].
- **Performance:** Similar to Order Two Runge-Kutta (RK2) [S2].
- **Strengths:** Handles black hole singularities from point lights better than explicit methods [S2].
- **Limitations:** May fail to converge on non-stiff equations or with large step sizes [S2]. Biased estimator in general cases, though consistent at boundaries [S1].
- **When to Use:** When handling singularities is critical and step sizes are controlled [S2].
- **When Not to Use:** For non-stiff problems where explicit methods are sufficient and faster [S2].

## Background
The method was tested in the context of differential ray marching, specifically the "Euler-Trapezoidal" variant, which uses an explicit Euler step as a predictor for the implicit trapezoidal corrector [S2]. It is also referenced in differentiable rendering of neural SDFs as a quadrature rule for warp estimation [S1].

## Sources
[S1] Differentiable Rendering of Neural SDFs through Reparameterization, https://doi.org/10.1145/3550469.3555397, fetched 2026-10-26
[S2] Differential Ray Marching, https://doi.org/10.2312/localchapterevents/ceig/ceig12/091-098, fetched 2026-10-26
