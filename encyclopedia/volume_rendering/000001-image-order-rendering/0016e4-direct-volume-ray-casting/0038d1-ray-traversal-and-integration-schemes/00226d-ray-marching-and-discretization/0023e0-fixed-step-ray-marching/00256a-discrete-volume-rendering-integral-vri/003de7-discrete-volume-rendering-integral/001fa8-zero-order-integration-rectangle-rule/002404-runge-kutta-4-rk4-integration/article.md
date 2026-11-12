# Runge-Kutta 4 (RK4) Integration

Also called: Fourth-order Runge-Kutta, RK4 Integrator, Runge-Kutta 4 Integrator, Runge-Kutta 4th order

## Overview
Runge-Kutta 4 (RK4) is a fourth-order numerical integration scheme used to solve initial value problems for ordinary differential equations (ODEs) in volume rendering, specifically for computing streamlines and particle trajectories in vector fields [S1, unofficial][S2, unofficial]. It solves the problem of trajectory accuracy by querying the vector field at four strategically chosen positions within a step and combining them via a weighted sum, significantly reducing integration error compared to first-order methods like [Euler Integration](3836-euler-integration.md) [S1, unofficial]. Unlike [Zero Order Integration (Rectangle Rule)](2082-zero-order-integration-rectangle-rule.md) which approximates the signal as constant, RK4 approximates the derivative (velocity) as varying linearly/quadratically within the step, allowing for larger step sizes while maintaining visual fidelity [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Current position $x_n$ (or $x_i$), step size $\delta$ (or $dt$), and optionally a time parameter $t_{\text{func}}$ for unsteady fields.
*   **Output:** Next position $x_{n+1}$ (or $x_{i+1}$).
*   **Vector Field:** A function $v(x, t)$ or $\nu(x, t)$ that returns the vector at position $x$ and time $t$.

**Data Structures:**
*   Positions and vectors are typically represented as 3D or 4D floating-point vectors (e.g., `double3`, `float4`).
*   The vector field is accessed via sampling (e.g., texture lookup in GPU implementations or function evaluation in CPU implementations).

**Parameters:**
*   **Step size ($\delta$ or $dt$):** Controls the distance traveled per integration step. In real-time applications, this is often dynamic to maintain frame rate [S2, unofficial].
*   **Time ($t_{\text{func}}$):** Required for unsteady vector fields. For steady fields, this argument is ignored or fixed [S2, unofficial].

**Algorithm Steps:**
1.  Calculate half-step size: $dt_{half} = dt \cdot 0.5$ (or $\delta/2$).
2.  Compute the first slope $k_1$ (or $v_0$) at the current position:
    $$k_1 = dt \cdot v(x_n)$$
3.  Compute the second slope $k_2$ (or $v_1$) at the midpoint, using $k_1$ to estimate the position:
    $$k_2 = dt \cdot v(x_n + k_1/2)$$
4.  Compute the third slope $k_3$ (or $v_2$) at the midpoint, using $k_2$ to estimate the position:
    $$k_3 = dt \cdot v(x_n + k_2/2)$$
5.  Compute the fourth slope $k_4$ (or $v_3$) at the end of the step, using $k_3$ to estimate the position:
    $$k_4 = dt \cdot v(x_n + k_3)$$
6.  Compute the weighted average of the slopes to determine the step displacement:
    $$\Delta x = \frac{1}{6} (k_1 + 2k_2 + 2k_3 + k_4)$$
7.  Update the position:
    $$x_{n+1} = x_n + \Delta x$$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def rk4_integrate(pos: Vector3, dt: float, vector_field_func: Callable[[Vector3, float], Vector3], t_func: float = 0.0) -> Vector3:
    """
    Performs one step of Runge-Kutta 4 integration.
    
    Args:
        pos: Current position (x_n).
        dt: Step size (delta).
        vector_field_func: Function v(x, t) returning the vector at position x and time t.
        t_func: Time parameter for unsteady fields. Ignored for steady fields.
        
    Returns:
        The next position (x_{n+1}).
    """
    dt_half = dt * 0.5
    dt_sixth = dt / 6.0
    
    # k1 = dt * v(x_n)
    k1 = dt * vector_field_func(pos, t_func)
    
    # k2 = dt * v(x_n + k1/2)
    pos_mid1 = pos + k1 * dt_half
    k2 = dt * vector_field_func(pos_mid1, t_func)
    
    # k3 = dt * v(x_n + k2/2)
    pos_mid2 = pos + k2 * dt_half
    k3 = dt * vector_field_func(pos_mid2, t_func)
    
    # k4 = dt * v(x_n + k3)
    pos_end = pos + k3 * dt
    k4 = dt * vector_field_func(pos_end, t_func)
    
    # x_{n+1} = x_n + (k1 + 2*k2 + 2*k3 + k4) / 6
    step = (k1 + 2.0 * k2 + 2.0 * k3 + k4) * dt_sixth
    next_pos = pos + step
    
    return next_pos
```

**Edge Cases and Pitfalls:**
*   **Steady vs. Unsteady:** For steady vector fields, the time argument $t_{\text{func}}$ must be handled consistently (e.g., set to a fixed value or ignored) to avoid unnecessary computation or errors [S2, unofficial].
*   **Precision:** The method is sensitive to the precision of the vector field sampling. In GPU implementations, texture interpolation modes (linear vs. nearest) affect the smoothness of the resulting trajectories [S1, unofficial].
*   **Vortex Behavior:** RK4 tends to stay within vortices better than Euler integration, but it may sometimes converge to the center of vortices rather than escaping them, which can be a visual artifact or a desired physical property depending on the context [S1, unofficial].

## Subtopics

### [Euler Integration](3836-euler-integration.md)
Euler Integration is a first-order numerical integration scheme used to solve initial value problems for ordinary differential equations (ODEs) by linearizing the Taylor series expansion. In the context of volume rendering and vector field visualization, it serves as the simplest method to compute streamlines or particle trajectories by stepping along the vector field using the current position and field value. It is distinguished from its sibling [Runge-Kutta 4 (RK4) Integration](3158-runge-kutta-4-rk4-integration.md) by its single-sample-per-step approach, which results in lower accuracy ($O(dt)$ global error) but significantly lower computational cost per step.

### [Runge-Kutta 2/3 ODE Solver](3839-runge-kutta-2-3-ode-solver.md)
A 2nd/3rd order Runge-Kutta method for solving ODEs with adaptive step size control. It sets itself apart from its parent [Runge-Kutta 4 (RK4) Integration](3158-runge-kutta-4-rk4-integration.md) by using a lower-order embedded pair to estimate local truncation error, enabling dynamic step size adjustment rather than fixed steps [G]. It is distinct from the [Hybrid ODE Solver for Ray Trajectory](#hybrid-ode-solver-for-ray-trajectory) variant, which selectively falls back to Euler integration in specific geometric regimes.

### Tensorline Integration
Tensorline Integration uses RK4 to trace lines in focus regions of a volume to visualize directional information from tensor fields [S3, book]. Seed points are randomly placed inside the masked volume (focus region) [S3, book]. Integration starts at these seed points using a fourth-order Runge-Kutta scheme [S3, book]. Integration stops when the line runs into an isotropic region [S3, book]. A key pitfall is that if the tensor is isotropic, the tensorline integration terminates immediately or does not provide directional information, so the method explicitly handles this by stopping integration [S3, book].

## Operating characteristics
**Time and Memory Complexity:**
*   **Time Complexity:** RK4 requires four vector field evaluations per step, making it approximately 4 times slower than Euler integration [S1, unofficial].
*   **Memory Complexity:** Minimal additional memory is required beyond storing the current position and temporary slope vectors.

**Measured Performance:**
*   RK4 provides an order of magnitude smaller error compared to Euler for the same step size [S1, unofficial].
*   RK4 allows for significantly larger time steps (e.g., $dt=1$) while maintaining lower error than Euler with smaller steps (e.g., $dt=0.5$) [S1, unofficial].

**Scaling:**
*   Performance scales linearly with the number of integration steps.
*   In real-time applications, the step size $\delta$ is often dynamically adjusted to maintain frame rate, balancing accuracy and performance [S2, unofficial].

**Strengths and Limitations:**
*   **Strengths:** High accuracy, stability in complex flow fields (e.g., vortices), and ability to use larger step sizes than lower-order methods [S1, unofficial].
*   **Limitations:** Higher computational cost per step compared to Euler. Potential for converging to vortex centers instead of following the flow, which may be undesirable in some visualizations [S1, unofficial].

**When to Use:**
*   Use RK4 when high accuracy is required and computational resources allow for 4x the cost of Euler integration.
*   Use when visualizing complex flow structures like vortices where lower-order methods fail to stay within the flow.

**When Not to Use:**
*   Use Euler integration when real-time performance is critical and lower accuracy is acceptable.
*   Use adaptive step-size methods (like RK2/3) when the vector field varies significantly in magnitude or direction, requiring dynamic step adjustment.

## Background
RK4 was developed as part of the broader Runge-Kutta family of methods for solving ODEs. In the context of volume rendering and vector field visualization, it has been widely adopted for its balance of accuracy and performance. Marek Fiser implemented RK4 in CUDA for real-time visualization of 3D vector fields, noting its superior error characteristics compared to Euler integration [S1, unofficial]. Dirk Sowinski's work on real-time smoke volume rendering also utilized RK4 for unsteady vector fields, emphasizing the need to synchronize step size with function time for real-time behavior [S2, unofficial]. The method is also used in stress tensor field visualization, such as in the "Tensorlines" approach for analyzing stress tensor fields [S3, book].

## Sources
[S1] Realtime visualization of 3D vector fields using CUDA, https://marekfiser.com/projects/realtime-visualization-of-3D-vector-fields-using-CUDA/, fetched 2026-10-28 (unofficial)
[S2] Real-Time Smoke Volume Rendering, http://v22015081501727525.yourvserver.net/webpage/dirkfiles/betreutearbeiten/Sowinski_MA_2012.pdf, fetched 2026-10-31 (unofficial)
[S3] A Visual Approach to Analysis of Stress Tensor Fields, https://doi.org/10.4230/dfu.vol2.sciviz.2011.188, fetched 2026-10-26 (book)
