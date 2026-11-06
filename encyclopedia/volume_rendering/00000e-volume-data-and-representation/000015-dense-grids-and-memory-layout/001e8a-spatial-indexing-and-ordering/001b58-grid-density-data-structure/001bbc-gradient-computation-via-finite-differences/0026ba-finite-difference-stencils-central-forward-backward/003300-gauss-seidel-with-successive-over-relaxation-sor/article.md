# Gauss-Seidel with Successive Over-Relaxation (SOR)

Also called: Gauss-Seidel SOR, Gauss-Seidel with Successive Over-Relaxation (SOR) for 3D Heat Conduction, Iterative Laplace Solver

## Overview
Gauss-Seidel with Successive Over-Relaxation (SOR) is an iterative numerical method used to solve the steady-state heat equation (Laplace equation $\nabla^2 T = 0$) on a 3D uniform Cartesian grid [S1, unofficial]. It updates grid points using the most recently computed neighbor values (Gauss-Seidel) and accelerates convergence by applying a relaxation factor $\omega > 1$ (Successive Over-Relaxation) [S1, unofficial]. Unlike standard Jacobi iteration which uses values from the previous iteration only, SOR utilizes updated values immediately, reducing the number of iterations required to reach a solution [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** A 3D scalar field (temperature) $T$ of shape $(n_x, n_y, n_z)$, grid dimensions $n_x, n_y, n_z$, physical domain length $L$, boundary temperatures $T_{hot}$ and $T_{cold}$, SOR relaxation parameter $\omega$, convergence tolerance $\epsilon$, and maximum iteration count $N_{max}$.
- **Output:** The converged temperature field $T$ and the number of iterations performed.

**Data Structures:**
- **Temperature Field:** A 3D array of shape $(n_x, n_y, n_z)$ with floating-point dtype. Indices $(i, j, k)$ correspond to grid points in the x, y, and z directions respectively [S1, unofficial].

**Algorithm Steps:**
1. **Initialization:** Initialize the temperature field with a linear profile between the hot and cold boundaries to reduce iteration count. For $k \in [0, n_z-1]$:
   $$ T[i, j, k] = T_{hot} + (T_{cold} - T_{hot}) \frac{k}{n_z - 1} $$
   [S1, unofficial]

2. **Boundary Conditions:** Apply Dirichlet and Neumann boundary conditions:
   - **Dirichlet (Fixed Temperature):** Set the bottom face ($k=0$) to $T_{hot}$ and the top face ($k=n_z-1$) to $T_{cold}$ [S1, unofficial].
   - **Neumann (Adiabatic/Zero Gradient):** On side walls ($x=0, x=n_x-1, y=0, y=n_y-1$), copy values from adjacent interior cells to enforce $\frac{\partial T}{\partial n} = 0$ [S1, unofficial].

3. **Iteration Loop:** Repeat for $iter = 1$ to $N_{max}$:
   a. Initialize `max_residual` to 0.0.
   b. Loop over all interior points $i \in [1, n_x-2]$, $j \in [1, n_y-2]$, $k \in [1, n_z-2]$:
      i. Calculate the sum of the six immediate neighbors:
         $$ S = T[i+1, j, k] + T[i-1, j, k] + T[i, j+1, k] + T[i, j-1, k] + T[i, j, k+1] + T[i, j, k-1] $$
         [S1, unofficial]
      ii. Calculate the tentative new value based on the Laplace equation average:
          $$ T_{new} = \frac{S}{6.0} $$
          [S1, unofficial]
      iii. Calculate the residual for this point:
           $$ r = |T_{new} - T[i, j, k]| $$
           Update `max_residual = max(max_residual, r)` [S1, unofficial].
      iv. Apply the SOR update formula:
          $$ T[i, j, k] = (1 - \omega) T[i, j, k] + \omega T_{new} $$
          [S1, unofficial]
   c. Re-apply boundary conditions to ensure Dirichlet values remain fixed and Neumann conditions are maintained [S1, unofficial].
   d. Check convergence: If `max_residual` < $\epsilon$, terminate and return the current iteration count [S1, unofficial].

4. **Termination:** If $N_{max}$ is reached without convergence, return the final field and iteration count [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def solve_gauss_seidel_sor(nx, ny, nz, t_hot, t_cold, omega, tolerance, max_iterations):
    # Initialize temperature field with linear interpolation
    temperature = [[[[0.0 for _ in range(nz)] for _ in range(ny)] for _ in range(nx)] for _ in range(1)]
    for k in range(nz):
        z_ratio = k / (nz - 1)
        for i in range(nx):
            for j in range(ny):
                temperature[0][i][j][k] = t_hot + (t_cold - t_hot) * z_ratio

    def apply_boundary_conditions(temp):
        # Dirichlet: Bottom (k=0) and Top (k=nz-1)
        for i in range(nx):
            for j in range(ny):
                temp[0][i][j][0] = t_hot
                temp[0][i][j][nz-1] = t_cold
        # Neumann: Side walls (copy from interior)
        for j in range(ny):
            for k in range(nz):
                temp[0][0][j][k] = temp[0][1][j][k]       # x=0
                temp[0][nx-1][j][k] = temp[0][nx-2][j][k] # x=L
        for i in range(nx):
            for k in range(nz):
                temp[0][i][0][k] = temp[0][i][1][k]       # y=0
                temp[0][i][ny-1][k] = temp[0][i][ny-2][k] # y=L

    apply_boundary_conditions(temperature)

    for iteration in range(1, max_iterations + 1):
        max_residual = 0.0
        for i in range(1, nx - 1):
            for j in range(1, ny - 1):
                for k in range(1, nz - 1):
                    # Sum of six neighbors
                    neighbor_sum = (
                        temperature[0][i+1][j][k] +
                        temperature[0][i-1][j][k] +
                        temperature[0][i][j+1][k] +
                        temperature[0][i][j-1][k] +
                        temperature[0][i][j][k+1] +
                        temperature[0][i][j][k-1]
                    )
                    # New value from Laplace equation
                    t_new = neighbor_sum / 6.0
                    # Compute residual
                    residual = abs(t_new - temperature[0][i][j][k])
                    if residual > max_residual:
                        max_residual = residual
                    # SOR update
                    temperature[0][i][j][k] = (1 - omega) * temperature[0][i][j][k] + omega * t_new
        # Re-apply boundary conditions
        apply_boundary_conditions(temperature)
        # Check convergence
        if max_residual < tolerance:
            return iteration
    return max_iterations
```

**Parameters:**
- $\omega$ (omega): SOR relaxation parameter. Typical value is 1.5. The range for over-relaxation is $1 < \omega < 2$. Optimal values are typically between 1.5 and 1.9 [S1, unofficial].
- $\epsilon$ (tolerance): Convergence criterion based on the maximum equation residual. Typical value is $10^{-5}$ [S1, unofficial].
- $N_{max}$ (max_iterations): Maximum number of iterations allowed. Typical value is 5000 [S1, unofficial].

**Pitfalls:**
- Boundary conditions must be reapplied after every iteration to maintain fixed Dirichlet values and zero-gradient Neumann conditions [S1, unofficial].
- Using $\omega \ge 2$ can cause divergence [S1, unofficial].
- The method assumes a uniform grid where $dx = dy = dz = h$ [S1, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
- **Time Complexity:** $O(N)$ per iteration, where $N = n_x \cdot n_y \cdot n_z$ is the total number of grid points. The total time depends on the number of iterations required for convergence, which is reduced by SOR compared to Jacobi or standard Gauss-Seidel [S1, unofficial].
- **Memory Complexity:** $O(N)$ to store the 3D temperature field [S1, unofficial].

**Performance:**
- SOR converges faster than standard Gauss-Seidel due to the over-relaxation factor [S1, unofficial].
- Performance scales with grid resolution; larger grids require more iterations [S1, unofficial].
- Strengths: Simple implementation, low memory overhead, effective for elliptic PDEs like the Laplace equation [S1, unofficial].
- Limitations: Convergence speed depends heavily on the choice of $\omega$; not parallel-friendly due to sequential dependency of updates (Gauss-Seidel nature) [S1, unofficial].

## Background
The method originates from solving the steady-state heat equation in a 3D cubic domain [S1, unofficial]. It combines the Gauss-Seidel iterative solver, which uses the latest available neighbor values, with Successive Over-Relaxation (SOR) to accelerate convergence [S1, unofficial]. It is used in volume rendering contexts to generate smooth scalar fields (e.g., temperature distributions) for visualization, where the scalar field is mapped to color and opacity via transfer functions [S1, unofficial].

## Sources
[S1] Vtk-Examples/src/02_advanced_shapes/volume_rendering.py at main ..., https://github.com/djeada/VTK-Examples/blob/main/src/02_advanced_shapes/volume_rendering.py, fetched 2026-10-29 (unofficial)
