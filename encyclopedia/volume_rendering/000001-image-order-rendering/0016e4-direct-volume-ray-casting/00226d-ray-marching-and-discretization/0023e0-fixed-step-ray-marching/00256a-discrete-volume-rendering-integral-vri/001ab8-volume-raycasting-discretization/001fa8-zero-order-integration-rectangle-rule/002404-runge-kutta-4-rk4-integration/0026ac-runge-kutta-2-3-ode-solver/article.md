# Runge-Kutta 2/3 ODE Solver
Also called: RK23, Runge-Kutta 23

## Overview
A 2nd/3rd order Runge-Kutta method for solving ODEs with adaptive step size control [S1, unofficial]. It sets itself apart from its parent [Runge-Kutta 4 (RK4) Integration](3158-runge-kutta-4-rk4-integration.md) by using a lower-order embedded pair to estimate local truncation error, enabling dynamic step size adjustment rather than fixed steps [G]. It is distinct from the [Hybrid ODE Solver for Ray Trajectory](#hybrid-ode-solver-for-ray-trajectory) variant, which selectively falls back to Euler integration in specific geometric regimes [S2].

## How it works
**Inputs:**
- $y$: Current state vector (position/velocity).
- $w$: Additional solver state/weights.
- $force$: Function evaluating the derivative $f(t, y)$.
- $t$: Current time.
- $t\_span$: Target end time.
- $rtol$: Relative tolerance (default $10^{-2}$) [S1, unofficial].
- $atol$: Absolute tolerance (default $10^{-4}$) [S1, unofficial].
- $hmax$: Maximum step size (default $0.1 \times t\_span$) [S1, unofficial].

**Outputs:**
- $y$: Updated state at $t\_span$.
- $t$: Updated time.

**Data Structures:**
- State vector $y$ and derivative vectors $k_i$ share the same dimension as the ODE system [G].
- Scalars $h, t, er, r$ are floating-point numbers [G].

**Algorithm:**
1. Compute initial slope: $k_1 = force(y)$ [S1, unofficial].
2. Estimate initial error magnitude $r$ using $k_1$ and $rtol$ [S1, unofficial].
3. Initialize step size: $h = 0.8 \times rtol^{1/3} / r$ [S1, unofficial].
4. Loop while $t < t\_span$:
    a. Clamp $h$ to $[h_{min}, h_{max}]$, where $h_{min} = 16 \times \text{FLOAT\_MIN} \times |t|$ [S1, unofficial].
    b. If $1.1 \times h \ge t\_span - t$, set $h = t\_span - t$ [S1, unofficial].
    c. Compute intermediate slopes:
       $$k_2 = force(y + \frac{h}{2} k_1)$$
       $$k_3 = force(y + \frac{3h}{4} k_2)$$
    d. Compute next state estimate (3rd order):
       $$y_{next} = y + \frac{2h}{9} k_1 + \frac{h}{3} k_2 + \frac{4h}{9} k_3$$
    e. Compute final slope for error estimation:
       $$k_4 = force(y_{next})$$
    f. Compute error estimate vector $e$:
       $$e = \frac{h}{72.0} (-5k_1 + 6k_2 + 8k_3 - 9k_4)$$
    g. Compute relative error $er$ using $e$, $\max(|y|, |y_{next}|)$, and $rtol$ [S1, unofficial].
    h. If $er \le rtol$:
       i. Accept step: $t \leftarrow t + h$, $y \leftarrow y_{next}$, $k_1 \leftarrow k_4$ [S1, unofficial].
       ii. Update step size: $h \leftarrow h \times \min(5.0, 0.8 \times (rtol/er)^{1/3})$ [S1, unofficial].
    i. If $h \le h_{min}$, break loop [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import sys

def compute_relative_error(e, y_max):
    # Computes the maximum relative error magnitude across vector elements
    return max(abs(e_i) / y_max_i for e_i, y_max_i in zip(e, y_max))

def rk23_solve(y, force, t_span, rtol=1e-2, atol=1e-4, hmax=None):
    t = 0.0
    hmax = hmax if hmax is not None else 0.1 * t_span
    
    # Initial slope
    k1 = force(y)
    
    # Estimate initial error to set initial step size
    y_max_init = max(abs(y))
    r = compute_relative_error(k1, y_max_init)
    if r == 0:
        r = 1e-12  # Guard against division by zero [G]
    
    h = 0.8 * (rtol ** (1/3)) / r
    
    while t < t_span:
        # Clamp step size
        h_min = 16 * sys.float_info.min * abs(t)
        h = max(h_min, min(h, hmax))
        
        # Check if step exceeds remaining time
        if 1.1 * h >= t_span - t:
            h = t_span - t
            
        # Compute stages
        k2 = force(y + (h / 2.0) * k1)
        k3 = force(y + (3.0 * h / 4.0) * k2)
        
        # 3rd order solution
        y_next = y + (2.0 * h / 9.0) * k1 + (h / 3.0) * k2 + (4.0 * h / 9.0) * k3
        
        # 4th stage for error estimation
        k4 = force(y_next)
        
        # Error estimate vector
        e = (h / 72.0) * (-5 * k1 + 6 * k2 + 8 * k3 - 9 * k4)
        
        # Relative error check
        y_max = max(max(abs(y)), max(abs(y_next)))
        er = compute_relative_error(e, y_max)
        
        if er <= rtol:
            # Accept step
            t += h
            y = y_next
            k1 = k4
            
            # Update step size
            h = h * min(5.0, 0.8 * (rtol / er) ** (1/3))
            
        if h <= h_min:
            break
            
    return y, t
```

**Edge Cases and Pitfalls:**
- **Precision:** $h_{min}$ is computed as $16 \times \text{FLOAT\_MIN} \times |t|$ [S1, unofficial].
- **Boundaries:** The check $1.1 \times h \ge t\_span - t$ prevents overshooting the integration interval [S1, unofficial].
- **Degenerate Input:** If $r=0$ during initialization, division by zero occurs [G].

## Subtopics

### Hybrid ODE Solver for Ray Trajectory
A hybrid integration strategy that uses an adaptive Runge-Kutta integrator when the sphere radius $S(\hat{x})$ is large compared to the error threshold $\hat{\epsilon}$, and falls back to simple Euler integration when $S(\hat{x})$ is small to reduce overhead [S2]. It evaluates RKF45, DP54, and BS23, selecting BS23 for its balance of accuracy (minimum four evaluations per step) and performance [S2]. If $S(\hat{x}) < a \cdot \hat{\epsilon}$ (where $a=3$), it falls back to forward Euler [S2]. Implicit solvers are avoided because the equations are not stiff, constant speed prevents runaway error, and implicit solvers incur difficult-to-control performance overhead compared to BS23's small minimum overhead [S2].

## Operating characteristics
- **Time Complexity:** Adaptive; depends on the number of function evaluations required to meet tolerance [S2]. The BS23 variant requires a minimum of four evaluations per step [S2].
- **Memory Complexity:** $O(d)$ where $d$ is the dimension of the state vector, storing $k_1, k_2, k_3, k_4$ and current state [G].
- **Strengths:** Adaptive step size allows for efficiency in smooth regions and accuracy in complex regions [G]. BS23 offers a good accuracy/performance balance for ray tracing applications [S2].
- **Limitations:** Overhead of error estimation and step size adjustment [G]. Not suitable for stiff equations without implicit variants [S2].
- **When to use:** When solving ODEs for ray trajectories where computational cost must be balanced against visual accuracy [S2].

## Background
The method originates from the Bogacki-Shampine scheme, a 2nd/3rd order Runge-Kutta method [S2]. It is used in non-linear sphere tracing for rendering deformed signed distance fields to integrate ray trajectories [S2]. The algorithm steps and parameter defaults follow the specification provided in the source notes [S1, unofficial].

## Sources
[S1] tink-expo/ntsl-graphics, https://github.com/tink-expo/ntsl-graphics, fetched 2026-10-26 (unofficial)
[S2] Non-linear sphere tracing for rendering deformed signed distance fields, https://doi.org/10.1145/3355089.3356502, fetched 2026-10-29
