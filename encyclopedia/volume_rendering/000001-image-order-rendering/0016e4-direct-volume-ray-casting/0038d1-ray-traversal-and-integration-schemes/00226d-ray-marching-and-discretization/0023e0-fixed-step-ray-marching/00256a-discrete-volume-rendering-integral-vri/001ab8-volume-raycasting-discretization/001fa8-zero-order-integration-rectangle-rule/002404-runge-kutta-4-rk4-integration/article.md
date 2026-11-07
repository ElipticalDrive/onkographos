# Runge-Kutta 4 (RK4) Integration

Also called: Fourth-order Runge-Kutta, RK4 Integrator, Runge-Kutta 4 Integrator, Runge-Kutta 4th order

## Overview
Runge-Kutta 4 (RK4) is a higher-order numerical integration method for volume rendering that queries the vector field at four distinct positions to compute a single step, significantly reducing truncation error compared to the zero-order [Rectangle Rule](2082-zero-order-integration-rectangle-rule.md) and first-order [Euler Integration](3839-runge-kutta-2-3-ode-solver.md) methods [S2, unofficial]. It sets itself apart by achieving a local truncation error of $O(dt^5)$ per step, allowing for larger step sizes while maintaining accuracy, at the cost of four vector field samples per step rather than one [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Current position $x_n$ (or $x_i$), step size $dt$ (or $\delta$), and optionally a time parameter $t_{\text{func}}$ for unsteady fields [S2, unofficial][S3, unofficial].
*   **Output:** Next position $x_{n+1}$ (or $x_{i+1}$) [S2, unofficial][S3, unofficial].

**Data Structures:**
*   Position: 3D vector (float/double) [S2, unofficial].
*   Vector Field Sample: 3D velocity vector plus an optional scalar component (e.g., density or weight), returned as a 4D vector in GPU implementations [S2, unofficial].

**Formulas:**
The method computes four intermediate slopes and combines them with specific weights [S2, unofficial]. Using the notation from [S2, unofficial] for steady fields:
$$k_1 = dt \cdot v(x_n)$$
$$k_2 = dt \cdot v(x_n + k_1/2)$$
$$k_3 = dt \cdot v(x_n + k_2/2)$$
$$k_4 = dt \cdot v(x_n + k_3)$$
$$x_{n+1} = x_n + \frac{k_1}{6} + \frac{k_2}{3} + \frac{k_3}{3} + \frac{k_4}{6} + O(dt^5)$$
where $x_n$ is the current position, $x_{n+1}$ is the next position, $dt$ is the step size, $v(x)$ is the vector field at position $x$, and $O(dt^5)$ is the local truncation error [S2, unofficial].

For unsteady fields, the notation from [S3, unofficial] applies:
$$v_0 = \nu(x_i, t_{\text{func}})$$
$$v_1 = \nu(x_i + \tfrac{1}{2} \delta v_0, t_{\text{func}})$$
$$v_2 = \nu(x_i + \tfrac{1}{2} \delta v_1, t_{\text{func}})$$
$$v_3 = \nu(x_i + \delta v_2, t_{\text{func}})$$
$$x_{i+1} = x_i + \tfrac{1}{6} \delta (v_0 + 2v_1 + 2v_2 + v_3)$$
where $\delta$ is the step size and $t_{\text{func}}$ is the function time [S3, unofficial].

**Parameters:**
*   **Step Size ($dt$ or $\delta$):** In real-time applications, the step size is often dynamic, adjusted to maintain frame rate [S2, unofficial][S3, unofficial]. For steady fields, $t_{\text{func}}$ is ignored or set to a fixed value [S3, unofficial].

**Pitfalls and Edge Cases:**
*   **Vortex Behavior:** RK4 stays within vortices better than Euler, but may sometimes converge to the center of vortices instead of escaping them [S2, unofficial].
*   **Precision vs Performance:** The error is smaller by an order of magnitude compared to Euler, but the method is approximately 4x slower due to the four required samples [S2, unofficial].
*   **Unsteady Fields:** If the field is time-dependent, the step size $\delta$ and function time $t_{\text{func}}$ must be synchronized to maintain real-time behavior [S3, unofficial].
*   **Isotropic Regions:** When integrating tensorlines, integration stops immediately if the line enters an isotropic region, as directional information is lost [S1, book].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def rk4_integrate(pos, dt, vector_field_sampler):
    """
    Performs one step of Runge-Kutta 4 integration.
    
    Args:
        pos: Current position (float3).
        dt: Step size (float).
        vector_field_sampler: Function returning (velocity: float3, scalar: float) at a position.
    
    Returns:
        next_pos: New position (float3).
    """
    # Step 1: Sample k1 at current position
    k1 = dt * vector_field_sampler(pos)
    
    # Step 2: Sample k2 at pos + k1/2
    k2 = dt * vector_field_sampler(pos + k1 / 2.0)
    
    # Step 3: Sample k3 at pos + k2/2
    k3 = dt * vector_field_sampler(pos + k2 / 2.0)
    
    # Step 4: Sample k4 at pos + k3
    k4 = dt * vector_field_sampler(pos + k3)
    
    # Step 5: Weighted sum for displacement
    return pos + (k1 + 2.0 * k2 + 2.0 * k3 + k4) / 6.0
```

## Subtopics

### Euler Integration
A first-order integrator that uses only the current position and vector field value to compute the next step ($x_{n+1} = x_n + dt \cdot v(x_n)$) [S2, unofficial]. It has a local error of $O(dt^2)$ and is significantly less accurate than RK4, often escaping vortices even at small step sizes [S2, unofficial]. It is used when performance is critical and accuracy requirements are low [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per step, but requires 4 vector field samples (texture lookups) per step, making it approximately 4x slower than Euler integration [S2, unofficial].
*   **Memory Complexity:** Not given in the sources.
*   **Accuracy:** Local truncation error is $O(dt^5)$, significantly smaller than Euler's $O(dt^2)$ [S2, unofficial]. RK4 with a large step size (e.g., $dt=1$) can have less error than Euler with a smaller step size (e.g., $dt=0.5$) [S2, unofficial].
*   **Strengths:** Better preservation of flow features like vortices compared to lower-order methods [S2, unofficial].
*   **Limitations:** Higher computational cost; may converge to vortex centers in specific flow configurations [S2, unofficial]; requires careful step size selection for real-time performance [S3, unofficial].
*   **When to use:** When visual accuracy of streamlines or particle paths is more important than raw performance, or when large step sizes are needed to cover long distances without excessive error [S2, unofficial].
*   **When not to use:** When real-time performance is the primary constraint and the vector field is simple or low-resolution, or when adaptive step sizing is required (in which case RK2/3 might be preferred) [S2, unofficial][G].

## Background
In the context of volume rendering and vector field visualization, RK4 is widely used as a standard high-order integrator for streamlines and pathlines [S1, book]. It is implemented in real-time visualization systems using CUDA for GPU acceleration [S2, unofficial], and in academic research on real-time smoke volume rendering [S3, unofficial]. The method is preferred over Euler integration when the visual fidelity of the flow topology is critical [S2, unofficial].

## Sources
[S1] A Visual Approach to Analysis of Stress Tensor Fields, https://doi.org/10.4230/dfu.vol2.sciviz.2011.188, fetched 2026-10-26 (book)
[S2] Realtime visualization of 3D vector fields using CUDA | Marek's blog, https://marekfiser.com/projects/realtime-visualization-of-3D-vector-fields-using-CUDA/, fetched 2026-10-28 (unofficial)
[S3] Real-Time Smoke Volume Rendering, http://v22015081501727525.yourvserver.net/webpage/dirkfiles/betreutearbeiten/Sowinski_MA_2012.pdf, fetched 2026-10-31 (unofficial)
