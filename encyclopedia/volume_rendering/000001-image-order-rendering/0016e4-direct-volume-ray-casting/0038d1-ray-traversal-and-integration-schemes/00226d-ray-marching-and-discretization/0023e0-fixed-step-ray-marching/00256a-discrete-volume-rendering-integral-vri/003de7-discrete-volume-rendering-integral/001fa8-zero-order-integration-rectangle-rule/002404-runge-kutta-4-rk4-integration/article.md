# Runge-Kutta 4 (RK4) Integration

Also called: Fourth-order Runge-Kutta, RK4 Integrator, Runge-Kutta 4 Integrator, Runge-Kutta 4th order

## Overview
Runge-Kutta 4 (RK4) is a fourth-order numerical integration method used to solve initial value problems for ordinary differential equations (ODEs) in volume rendering, specifically for computing streamlines and particle trajectories through vector fields [S1, unofficial][S2, unofficial]. It improves upon [Zero Order Integration (Rectangle Rule)](2082-zero-order-integration-rectangle-rule.md) by querying the vector field at four distinct positions within a step to compute a weighted average slope, significantly reducing integration error compared to first-order methods like [Euler Integration](3836-euler-integration.md) [S1, unofficial]. While it requires more computations per step, it allows for larger step sizes while maintaining accuracy, making it suitable for visualizing complex flow structures such as vortices [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: Current position $x_n$ (or $x_i$), step size $\delta$ (or $dt$), and optionally a time function $t_{\text{func}}$ for unsteady fields.
- Output: Next position $x_{n+1}$ (or $x_{i+1}$).

**Data Structures:**
- Positions and vectors are typically represented as 3D or 4D floating-point vectors (e.g., `double3`, `float4`).
- The vector field $v(x)$ is accessed via trilinear interpolation from a 3D texture or grid [S1, unofficial].

**Steps:**
1. Calculate half-step size: $dt_{\text{half}} = dt \cdot 0.5$ [S1, unofficial].
2. Sample the vector field at the current position to get $k_1$ (or $v_0$):
   $$k_1 = dt \cdot v(x_n)$$
3. Sample the vector field at the midpoint using $k_1$ to get $k_2$ (or $v_1$):
   $$k_2 = dt \cdot v(x_n + k_1/2)$$
4. Sample the vector field at the midpoint using $k_2$ to get $k_3$ (or $v_2$):
   $$k_3 = dt \cdot v(x_n + k_2/2)$$
5. Sample the vector field at the end of the step using $k_3$ to get $k_4$ (or $v_3$):
   $$k_4 = dt \cdot v(x_n + k_3)$$
6. Compute the weighted sum of the slopes to determine the displacement:
   $$x_{n+1} = x_n + \frac{1}{6}k_1 + \frac{1}{3}k_2 + \frac{1}{3}k_3 + \frac{1}{6}k_4$$
   Or equivalently:
   $$x_{n+1} = x_n + \frac{dt}{6}(k_1 + 2k_2 + 2k_3 + k_4)$$
   [S1, unofficial][S2, unofficial]

**Parameters:**
- Step size $dt$ (or $\delta$): In real-time applications, this is often dynamic, adjusted to maintain frame rate [S2, unofficial]. For steady fields, $t_{\text{func}}$ is ignored or fixed [S2, unofficial].

**Edge Cases and Pitfalls:**
- **Vortex Convergence:** RK4 may converge to the center of vortices rather than escaping them, unlike Euler which tends to escape [S1, unofficial].
- **Precision:** Using single precision for intermediate calculations may introduce errors; double precision is preferred for high accuracy [S1, unofficial].
- **Unsteady Fields:** For time-dependent fields, the time argument $t_{\text{func}}$ must be synchronized with the integration step size to maintain real-time behavior [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def rk4_integrate(pos, dt, vector_field_func):
    """
    Perform one step of Runge-Kutta 4 integration.
    
    Args:
        pos: Current position (x, y, z).
        dt: Step size.
        vector_field_func: Function that returns vector (vx, vy, vz) at a given position.
        
    Returns:
        Next position (x, y, z).
    """
    dt_half = dt * 0.5
    
    # Sample k1 at current position
    k1 = vector_field_func(pos)
    
    # Sample k2 at pos + dt_half * k1
    pos_k2 = (pos.x + dt_half * k1.x, pos.y + dt_half * k1.y, pos.z + dt_half * k1.z)
    k2 = vector_field_func(pos_k2)
    
    # Sample k3 at pos + dt_half * k2
    pos_k3 = (pos.x + dt_half * k2.x, pos.y + dt_half * k2.y, pos.z + dt_half * k2.z)
    k3 = vector_field_func(pos_k3)
    
    # Sample k4 at pos + dt * k3
    pos_k4 = (pos.x + dt * k3.x, pos.y + dt * k3.y, pos.z + dt * k3.z)
    k4 = vector_field_func(pos_k4)
    
    # Weighted sum
    dt_sixth = dt / 6.0
    new_x = pos.x + dt_sixth * (k1.x + 2.0 * k2.x + 2.0 * k3.x + k4.x)
    new_y = pos.y + dt_sixth * (k1.y + 2.0 * k2.y + 2.0 * k3.y + k4.y)
    new_z = pos.z + dt_sixth * (k1.z + 2.0 * k2.z + 2.0 * k3.z + k4.z)
    
    return (new_x, new_y, new_z)
```

## Subtopics

### [Euler Integration](3836-euler-integration.md)
First-order method, lower accuracy ($O(dt)$ global error), lower cost.

### [Runge-Kutta 2/3 ODE Solver](3839-runge-kutta-2-3-ode-solver.md)
Lower-order embedded pair for adaptive step size.

### [Tensorline Integration](9613-tensorline-integration.md)
Targets tensor fields, uses RK4 for trajectory accuracy.

### [Dormand-Prince 5(4) Step](13933-dormand-prince-5-4-step.md)
Fifth-order method with adaptive step control via error estimation.

## Operating characteristics
- **Time Complexity:** RK4 requires four vector field samples per step, making it approximately 4 times slower than Euler integration [S1, unofficial].
- **Memory Complexity:** Minimal additional memory beyond storing the current position and intermediate vectors.
- **Accuracy:** Global error is $O(dt^5)$ for the local truncation error, resulting in significantly smaller errors compared to Euler ($O(dt^2)$) [S1, unofficial]. For example, RK4 with $dt=1$ can have less error than Euler with $dt=0.5$ [S1, unofficial].
- **Strengths:** High accuracy, ability to use larger step sizes, better handling of complex flow structures like vortices [S1, unofficial].
- **Limitations:** Higher computational cost per step, potential for converging to vortex centers instead of escaping them [S1, unofficial].
- **When to Use:** When high accuracy is required and computational resources allow for more samples per step, especially in complex vector fields [S1, unofficial].
- **When Not to Use:** When real-time performance is critical and lower accuracy is acceptable, or when adaptive step size control is needed (use Dormand-Prince instead) [S1, unofficial][S2, unofficial].

## Background
RK4 was developed as part of the Runge-Kutta family of methods for solving ODEs [S1, unofficial]. It has been widely adopted in volume rendering and vector field visualization for its balance of accuracy and performance [S1, unofficial][S2, unofficial]. It is commonly used in real-time visualization tools and research projects involving 3D vector fields [S1, unofficial][S2, unofficial].

## Sources
[S1] Realtime visualization of 3D vector fields using CUDA, https://marekfiser.com/projects/realtime-visualization-of-3D-vector-fields-using-CUDA/, fetched 2026-10-28 (unofficial)
[S2] Real-Time Smoke Volume Rendering, http://v22015081501727525.yourvserver.net/webpage/dirkfiles/betreutearbeiten/Sowinski_MA_2012.pdf, fetched 2026-10-31 (unofficial)
