# Runge-Kutta 4 (RK4) Integration

Also called: Fourth-order Runge-Kutta, RK4 Integrator, Runge-Kutta 4 Integrator, Runge-Kutta 4th order

## Overview
Runge-Kutta 4 (RK4) is a higher-order numerical integration scheme for volume rendering that queries the vector field at four strategically chosen positions to compute a single integration step, significantly reducing trajectory error compared to first-order methods like [Euler Integration](3836-euler-integration.md) [S1, unofficial]. It solves the initial value problem for a point in a vector field by calculating the next position using a weighted sum of these samples, offering superior accuracy for a modest increase in computational cost [S1, unofficial][S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Current position $x_n$ (or $x_i$), step size $\delta$ (or $dt$), and optionally a time parameter $t_{\text{func}}$ for unsteady fields.
*   **Output:** Next position $x_{n+1}$ (or $x_{i+1}$).

**Data Structures:**
*   Position: 3D vector (e.g., `double3` or `float3`).
*   Vector Field Sample: 3D vector (velocity/direction) at a specific position. In unsteady fields, the field function also accepts time.

**Numbered Steps:**
1.  Calculate half-step size: $dt_{half} = dt \cdot 0.5$.
2.  Sample the vector field at the current position to get $k_1$ (or $v_0$):
    $$k_1 = dt \cdot v(x_n)$$
3.  Sample the vector field at the midpoint using $k_1$ to get $k_2$ (or $v_1$):
    $$k_2 = dt \cdot v(x_n + k_1 / 2)$$
4.  Sample the vector field at the midpoint using $k_2$ to get $k_3$ (or $v_2$):
    $$k_3 = dt \cdot v(x_n + k_2 / 2)$$
5.  Sample the vector field at the full step using $k_3$ to get $k_4$ (or $v_3$):
    $$k_4 = dt \cdot v(x_n + k_3)$$
6.  Compute the weighted sum of the samples to determine the displacement:
    $$\Delta x = \frac{dt}{6} (k_1 + 2k_2 + 2k_3 + k_4)$$
    *Note: Sources [S1, unofficial] and [S2, unofficial] use equivalent formulations. [S1, unofficial] defines $k_i$ as the full step contribution ($dt \cdot v$), while [S2, unofficial] defines $v_i$ as the velocity ($v$) and applies the weighting and $dt$ in the final sum. The result is identical.*
7.  Update the position:
    $$x_{n+1} = x_n + \Delta x$$

**Formulas:**
Using the notation from [S1, unofficial]:
$$k_1 = dt \cdot v(x_n)$$
$$k_2 = dt \cdot v(x_n + k_1/2)$$
$$k_3 = dt \cdot v(x_n + k_2/2)$$
$$k_4 = dt \cdot v(x_n + k_3)$$
$$x_{n+1} = x_n + \frac{k_1}{6} + \frac{k_2}{3} + \frac{k_3}{3} + \frac{k_4}{6} + O(dt^5)$$

Using the notation from [S2, unofficial] for unsteady fields $\nu(x, t)$:
$$v_0 = \nu(x_i, t_{\text{func}})$$
$$v_1 = \nu(x_i + \frac{1}{2} \delta v_0, t_{\text{func}})$$
$$v_2 = \nu(x_i + \frac{1}{2} \delta v_1, t_{\text{func}})$$
$$v_3 = \nu(x_i + \delta v_2, t_{\text{func}})$$
$$x_{i+1} = x_i + \frac{1}{6} \delta (v_0 + 2v_1 + 2v_2 + v_3)$$

**Parameters:**
*   **Step size ($dt$ or $\delta$):** In real-time applications, this is often dynamic, adjusted to maintain frame rate [S2, unofficial]. For steady fields, it is fixed.
*   **Time ($t_{\text{func}}$):** Required for unsteady vector fields. For steady fields, this argument is ignored or set to a fixed value [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Steady vs. Unsteady:** For steady fields, the time argument must be handled correctly (ignored or constant) to avoid unnecessary computation or errors [S2, unofficial].
*   **Vortex Behavior:** RK4 tends to stay within vortices better than Euler integration, which may escape them. However, RK4 can sometimes converge to the center of vortices instead of following the flow correctly [S1, unofficial].
*   **Precision:** The method assumes $dt \ll 1$ for the error term $O(dt^5)$ to hold [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def rk4_integrate(pos, dt, vector_field_func, t_func=None):
    """
    Performs one step of Runge-Kutta 4 integration.
    
    Args:
        pos: Current position (x, y, z).
        dt: Step size.
        vector_field_func: Function that returns vector (vx, vy, vz) at a given position.
                           For unsteady fields, it may also accept time.
        t_func: Current time for unsteady fields. If None, treated as steady.
    
    Returns:
        New position (x, y, z).
    """
    dt_half = dt * 0.5
    
    # Sample 1: At current position
    # If unsteady, pass t_func; otherwise ignore or pass constant
    if t_func is not None:
        k1 = dt * vector_field_func(pos.x, pos.y, pos.z, t_func)
    else:
        k1 = dt * vector_field_func(pos.x, pos.y, pos.z)
        
    # Sample 2: At midpoint using k1
    mid_pos_2 = (
        pos.x + dt_half * k1.x,
        pos.y + dt_half * k1.y,
        pos.z + dt_half * k1.z
    )
    if t_func is not None:
        k2 = dt * vector_field_func(mid_pos_2[0], mid_pos_2[1], mid_pos_2[2], t_func)
    else:
        k2 = dt * vector_field_func(mid_pos_2[0], mid_pos_2[1], mid_pos_2[2])
        
    # Sample 3: At midpoint using k2
    mid_pos_3 = (
        pos.x + dt_half * k2.x,
        pos.y + dt_half * k2.y,
        pos.z + dt_half * k2.z
    )
    if t_func is not None:
        k3 = dt * vector_field_func(mid_pos_3[0], mid_pos_3[1], mid_pos_3[2], t_func)
    else:
        k3 = dt * vector_field_func(mid_pos_3[0], mid_pos_3[1], mid_pos_3[2])
        
    # Sample 4: At full step using k3
    full_pos_4 = (
        pos.x + dt * k3.x,
        pos.y + dt * k3.y,
        pos.z + dt * k3.z
    )
    if t_func is not None:
        k4 = dt * vector_field_func(full_pos_4[0], full_pos_4[1], full_pos_4[2], t_func)
    else:
        k4 = dt * vector_field_func(full_pos_4[0], full_pos_4[1], full_pos_4[2])
        
    # Weighted sum
    dt_sixth = dt / 6.0
    
    new_x = pos.x + dt_sixth * (k1.x + 2.0 * k2.x + 2.0 * k3.x + k4.x)
    new_y = pos.y + dt_sixth * (k1.y + 2.0 * k2.y + 2.0 * k3.y + k4.y)
    new_z = pos.z + dt_sixth * (k1.z + 2.0 * k2.z + 2.0 * k3.z + k4.z)
    
    return (new_x, new_y, new_z)
```

## Subtopics

### [Euler Integration](3836-euler-integration.md)
Euler Integration is a first-order numerical integration scheme for univariate functions $x: \mathbb{R} \to \mathbb{R}^n$, used to approximate the solution of initial value problems (IVPs) in vector fields. It linearizes the Taylor series expansion to compute the next position from the current position and the vector field derivative. It is distinguished from its parent [Runge-Kutta 4 (RK4) Integration](3158-runge-kutta-4-rk4-integration.md) by its simplicity and lower accuracy; while RK4 uses four samples to achieve fourth-order accuracy, Euler uses only one sample, resulting in first-order global error.

### [Runge-Kutta 2/3 ODE Solver](3839-runge-kutta-2-3-ode-solver.md)
A 2nd/3rd order Runge-Kutta method for solving ODEs with adaptive step size control. It sets itself apart from its parent [Runge-Kutta 4 (RK4) Integration](3158-runge-kutta-4-rk4-integration.md) by using a lower-order embedded pair to estimate local truncation error, enabling dynamic step size adjustment rather than fixed steps [G]. It is distinct from the [Hybrid ODE Solver for Ray Trajectory](#hybrid-ode-solver-for-ray-trajectory) variant, which selectively falls back to Euler integration in specific geometric regimes.

### Tensorline Integration
Tensorline Integration uses RK4 to integrate lines from seed points placed randomly inside a masked volume (focus region) [S3, book]. The integration starts at these seed points and stops when the line enters an isotropic region [S3, book]. A key pitfall is that if the tensor is isotropic, the integration terminates immediately or provides no directional information, which the method explicitly handles by stopping [S3, book].

## Operating characteristics
*   **Time Complexity:** RK4 requires four vector field samples per step, making it approximately 4 times slower than Euler integration [S1, unofficial].
*   **Accuracy:** RK4 has an error of $O(dt^5)$ per step (global error $O(dt^4)$), which is an order of magnitude smaller than Euler's $O(dt^2)$ per step [S1, unofficial]. It maintains accuracy even with larger time steps; for example, RK4 with $dt=1$ has less error than Euler with $dt=0.5$ [S1, unofficial].
*   **Memory:** Requires storage for the current position and temporary variables for the four samples. No additional large data structures are needed beyond the vector field texture/array.
*   **Strengths:** Significantly better trajectory accuracy than Euler, especially in complex flow fields like vortices. It stays within vortices where Euler tends to escape [S1, unofficial].
*   **Limitations:** Higher computational cost. May converge to vortex centers instead of following the flow in some specific cases [S1, unofficial]. Not suitable for real-time rendering if step sizes cannot be reduced sufficiently without performance loss, though it is often used in real-time contexts by adjusting step size dynamically [S2, unofficial].

## Background
RK4 was implemented in the context of real-time visualization of 3D vector fields using CUDA to overcome the limitations of Euler integration, which suffered from high error even with small step sizes [S1, unofficial]. It is a standard method for solving initial value problems in unsteady vector fields, where the step size and function time must be synchronized for real-time behavior [S2, unofficial]. It is also used in tensor line visualization to integrate paths through focus regions in stress tensor fields [S3, book].

## Sources
[S1] Realtime visualization of 3D vector fields using CUDA | Marek's blog, https://marekfiser.com/projects/realtime-visualization-of-3D-vector-fields-using-CUDA/, fetched 2026-10-28 (unofficial)
[S2] Real-Time Smoke Volume Rendering, http://v22015081501727525.yourvserver.net/webpage/dirkfiles/betreutearbeiten/Sowinski_MA_2012.pdf, fetched 2026-10-31 (unofficial)
[S3] A Visual Approach to Analysis of Stress Tensor Fields, https://doi.org/10.4230/dfu.vol2.sciviz.2011.188, fetched 2026-10-26 (book)
