# Runge-Kutta 2/3 ODE Solver
Also called: RK23, Runge-Kutta 23

## Overview
A 2nd/3rd order Runge-Kutta method for solving ODEs with adaptive step size control [S1, unofficial]. It sets itself apart from its parent [Runge-Kutta 4 (RK4) Integration](3158-runge-kutta-4-rk4-integration.md) by using a lower-order formulation that requires fewer function evaluations per step, and from its sibling [Dormand-Prince 5(4) Step](13933-dormand-prince-5-4-step.md) by offering a balance of accuracy and performance with a minimum of four evaluations per step [S2].

## How it works
**Inputs:**
- $y$: Current state vector (position/field value).
- $w$: Additional parameters (e.g., time, grid metadata).
- $force$: Function evaluating the derivative $\frac{dy}{dt}$.
- $t$: Current time/parameter.
- $t_{span}$: End time/parameter.
- $rtol$: Relative tolerance (default $10^{-2}$) [S1, unofficial].
- $atol$: Absolute tolerance (default $10^{-4}$) [S1, unofficial].
- $h_{max}$: Maximum step size (default $0.1 \times t_{span}$) [S1, unofficial].

**Outputs:**
- $y_{final}$: State at $t_{span}$.
- $t_{final}$: Final time reached.

**Data Structures:**
- Vectors $k_1, k_2, k_3, k_4$ of same dimension as $y$.
- Scalar $h$: Step size.
- Scalar $e$: Error estimate.
- Scalar $er$: Relative error.

**Steps:**
1. Compute $k_1 = force(y, w)$.
2. Estimate initial error $r$ using $k_1$ and a threshold [S1, unofficial].
3. Initialize step size $h = 0.8 \times rtol^{1/3} / r$ [S1, unofficial].
4. While $t < t_{span}$:
    a. Clamp $h$ to $[h_{min}, h_{max}]$, where $h_{min} = 16 \times \text{FLOAT\_MIN} \times |t|$ [S1, unofficial].
    b. If $1.1 \times h \ge t_{span} - t$, set $h = t_{span} - t$ [S1, unofficial].
    c. Compute $k_2 = force(y + (h/2)k_1, w)$.
    d. Compute $k_3 = force(y + (3h/4)k_2, w)$.
    e. Compute $y_{next} = y + (2h/9)k_1 + (h/3)k_2 + (4h/9)k_3$ [S1, unofficial].
    f. Compute $k_4 = force(y_{next}, w)$.
    g. Compute error estimate $e = (-5k_1 + 6k_2 + 8k_3 - 9k_4) \times h / 72.0$ [S1, unofficial].
    h. Compute relative error $er$ using $e$, $\max(|y|, |y_{next}|)$, and threshold [S1, unofficial].
    i. If $er \le rtol$:
        - Accept step: $t = t + h$, $y = y_{next}$, $k_1 = k_4$.
        - Update $h = h \times \min(5.0, 0.8 \times (rtol/er)^{1/3})$ [S1, unofficial].
    j. If $h \le h_{min}$, break loop [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def rk23_solve(y, w, force, t, t_span, rtol=1e-2, atol=1e-4, h_max=None):
    if h_max is None:
        h_max = 0.1 * t_span
    
    # Initial step setup
    k1 = force(y, w)
    # Estimate initial error r based on k1 and threshold [S1, unofficial]
    r = error_estimate(k1, y, rtol) 
    h = 0.8 * (rtol ** (1/3)) / r
    
    while t < t_span:
        # Clamp step size
        h_min = 16 * float('min') * abs(t) # float('min') is implementation specific smallest positive float
        h = max(h_min, min(h, h_max))
        
        # Adjust for end of span
        if 1.1 * h >= t_span - t:
            h = t_span - t
            
        # Stage evaluations
        k2 = force(y + (h / 2) * k1, w)
        k3 = force(y + (3 * h / 4) * k2, w)
        
        # 3rd order solution
        y_next = y + (2 * h / 9) * k1 + (h / 3) * k2 + (4 * h / 9) * k3
        
        # 2nd order error estimate stage
        k4 = force(y_next, w)
        
        # Error calculation
        e = (-5 * k1 + 6 * k2 + 8 * k3 - 9 * k4) * h / 72.0
        
        # Relative error
        max_val = max(abs(y), abs(y_next))
        er = error_estimate(e, max_val, rtol)
        
        if er <= rtol:
            # Accept step
            t = t + h
            y = y_next
            k1 = k4
            
            # Update step size
            h = h * min(5.0, 0.8 * (rtol / er) ** (1/3))
        else:
            # Reject step, reduce h (implementation detail not fully specified in S1, 
            # but standard practice implies reducing h. S1 only specifies update on accept.
            # We assume a fallback reduction or loop continues with same h? 
            # S1 says "Update h" inside the accept block. 
            # If er > rtol, h is not updated in S1 snippet. 
            # This implies infinite loop if er > rtol and h doesn't change.
            # [G] Standard RK adaptive logic reduces h on rejection. 
            # S1 snippet is incomplete regarding rejection handling.
            # We assume h is reduced here to prevent infinite loop.
            h = h * 0.5 # [G] Heuristic reduction for rejection
            
        if h <= h_min:
            break
            
    return y, t
```

## Subtopics

### Hybrid ODE Solver for Ray Trajectory
A hybrid integration strategy that uses an adaptive Runge-Kutta integrator when the sphere radius $S(\hat{x})$ is large compared to the error threshold $\hat{\epsilon}$, and falls back to simple Euler integration when $S(\hat{x})$ is small to reduce overhead [S2]. It evaluates RKF45, DP54, and BS23, selecting BS23 for its balance of accuracy (minimum four evaluations per step) and performance [S2]. If $S(\hat{x}) < a \cdot \hat{\epsilon}$ (where $a=3$), it falls back to forward Euler [S2]. Implicit solvers are avoided because the equations are not stiff, constant speed prevents runaway error, and implicit solvers incur difficult-to-control performance overhead compared to BS23's small minimum overhead [S2].

### Adaptive Integration
Automatically selects appropriate step sizes $h$ during integration by comparing results from different $h$ or approximation orders to estimate accuracy [S3, unofficial]. It computes $x(t+h)$ for different $h$ or orders; if the error (e.g., angular deviation) exceeds a threshold, it repeats with decreased $h$ [S3, unofficial]. The error is measured as $E(t; h_0, h_1) = 1 - (x(t + h_0) - x(t)) \cdot (x(t + h_1) - x(t))$ [S3, unofficial]. This leads to an increased number of field evaluations but yields higher accuracy in turbulent regions [S3, unofficial].

## Operating characteristics
- **Time Complexity:** Depends on the number of adaptive steps. Each accepted step requires 4 function evaluations [S2].
- **Memory Complexity:** $O(d)$ where $d$ is the dimension of the state vector, storing $k_1, k_2, k_3, k_4$ and intermediate states.
- **Performance:** BS23 requires a minimum of four evaluations per step, which is lower than RKF45 and DP54 [S2].
- **Strengths:** Balance of accuracy and performance; adaptive step size handles varying field complexity [S2][S3, unofficial].
- **Limitations:** Increased field evaluations in turbulent regions compared to fixed-step methods [S3, unofficial].

## Background
The method is based on the Bogacki-Shampine (BS23) scheme [S2]. It is used in volume rendering for ray trajectory integration, particularly in non-linear sphere tracing for rendering deformed signed distance fields [S2].

## Sources
[S1] tink-expo/ntsl-graphics, https://github.com/tink-expo/ntsl-graphics, fetched 2026-10-26 (unofficial)
[S2] Non-linear sphere tracing for rendering deformed signed distance fields, https://doi.org/10.1145/3355089.3356502, fetched 2026-10-29
[S3] Multi-Field Visualization, https://kluedo.ub.rptu.de/files/2291/_diss.pdf, fetched 2026-10-27 (unofficial)
