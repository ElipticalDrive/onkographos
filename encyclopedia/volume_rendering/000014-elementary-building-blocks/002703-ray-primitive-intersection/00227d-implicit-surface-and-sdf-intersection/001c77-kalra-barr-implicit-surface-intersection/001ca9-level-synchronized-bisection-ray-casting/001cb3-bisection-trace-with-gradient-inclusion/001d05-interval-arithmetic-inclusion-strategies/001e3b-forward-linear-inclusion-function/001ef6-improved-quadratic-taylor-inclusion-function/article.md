# Improved Quadratic Taylor Inclusion Function

Also called: Hybrid quadratic-linear bound, Improved Quad. Bounds, Improved Quadratic Taylor

## Overview
The Improved Quadratic Taylor Inclusion Function is a method for computing tighter, valid bounds on a scalar field function $f(t)$ along a ray, used to determine safe step sizes in sphere tracing [S1]. It solves the problem where standard quadratic bounds violate the first-order Lipschitz constraint, causing the bound to grow too fast and result in smaller, inefficient step sizes than linear bounds [S1]. It sets itself apart by enforcing both first-order ($\lambda$) and second-order ($\gamma$) Lipschitz constraints simultaneously, switching to a linear bound when the quadratic derivative exceeds the first-order limit [S1].

## How it works
**Inputs and Outputs:**
- Input: Current ray parameter $t_0$, function value $f(t_0)$, first derivative $f'(t_0)$, first-order Lipschitz constant $\lambda$, second-order Lipschitz constant $\gamma$ [S1].
- Output: Inclusion function $f_{\pm}(t)$ providing lower ($f_-$) and upper ($f_+$) bounds for $t > t_0$ [S1].

**Data Structures:**
No complex data structures are required; the method relies on scalar arithmetic operations [S1].

**Formulas:**
The improved inclusion function $f_{\pm}(t)$ is defined piecewise using a quadratic component $q_{\pm}(t)$ and a switch point $t_{\pm}$ [S1]:

$$
q_{\pm}(t) = f(t_0) + (t - t_0) \cdot \left( f'(t_0) \pm \frac{1}{2}\gamma(t - t_0) \right)
$$

$$
t_{\pm} = t_0 + \frac{\pm\lambda - f'(t_0)}{\pm\gamma}
$$

The final bound is:

$$
f_{\pm}(t) = \begin{cases} 
q_{\pm}(t) & \forall t \in [t_0, t_{\pm}] \\
q_{\pm}(t_{\pm}) \pm \lambda \cdot (t - t_{\pm}) & \forall t > t_{\pm}
\end{cases}
$$

**Parameters:**
- $\lambda$: First-order Lipschitz constant [S1].
- $\gamma$: Second-order Lipschitz constant [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def improved_quadratic_inclusion(t0, f_t0, f_prime_t0, lambda_val, gamma_val):
    """
    Computes the improved quadratic inclusion bounds for t > t0.
    
    Args:
        t0: Current ray parameter.
        f_t0: Function value at t0.
        f_prime_t0: First derivative at t0.
        lambda_val: First-order Lipschitz constant.
        gamma_val: Second-order Lipschitz constant.
        
    Returns:
        A function that takes t and returns (lower_bound, upper_bound).
    """
    # Calculate switch points for upper and lower bounds
    # t_plus = t0 + (lambda - f_prime) / gamma
    # t_minus = t0 + (-lambda - f_prime) / (-gamma) = t0 + (lambda + f_prime) / gamma
    
    t_plus = t0 + (lambda_val - f_prime_t0) / gamma_val
    t_minus = t0 + (lambda_val + f_prime_t0) / gamma_val
    
    def bound(t):
        if t <= t0:
            return f_t0, f_t0
            
        # Calculate quadratic components
        # q_plus(t) = f(t0) + (t - t0) * (f'(t0) + 0.5 * gamma * (t - t0))
        # q_minus(t) = f(t0) + (t - t0) * (f'(t0) - 0.5 * gamma * (t - t0))
        
        dt = t - t0
        
        q_plus = f_t0 + dt * (f_prime_t0 + 0.5 * gamma_val * dt)
        q_minus = f_t0 + dt * (f_prime_t0 - 0.5 * gamma_val * dt)
        
        # Determine if we are past the switch point for upper bound
        if t > t_plus:
            # Switch to linear bound
            # f_plus(t) = q_plus(t_plus) + lambda * (t - t_plus)
            dt_switch = t_plus - t0
            q_plus_at_switch = f_t0 + dt_switch * (f_prime_t0 + 0.5 * gamma_val * dt_switch)
            upper = q_plus_at_switch + lambda_val * (t - t_plus)
        else:
            upper = q_plus
            
        # Determine if we are past the switch point for lower bound
        if t > t_minus:
            # Switch to linear bound
            # f_minus(t) = q_minus(t_minus) - lambda * (t - t_minus)
            dt_switch = t_minus - t0
            q_minus_at_switch = f_t0 + dt_switch * (f_prime_t0 - 0.5 * gamma_val * dt_switch)
            lower = q_minus_at_switch - lambda_val * (t - t_minus)
        else:
            lower = q_minus
            
        return lower, upper

    return bound
```

**Edge Cases and Pitfalls:**

## Subtopics

### [Quadratic Taylor Inclusion Functions](1974-quadratic-taylor-inclusion-functions.md)
Standard quadratic bounds that do not enforce the first-order Lipschitz constraint, potentially leading to loose bounds. Use when $\gamma$ is small and $\lambda$ is not a limiting factor.

## Operating characteristics
**Time and Memory Complexity:**
The computational cost is the same as regular quadratic inclusion functions [S1].

**Measured Performance:**
- On the Sponge scene with all enhancements (SS, ES+1): Both Regular Quad and Improved Quad achieve 70.0 avg steps, 651.1 avg octaves, and 16.3 ms time, providing a $\times 4.0$ speedup vs Linear [S1].
- On Sponge (far) scene with all enhancements (SS, ES+1): Regular Quad has 55.8 avg steps, 300.6 avg octaves, 9.8 ms time [S1]. Improved Quad has 55.8 avg steps, 300.6 avg octaves, 9.7 ms time [S1]. Both provide a $\times 1.7$ speedup vs Linear [S1].
- Without enhancements, improved quadratic bounds allow $\times 1.2$ and $\times 1.1$ reduction in steps compared to naive methods, beating linear and regular quadratic bounds, but are slightly slower than linear bounds due to gradient computation overhead [S1].
- When used with enhancements (Adaptive $\gamma$Fn), the improved quadratic bound gives results similar to regular quadratic bounds, with performance gains too small to appear in tables [S1].

**Strengths and Limitations:**
- Strengths: Prevents the quadratic term from growing too fast by enforcing the first-order Lipschitz constraint, yielding bigger step sizes than regular quadratic bounds when the derivative would otherwise exceed $\lambda$ [S1].
- Limitations: Requires gradient computation, making it slightly slower than linear bounds in naive contexts [S1].

**When to Use:**
Use when tighter bounds than linear are needed and the second-order Lipschitz constant $\gamma$ is available, especially when regular quadratic bounds might violate the first-order constraint [S1].

## Background
The Improved Quadratic Taylor Inclusion Function was proposed to address the suboptimal performance of regular quadratic inclusion functions, which can perform worse than linear bounds because their derivative can exceed the first-order Lipschitz constant $\lambda$ [S1]. It was introduced in the paper "Fast sphere tracing of procedural volumetric noise for very large and detailed scenes" [S1].

## Sources
[S1] Fast sphere tracing of procedural volumetric noise for very large and detailed scenes, https://doi.org/10.1111/cgf.70072, fetched 2026-10-26
