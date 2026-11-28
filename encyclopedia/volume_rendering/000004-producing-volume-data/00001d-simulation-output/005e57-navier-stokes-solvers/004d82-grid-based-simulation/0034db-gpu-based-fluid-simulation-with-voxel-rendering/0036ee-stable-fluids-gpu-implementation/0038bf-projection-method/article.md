# Projection Method

Also called: Hodge Decomposition, Pressure Projection

## Overview
The Projection Method enforces incompressibility on a fluid velocity field by subtracting the gradient of a pressure field, ensuring the resulting velocity is divergence-free [S1, unofficial]. It serves as the incompressibility enforcement step within the Stable Fluids framework [G].

## How it works
**Inputs and Outputs:**
*   Input: An intermediate velocity field $\mathbf{u}^*$ (potentially divergent) and a grid of size $N \times N \times N$ [G].
*   Output: A divergence-free velocity field $\mathbf{u}$ and a pressure field $p$ [S1, unofficial].

**Data Structures:**
*   Velocity field: 3D array of floats, dimensions $N \times N \times N$, indexed by $(i, j, k)$ [G].
*   Pressure field: 3D array of floats, dimensions $N \times N \times N$ [G].
*   Divergence field: 3D array of floats, dimensions $N \times N \times N$ [G].

**Steps:**
1.  **Compute Divergence:** Calculate the divergence of the intermediate velocity field $\mathbf{u}^*$ [S1, unofficial]. For a cell $(i, j, k)$, the divergence $d_{i,j,k}$ is approximated using central differences with uniform grid spacing $\Delta x = \Delta y = \Delta z = 1$:
    $$ d_{i,j,k} = \frac{u^*_{i+1,j,k} - u^*_{i-1,j,k}}{2} + \frac{v^*_{i,j+1,k} - v^*_{i,j-1,k}}{2} + \frac{w^*_{i,j,k+1} - w^*_{i,j,k-1}}{2} $$
    [S1, unofficial]
2.  **Solve Poisson Equation:** Solve for the pressure field $p$ such that $\nabla^2 p = d$ [S1, unofficial]. This is typically done using Jacobi iteration:
    $$ p^{(k+1)}_{i,j,k} = \frac{1}{6} \left( p^{(k)}_{i+1,j,k} + p^{(k)}_{i-1,j,k} + p^{(k)}_{i,j+1,k} + p^{(k)}_{i,j-1,k} + p^{(k)}_{i,j,k+1} + p^{(k)}_{i,j,k-1} - d_{i,j,k} \right) $$
    The iteration runs for a fixed number of steps, typically 20-40 iterations [S1, unofficial].
3.  **Subtract Gradient:** Update the velocity field by subtracting the gradient of the pressure field to make it divergence-free [S1, unofficial]:
    $$ \mathbf{u} = \mathbf{u}^* - \nabla p $$
    Component-wise:
    $$ u_{i,j,k} = u^*_{i,j,k} - \frac{p_{i+1,j,k} - p_{i-1,j,k}}{2} $$
    $$ v_{i,j,k} = v^*_{i,j,k} - \frac{p_{i,j+1,k} - p_{i,j-1,k}}{2} $$
    $$ w_{i,j,k} = w^*_{i,j,k} - \frac{p_{i,j,k+1} - p_{i,j,k-1}}{2} $$
    [S1, unofficial]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def projection_method(u_star, N, iterations=30):
    """
    Enforces incompressibility on velocity field u_star.
    u_star: list of 3 arrays (u, v, w) of size N*N*N.
    Returns: divergence-free velocity field (u, v, w).
    """
    # Initialize pressure field
    p = [[[0.0 for _ in range(N)] for _ in range(N)] for _ in range(N)]
    
    # Step 1: Compute divergence of u_star
    div = [[[0.0 for _ in range(N)] for _ in range(N)] for _ in range(N)]
    for i in range(1, N-1):
        for j in range(1, N-1):
            for k in range(1, N-1):
                # Central differences for divergence (Δx=1)
                du = (u_star[0][i+1][j][k] - u_star[0][i-1][j][k]) / 2.0
                dv = (u_star[1][i][j+1][k] - u_star[1][i][j-1][k]) / 2.0
                dw = (u_star[2][i][j][k+1] - u_star[2][i][j][k-1]) / 2.0
                div[i][j][k] = du + dv + dw

    # Step 2: Solve Poisson equation for pressure using Jacobi iteration
    for _ in range(iterations):
        new_p = [[[0.0 for _ in range(N)] for _ in range(N)] for _ in range(N)]
        for i in range(1, N-1):
            for j in range(1, N-1):
                for k in range(1, N-1):
                    # Jacobi update: average of neighbors minus divergence
                    new_p[i][j][k] = (p[i+1][j][k] + p[i-1][j][k] +
                                      p[i][j+1][k] + p[i][j-1][k] +
                                      p[i][j][k+1] + p[i][j][k-1] -
                                      div[i][j][k]) / 6.0
        p = new_p

    # Step 3: Subtract gradient of pressure from velocity
    u = [[[0.0 for _ in range(N)] for _ in range(N)] for _ in range(N)]
    v = [[[0.0 for _ in range(N)] for _ in range(N)] for _ in range(N)]
    w = [[[0.0 for _ in range(N)] for _ in range(N)] for _ in range(N)]
    
    for i in range(1, N-1):
        for j in range(1, N-1):
            for k in range(1, N-1):
                # Central differences for gradient (Δx=1)
                u[i][j][k] = u_star[0][i][j][k] - (p[i+1][j][k] - p[i-1][j][k]) / 2.0
                v[i][j][k] = u_star[1][i][j][k] - (p[i][j+1][k] - p[i][j-1][k]) / 2.0
                w[i][j][k] = u_star[2][i][j][k] - (p[i][j][k+1] - p[i][j][k-1]) / 2.0
                    
    return (u, v, w)
```
[G]

**Parameters:**
*   `iterations`: Number of Jacobi iterations. Typical value: 20-40 [S1, unofficial].

**Edge Cases and Pitfalls:**
*   Boundary conditions: The source code snippet shows specific handling for boundaries, including reflection for free-slip and averaging for corners [S1, unofficial]. Failure to handle boundaries correctly leads to inaccurate pressure gradients at the domain edges [G].
*   Precision: Floating-point errors can accumulate during iterative solving [G]. Ensure convergence criteria are met or use a fixed sufficient number of iterations [G].

## Subtopics

### [Neural Pressure Projection Solver](12463-neural-pressure-projection-solver.md)
A neural network approach that predicts velocity updates to enforce incompressibility, replacing iterative solvers. It enables differentiable simulation but requires training data.

## Operating characteristics
*   **Time Complexity:** $O(N^3 \cdot I)$, where $N$ is the grid dimension and $I$ is the number of Jacobi iterations [G].
*   **Memory Complexity:** $O(N^3)$ for storing velocity, pressure, and divergence fields [G].
*   **Performance:** The method is computationally intensive due to the iterative solver [G]. Performance scales with grid resolution and iteration count [G]. GPU acceleration is commonly used for the parallelizable Jacobi iteration and gradient computation [G].
*   **Strengths:** Simple to implement, robust for incompressible flow enforcement [G].
*   **Limitations:** Iterative nature can be slow for high-precision requirements [G]. Convergence depends on the grid structure and boundary conditions [G].

## Background
The Projection Method is a fundamental technique in computational fluid dynamics [G]. It is a core component of the Stable Fluids method, which discretizes the Navier-Stokes equations on a fixed grid [G]. The method relies on the Helmholtz-Hodge decomposition theorem, which states that any vector field can be decomposed into a divergence-free component and a curl-free component (the gradient of a scalar potential, i.e., pressure) [G].

## Sources
[S1] tzhazuma/Fluid_simulation, https://github.com/tzhazuma/Fluid_simulation, fetched 2026-10-26 (unofficial)
