# Euler Integration

Also called: Euler Integrator, Euler Method, First-order Euler Integrator, Forward Euler

## Overview
Euler Integration is a first-order numerical integration scheme used to solve initial value problems for ordinary differential equations (ODEs) by linearizing the Taylor series expansion [S1, unofficial]. In the context of volume rendering and vector field visualization, it serves as the simplest method to compute streamlines or particle trajectories by stepping along the vector field using the current position and field value [S2, unofficial]. It is distinguished from its sibling [Runge-Kutta 4 (RK4) Integration](3158-runge-kutta-4-rk4-integration.md) by its single-sample-per-step approach, which results in lower accuracy ($O(dt)$ global error) but significantly lower computational cost per step [S2, unofficial].

## How it works
**Inputs:**
*   $x_n$: Current position vector (typically 3D).
*   $dt$: Time step or step size scalar.
*   $v(x)$: Vector field function that returns the vector at position $x$.

**Outputs:**
*   $x_{n+1}$: Next position vector.

**Data Structures:**
*   Position: A 3D vector $(x, y, z)$.
*   Vector Field: A 3D texture or grid storing 3D vectors at voxel centers.

**Steps:**
1.  Sample the vector field at the current position $x_n$ to obtain the velocity vector $v_n = v(x_n)$ [S2, unofficial].
2.  Compute the displacement by scaling the velocity vector by the step size $dt$ [S1, unofficial].
3.  Add the displacement to the current position to determine the next position $x_{n+1}$ [S2, unofficial].

**Formula:**
The update rule is defined as:
$$ x_{n+1} = x_n + dt \cdot v(x_n) $$
where:
*   $x_n$ is the current position.
*   $x_{n+1}$ is the next position.
*   $dt$ is the time step (step size).
*   $v(x_n)$ is the vector field value at $x_n$ [S2, unofficial].

The local truncation error per step is $O(dt^2)$, and the global error over a fixed interval is $O(dt)$, making it a first-order method [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def euler_integrate(pos, dt, vector_field_sampler):
    """
    Performs one step of Euler integration.
    
    Args:
        pos: Current position (Vec3 or tuple of floats).
        dt: Step size (float).
        vector_field_sampler: Function that takes (x, y, z) and returns (vx, vy, vz).
        
    Returns:
        next_pos: The new position after the step.
    """
    # 1. Sample vector field at current position
    # Note: In practice, this may involve texture sampling with interpolation.
    vx, vy, vz = vector_field_sampler(pos.x, pos.y, pos.z)
    
    # 2. Compute displacement and add to current position
    next_x = pos.x + dt * vx
    next_y = pos.y + dt * vy
    next_z = pos.z + dt * vz
    
    return Vec3(next_x, next_y, next_z)
```

**Parameters:**
*   $dt$ (step size): Determines the precision and computational cost. Smaller $dt$ reduces error but increases computation time [S2, unofficial]. Typical values depend on the field's magnitude and required visual fidelity; however, even very low $dt$ values may not prevent qualitative errors like escaping vortices [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Vortex Escaping:** Euler integration tends to escape vortices even for very low time step values, whereas higher-order methods like RK4 may converge to the center of vortices [S2, unofficial].
*   **Step Size Sensitivity:** For inappropriately large $h$ (or $dt$), the error remains undesirably large. While higher-order schemes reduce approximation order, they are still susceptible to large errors if $h$ is too large [S1, unofficial].
*   **Precision:** The global error is proportional to the step size. If high accuracy is required, $dt$ must be very small, which can lead to performance bottlenecks compared to higher-order methods [S2, unofficial].

## Subtopics

### Bidirectional Diffusion Tensor Tracking
A variant used in diffusion tensor imaging (DTI) where the tracking process takes both forward and backward steps from a seed point [S3, unofficial]. By reversing the time step sign ($\Delta t \to -\Delta t$), the method tracks "backwards" through the flow [S3, unofficial]. This approach overcomes the arbitrary choice between eigenvectors and their negatives when decomposing the tensor field, which is particularly useful for identifying feeding vessels in arterio-venous malformations and aneurysms [S3, unofficial].

### Verlet Integration
A numerical integration method for particle dynamics that is time-reversible and unconditionally stable [S4, unofficial]. It updates particle positions based on current and previous positions without explicitly storing velocity [S4, unofficial]. The position is updated using the scheme: $new\_pos = pos + (pos - old\_pos) \cdot damping + acc \cdot dt^2$, followed by updating $old\_pos = pos$ and resetting acceleration [S4, unofficial]. A damping scalar (default 0.99) simulates energy loss [S4, unofficial]. Pinned particles do not update their position or apply forces [S4, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
*   **Time:** $O(1)$ per step, requiring only one vector field evaluation [S1, unofficial].
*   **Memory:** $O(1)$ auxiliary memory per particle/point, storing only the current position and potentially the previous position if needed for specific variants [S4, unofficial].

**Measured Performance:**
*   Euler integration is significantly faster per step than [Runge-Kutta 4 (RK4) Integration](3158-runge-kutta-4-rk4-integration.md) because it samples the vector field only once per step [S2, unofficial].
*   However, to achieve comparable accuracy to RK4, the step size $dt$ must be drastically reduced, which increases the total number of steps and may negate the per-step speed advantage [S2, unofficial].

**Strengths and Limitations:**
*   **Strengths:** Simplicity of implementation (literally two lines of code in CUDA) [S2, unofficial]; low computational overhead per step; suitable for real-time applications where approximate trajectories are acceptable [S2, unofficial].
*   **Limitations:** Low accuracy ($O(dt)$ global error); prone to qualitative errors such as escaping vortices even with small step sizes [S2, unofficial]; requires very small step sizes for high precision, impacting performance [S2, unofficial].

**When to Use:**
*   Use when computational resources are limited and approximate streamline visualization is sufficient.
*   Use as a baseline for comparison with higher-order methods.

**When Not to Use:**
*   Use when high accuracy is required, especially in regions with complex flow features like vortices.
*   Use when the vector field varies rapidly and small step sizes would make Euler integration too slow compared to higher-order methods.

## Background
The Euler Method is a first-order numerical integration scheme for univariate functions $x: \mathbb{R} \to \mathbb{R}^n$, representing a linearization of the Taylor series expansion [S1, unofficial]. It is one of the oldest and simplest methods for solving initial value problems (IVPs) [S1, unofficial]. In the context of vector field visualization, it was implemented early in projects due to its simplicity and the speed of GPUs, which allowed for small time steps to mitigate error [S2, unofficial]. However, it was later found that even with small time steps, Euler integration suffered from qualitative inaccuracies compared to higher-order methods like Runge-Kutta 4 [S2, unofficial].

## Sources
[S1] Multi-Field Visualization, https://kluedo.ub.rptu.de/files/2291/_diss.pdf, fetched 2026-10-27 (unofficial)
[S2] Realtime visualization of 3D vector fields using CUDA | Marek's blog, https://marekfiser.com/projects/realtime-visualization-of-3D-vector-fields-using-CUDA/, fetched 2026-10-28 (unofficial)
[S3] Angiotool : a tool for interactive visualization of MRI vector and tensor fields, https://doi.org/10.82444/warw.33642268, fetched 2026-10-26 (unofficial)
[S4] arguskern/lumina-graphics-lab, https://github.com/arguskern/lumina-graphics-lab, fetched 2026-10-26 (unofficial)
