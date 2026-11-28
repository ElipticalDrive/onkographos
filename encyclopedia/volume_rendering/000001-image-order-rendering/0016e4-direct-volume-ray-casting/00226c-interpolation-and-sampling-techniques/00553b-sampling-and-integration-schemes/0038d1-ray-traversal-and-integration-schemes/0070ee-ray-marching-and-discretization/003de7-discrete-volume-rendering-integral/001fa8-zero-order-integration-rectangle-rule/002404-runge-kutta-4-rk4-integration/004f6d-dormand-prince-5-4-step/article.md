# Dormand-Prince 5(4) Step
Also called: DP5(4), DP5(4) ODE Step, Dormand-Prince 5(4) method, Runge-Kutta-Fehlberg 7(8) Variant, dormandPrinceStep

## Overview
The Dormand-Prince 5(4) step is an explicit Runge-Kutta method providing 5th-order accuracy with embedded 4th-order error estimation, used for integrating ordinary differential equations (ODEs) in volume rendering [S1, unofficial]. It sets itself apart from its sibling [Runge-Kutta 4 (RK4) Integration](3158-runge-kutta-4-rk4-integration.md) by computing an embedded error estimate to control step acceptance, whereas RK4 uses a fixed step size [G].

## How it works
**Inputs and Outputs:**
- `rhs`: A function representing the right-hand side of the ODE $y' = f(t, y)$, returning the derivative vector.
- `y`: The current state vector (position in the volume field).
- `h`: The step size (can be negative for backward integration) [S1, unofficial].
- Output: A `DPStepResult` object containing:
  - `ok`: Boolean flag indicating success or failure.
  - `y_new`: The updated state vector (if `ok` is true).
  - `failedStage`: The index of the stage that failed (if `ok` is false) [S1, unofficial].

**Data Structures:**
- The method uses the FSAL (First Same As Last) property: the first stage derivative $k_0$ is carried over from the previous step's final derivative $k_{\text{last}}$, which equals $f(y_{\text{new}})$, avoiding a redundant function evaluation [S1, unofficial].
- No additional complex data structures are required beyond standard vectors for the state and stage derivatives [S1, unofficial].

**Numbered Steps:**
1. **Initialize Stage 0:** If this is the first step, compute $k_0 = \text{rhs}(y)$. Otherwise, set $k_0$ to the final derivative from the previous step [S1, unofficial].
2. **Evaluate Stages 1–5:** Compute the remaining five stage derivatives using the standard Dormand-Prince coefficients and the previously computed $k$ values [G].
3. **Check RHS Validity:** If the RHS function returns null or an invalid value during any stage evaluation, the step fails immediately at that stage [S1, unofficial].
4. **Compute Solution:** Calculate the new state $y_{\text{new}}$ using the weighted sum of the stage derivatives and the step size $h$ [G].
5. **Error Estimation:** Compute the local truncation error estimate using the difference between the 5th-order and 4th-order solutions [G].
6. **Check Acceptance:** Compare the error estimate against the `ONE_STEP_TRUNCATION` threshold (default $10^{-9}$) [S1, unofficial].
7. **Return Result:** If the error is within bounds, return `DPStepResult(ok=true, y_new=y_{\text{new}})`. Otherwise, return `DPStepResult(ok=false, failedStage=stage_index)` [S1, unofficial].

**Formulas:**
- Local truncation error is $O(h^6)$ [S1, unofficial].
- Global error converges at 5th order ($\propto h^5$) [S1, unofficial].
- Halving step size $h$ reduces global error by $\ge 24\times$ (theoretical $2^5 = 32\times$) [S1, unofficial].

**Parameters:**
- `ONE_STEP_TRUNCATION`: Expected truncation error bound for one step. Value: $10^{-9}$ [S1, unofficial].
- `ENERGY_DRIFT_1000_STEPS`: Expected energy drift bound for 1000 steps. Value: $10^{-6}$ [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Stage Failures:** If the RHS function returns null or invalid data during any stage evaluation, the step fails. The method reports the failing stage index in `failedStage` [S1, unofficial].
- **Precision:** One step of $y' = -y$ over $h=0.1$ lands $\sim 10^{-10}$ from the analytic $e^{-h}$ [S1, unofficial].
- **Drift:** 1000 steps of a harmonic oscillator ($x'=v, v'=-x$) with $h=0.05$ drift $\sim 10^{-7}$ in energy [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class DPStepResult:
    def __init__(self, ok, y_new=None, failed_stage=None):
        self.ok = ok
        self.y_new = y_new
        self.failed_stage = failed_stage

def dormandPrinceStep(rhs, y, h, k0=None):
    # FSAL: k0 is carried over from the previous step's final derivative
    if k0 is None:
        k0 = rhs(y)
        if k0 is None:
            return DPStepResult(ok=False, failed_stage=0)

    # Standard Dormand-Prince 5(4) Butcher tableau coefficients [G]
    # Stage 2
    k1 = rhs(y + h * (1/5 * k0))
    if k1 is None: return DPStepResult(ok=False, failed_stage=1)
    # Stage 3
    k2 = rhs(y + h * (3/40 * k0 + 9/40 * k1))
    if k2 is None: return DPStepResult(ok=False, failed_stage=2)
    # Stage 4
    k3 = rhs(y + h * (44/45 * k0 - 56/15 * k1 + 32/9 * k2))
    if k3 is None: return DPStepResult(ok=False, failed_stage=3)
    # Stage 5
    k4 = rhs(y + h * (19372/6561 * k0 - 25360/2187 * k1 + 64443/6561 * k2 - 212/729 * k3))
    if k4 is None: return DPStepResult(ok=False, failed_stage=4)
    # Stage 6
    k5 = rhs(y + h * (9017/3168 * k0 - 355/33 * k1 + 46732/5247 * k2 + 49/176 * k3 - 5103/18656 * k4))
    if k5 is None: return DPStepResult(ok=False, failed_stage=5)

    # 5th-order solution [G]
    # Weights: [25/216, 0, 1408/2565, 2197/4104, 1/5, 0]
    y_new = y + h * (25/216 * k0 + 1408/2565 * k2 + 2197/4104 * k3 + 1/5 * k4)

    # Error estimation uses the embedded 4th-order pair [G]
    # 4th-order weights: [5179/57600, 0, 1525/16587, 3759/4104, 125/1152, -1/4]
    y_new_4th = y + h * (5179/57600 * k0 + 1525/16587 * k2 + 3759/4104 * k3 + 125/1152 * k4 - 1/4 * k5)
    error = abs(y_new - y_new_4th)

    # Check Acceptance
    if error > 1e-9: # ONE_STEP_TRUNCATION
        return DPStepResult(ok=False, failed_stage=5) # Step rejected

    # Store k5 for FSAL in the next step
    # In practice, the caller stores k5 and passes it as k0 in the next call
    return DPStepResult(ok=True, y_new=y_new)
```

## Subtopics

### [Euler Integration](3836-euler-integration.md)
A first-order explicit method that computes the next state using a single derivative evaluation. Choose it when computational resources are extremely limited and low accuracy is acceptable.

### [Runge-Kutta 2/3 ODE Solver](3839-runge-kutta-2-3-ode-solver.md)
A third-order method with embedded second-order error estimation requiring fewer RHS evaluations than DP5(4). Choose it when a balance between accuracy and computational cost is needed.

### [Tensorline Integration](9613-tensorline-integration.md)
Integrates along higher-order tensor fields rather than vector fields. Choose it when dealing with tensor data, as it employs integration methods like Euler's method with adaptive stepsize control, second order Runge-Kutta, and dopri5/RK45 to prevent flipping of eigenvector directions in regions of high isotropy.

### Integration Methods for Eigenvector Interpolation
Integration methods used to get correct interpolation of eigenvectors and prevent flipping of eigenvector directions in regions of high isotropy. Implemented methods include Euler’s method with adaptive stepsize control, second order Runge-Kutta, and dopri5/RK45 [S2].

## Operating characteristics
- **Time Complexity:** Each step requires 6 evaluations of the RHS function [S1, unofficial].
- **Memory Complexity:** Requires storage for the state vector and stage derivatives [S1, unofficial].
- **Measured Performance:** One step of $y' = -y$ over $h=0.1$ lands $\sim 10^{-10}$ from the analytic $e^{-h}$ [S1, unofficial]. 1000 steps of a harmonic oscillator ($x'=v, v'=-x$) with $h=0.05$ drift $\sim 10^{-7}$ in energy [S1, unofficial].
- **Scaling:** Halving step size $h$ reduces global error by $\ge 24\times$ (theoretical $2^5 = 32\times$) [S1, unofficial].
- **Strengths:** High accuracy (5th-order) with adaptive step control via embedded error estimation.
- **Limitations:** Higher computational cost per step compared to lower-order methods due to 6 RHS evaluations.

## Background
Not given in the sources.

## Sources
[S1] rusaitis/webpic, https://github.com/rusaitis/webpic, fetched 2026-10-27 (unofficial)
[S2] HOT- Lines: Tracking Lines in Higher Order Tensor Fields, https://doi.org/10.1109/vis.2005.48, fetched 2026-10-27
