# Runge-Kutta 4 (RK4) Integration

Also called: Fourth-order Runge-Kutta, RK4 Integrator, Runge-Kutta 4 Integrator, Runge-Kutta 4th order

## Overview
Runge-Kutta 4 (RK4) is a higher-order numerical integration scheme for volume rendering that approximates the trajectory of a point in a vector field by querying the field at four distinct positions and combining them into a single weighted step [S1, unofficial][S2, unofficial]. It solves the initial value problem for ray traversal more accurately than the parent [Zero Order Integration (Rectangle Rule)](2082-zero-order-integration-rectangle-rule.md), significantly reducing integration error and preventing rays from escaping vortices or complex flow structures, at the cost of increased computational load [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Current position $x_n$ (or $x_i$), step size $\theta$ (or $dt$), and optionally a function time $t_{\text{func}}$ for unsteady fields.
*   **Output:** Next position $x_{n+1}$ (or $x_{i+1}$).

**Data Structures:**
*   Position: A 3D vector (e.g., `double3` or `float3`).
*   Vector Field Sample: A 3D vector representing the velocity/direction at a specific point. In unsteady fields, the field function also accepts time.

**Parameters:**
*   Step size $\theta$ (or $dt$): Controls the distance advanced per integration step. For real-time applications, this may be dynamic to maintain frame rate [S2, unofficial].
*   Function time $t_{\text{func}}$: Required for unsteady vector fields; must be synchronized with the integration step size $\theta$ [S2, unofficial]. For steady fields, this argument is ignored or set to a fixed value [S2, unofficial].

**Formulas:**
The method computes four intermediate slopes ($k_1, k_2, k_3, k_4$ or $v_0, v_1, v_2, v_3$) and updates the position using a weighted average [S1, unofficial][S2, unofficial]:

$$
\begin{aligned}
k_1 &= dt \cdot v(x_n) \\
k_2 &= dt \cdot v(x_n + k_1/2) \\
k_3 &= dt \cdot v(x_n + k_2/2) \\
k_4 &= dt \cdot v(x_n + k_3) \\
x_{n+1} &= x_n + \frac{k_1}{6} + \frac{k_2}{3} + \frac{k_3}{3} + \frac{k_4}{6} + O(dt^5)
\end{aligned}
$$

Where:
*   $x_n$ is the current position.
*   $x_{n+1}$ is the next position.
*   $dt$ is the time/step size.
*   $v(x)$ is the vector field value at position $x$.
*   $O(dt^5)$ represents the local truncation error [S1, unofficial].

Alternatively, using the notation from [S2, unofficial] where $v_i$ are the sampled vector values (not scaled by $dt$ in the definition of $v$, but scaled in the final sum):

$$
\begin{aligned}
v_0 &= \nu(x_i, t_{\text{func}}) \\
v_1 &= \nu(x_i + \frac{1}{2} \theta v_0, t_{\text{func}}) \\
v_2 &= \nu(x_i + \frac{1}{2} \theta v_1, t_{\text{func}}) \\
v_3 &= \nu(x_i + \theta v_2, t_{\text{func}}) \\
x_{i+1} &= x_i + \frac{1}{6} \theta (v_0 + 2v_1 + 2v_2 + v_3)
\end{aligned}
$$

**Pseudocode:**
The following implementation assumes a steady vector field for clarity, consistent with the primary source [S1, unofficial]. For unsteady fields, the `sampleVectorField` function must accept a time parameter [S2, unofficial].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def rk4_integrate(pos: Vector3, dt: float, sample_vector_field: Callable[[Vector3], Vector3]) -> Vector3:
    """
    Performs one step of Runge-Kutta 4 integration.
    
    Args:
        pos: Current position (x_n).
        dt: Step size.
        sample_vector_field: Function that returns the vector field value v(x) at position x.
        
    Returns:
        The next position (x_{n+1}).
    """
    dt_half = dt * 0.5
    dt_sixth = dt / 6.0
    
    # 1. Sample k1 at current position
    k1 = sample_vector_field(pos)
    
    # 2. Sample k2 at pos + dt/2 * k1
    # Note: k1 is a vector. We scale it by dt_half to get the displacement.
    pos_k2 = pos + dt_half * k1
    k2 = sample_vector_field(pos_k2)
    
    # 3. Sample k3 at pos + dt/2 * k2
    pos_k3 = pos + dt_half * k2
    k3 = sample_vector_field(pos_k3)
    
    # 4. Sample k4 at pos + dt * k3
    pos_k4 = pos + dt * k3
    k4 = sample_vector_field(pos_k4)
    
    # 5. Compute weighted sum: x_{n+1} = x_n + (k1 + 2*k2 + 2*k3 + k4) * dt / 6
    # The k values in the source [S1, unofficial] are defined as dt * v(...).
    # However, the pseudocode in [S1, unofficial] samples v(...) and then multiplies by dt in the final sum.
    # Let's align with the formula: k1 = dt * v(x).
    # If sample_vector_field returns v(x), then:
    
    k1_scaled = dt * k1
    k2_scaled = dt * k2
    k3_scaled = dt * k3
    k4_scaled = dt * k4
    
    next_pos = pos + (k1_scaled + 2.0 * k2_scaled + 2.0 * k3_scaled + k4_scaled) * (1.0 / 6.0)
    
    return next_pos
```

**Edge Cases and Pitfalls:**
*   **Steady vs. Unsteady:** For steady fields, the time argument $t_{\text{func}}$ must be ignored or fixed [S2, unofficial].
*   **Precision:** The method has a local error of $O(dt^5)$ and global error of $O(dt^4)$ [S1, unofficial].
*   **Vortex Behavior:** RK4 tends to stay within vortices where Euler integration escapes, but it may sometimes converge to the center of vortices instead of following the flow correctly [S1, unofficial].

## Subtopics

### Euler Integration
Euler Integration is a first-order numerical integration scheme that linearizes the Taylor series expansion to compute the next position from the current position and the vector field derivative. It is distinguished from RK4 by its simplicity and lower accuracy; while RK4 uses four samples to achieve fourth-order accuracy, Euler uses only one sample, resulting in first-order global error. It is significantly faster but less accurate, often escaping vortices even at small step sizes. See [Euler Integration](3836-euler-integration.md).

### [Runge-Kutta 2/3 ODE Solver](3839-runge-kutta-2-3-ode-solver.md)
A 2nd/3rd order Runge-Kutta method for solving ODEs with adaptive step size control. It sets itself apart from RK4 by using a lower-order embedded pair to estimate local truncation error, enabling dynamic step size adjustment rather than fixed steps [G]. It is distinct from the Hybrid ODE Solver for Ray Trajectory variant, which selectively falls back to Euler integration in specific geometric regimes. See [Runge-Kutta 2/3 ODE Solver](3839-runge-kutta-2-3-ode-solver.md).

### Tensorline Integration
Tensorline Integration uses RK4 to integrate lines starting from seed points randomly placed inside a masked volume (focus region) [S3, book]. The integration stops when the line runs into an isotropic region [S3, book]. A key pitfall is that if the tensor is isotropic, the integration terminates immediately or provides no directional information, which the method explicitly handles by stopping [S3, book].

## Operating characteristics
*   **Time Complexity:** RK4 requires four vector field samples per step, making it approximately 4 times slower than Euler integration [S1, unofficial].
*   **Memory Complexity:** Minimal additional memory beyond the current position and temporary vectors for the four slopes.
*   **Accuracy:** RK4 error is smaller by an order of magnitude compared to Euler [S1, unofficial]. It maintains accuracy even for larger time steps; for example, RK4 with $dt=1$ has less error than Euler with $dt=0.5$ [S1, unofficial].
*   **Strengths:** Significantly better trajectory tracking in complex flows (e.g., vortices) compared to lower-order methods [S1, unofficial].
*   **Limitations:** Higher computational cost than Euler. May converge to vortex centers rather than following the flow in some cases [S1, unofficial].
*   **When to use:** When higher accuracy is required and computational resources allow for 4x the cost of Euler. Preferred for visualizing complex vector field structures where Euler fails to capture topology [S1, unofficial].
*   **When not to use:** When real-time performance is critical and lower accuracy is acceptable (use Euler).

## Background
RK4 was implemented in the context of real-time 3D vector field visualization using CUDA [S1, unofficial]. The author initially used Euler integration but found that lowering the step size did not sufficiently reduce error and hurt performance. RK4 was adopted to achieve better visual fidelity with larger step sizes, reducing the number of steps required while maintaining accuracy [S1, unofficial]. It is also used in real-time smoke volume rendering for unsteady vector fields, where it solves the initial value problem for point trajectories [S2, unofficial]. Additionally, it is applied in tensor field analysis for drawing tensorlines in focus regions [S3, book].

## Sources
[S1] Realtime visualization of 3D vector fields using CUDA, https://marekfiser.com/projects/realtime-visualization-of-3D-vector-fields-using-CUDA/, fetched 2026-10-28 (unofficial)
[S2] Real-Time Smoke Volume Rendering, http://v22015081501727525.yourvserver.net/webpage/dirkfiles/betreutearbeiten/Sowinski_MA_2012.pdf, fetched 2026-10-31 (unofficial)
[S3] A Visual Approach to Analysis of Stress Tensor Fields, https://doi.org/10.4230/dfu.vol2.sciviz.2011.188, fetched 2026-10-26 (book)
