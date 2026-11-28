# Type-1 Direct Connection Algorithm

Also called: Type-1 connection, boundary-constrained direct connection

## Overview
The Type-1 Direct Connection Algorithm computes a refractive light path between a vertex $\mathbf{y}$ located on a medium boundary (defined by an implicit surface $g(\cdot)=0$) and a vertex $\mathbf{x}_0$ located inside the medium [S1]. It solves the boundary value problem by optimizing the initial velocity $\mathbf{v}_0$ of a ray propagated from $\mathbf{x}_0$ such that the ray intersects $\mathbf{y}$ [S1]. It is distinguished from Type-2 connections by its use of the explicit geometric constraint of the medium boundary, which provides a better-behaved loss function and faster convergence rates for surface intersections [S1].

## How it works
**Inputs:**
*   Source vertex $\mathbf{x}_0$ inside the medium.
*   Target vertex $\mathbf{y}$ on the medium boundary.
*   Implicit surface function $g(\mathbf{x})$ defining the boundary, where $g(\mathbf{y})=0$.
*   Refractive index field $n(\mathbf{x})$.
*   Initial guess for velocity $\mathbf{v}_0^0$.
*   Optimization parameters for BFGS (e.g., tolerance, max iterations).

**Outputs:**
*   Optimized initial velocity $\mathbf{v}_0^*$ that defines the refractive path from $\mathbf{x}_0$ to $\mathbf{y}$.

**Data Structures:**
*   Ray state: $\mathbf{x}$ (position), $\mathbf{v}$ (direction/velocity), $s$ (geometric distance).
*   Jacobian matrices: $\frac{\partial \mathbf{x}_s}{\partial \mathbf{v}_0}$ ($3 \times 3$), $\frac{\partial \mathbf{v}_s}{\partial \mathbf{v}_0}$ ($3 \times 3$).
*   Gradient vector: $\nabla n(\mathbf{x})$ ($3 \times 1$).
*   Hessian matrix: $H_n(\mathbf{x})$ ($3 \times 3$).

**Steps:**

1.  **Define Loss Function:**
    Propagate a ray from $\mathbf{x}_0$ with initial velocity $\mathbf{v}_0$ using Hamilton's equations until it reaches the boundary. Let $\mathbf{x}_{s^}$ be the position at geometric distance $s^*$, where $s^*$ is the distance at which $g(\mathbf{x}_{s^}) = 0$. Define the loss function:
    $$ \mathcal{L}(\mathbf{v}_0) = \frac{1}{2} \|\mathbf{x}_{s^} - \mathbf{y}\|^2 $$
    The goal is to find $\mathbf{v}_0$ that minimizes $\mathcal{L}(\mathbf{v}_0)$ [S1].

2.  **Compute Gradient of Loss:**
    The gradient required for BFGS optimization is:
    $$ \frac{d\mathcal{L}}{d\mathbf{v}_0} = (\mathbf{x}_{s^} - \mathbf{y})^T \frac{d\mathbf{x}_{s^}}{d\mathbf{v}_0} $$
    The total derivative $\frac{d\mathbf{x}_{s^}}{d\mathbf{v}_0}$ accounts for the dependence of the final position on both the initial velocity and the final distance $s^*$:
    $$ \frac{d\mathbf{x}_{s^}}{d\mathbf{v}_0} = \frac{\partial \mathbf{x}_{s^}}{\partial \mathbf{v}_0} + \frac{\partial \mathbf{x}_{s^}}{\partial s^} \frac{ds^*}{d\mathbf{v}_0} $$
    Here, $\frac{\partial \mathbf{x}_{s^}}{\partial s^}$ is the direction of the ray at $s^*$, denoted as $\mathbf{v}_{s^}$ normalized or scaled appropriately depending on the integration variable (geometric distance). Specifically, $\frac{\partial \mathbf{x}_{s^}}{\partial s^} = \frac{\mathbf{v}_{s^}}{n(\mathbf{x}_{s^})}$ if $s$ is geometric distance and the Hamiltonian formulation uses optical path length, but in the context of the provided derivation where $s$ is geometric distance and ray propagation updates position by $s \frac{\mathbf{v}}{n}$, the partial derivative of position with respect to distance is the unit tangent vector. However, the source defines the term $\frac{\partial \mathbf{x}_{s^}}{\partial s^}$ directly in the matrix equation. We calculate $\frac{ds^*}{d\mathbf{v}_0}$ using the implicit surface constraint [S1]:
    $$ \frac{ds^*}{d\mathbf{v}_0} = -\frac{\frac{dg(\mathbf{x}_{s^})}{d\mathbf{x}_{s^}} \frac{\partial \mathbf{x}_{s^}}{\partial \mathbf{v}_0}}{\frac{dg(\mathbf{x}_{s^})}{d\mathbf{x}_{s^}} \frac{\partial \mathbf{x}_{s^}}{\partial s^}} $$
    Substituting this back yields the final gradient expression:
    $$ \frac{d\mathbf{x}_{s^}}{d\mathbf{v}_0} = \left( I_{3 \times 3} - \frac{\frac{\partial \mathbf{x}_{s^}}{\partial s^} \frac{dg(\mathbf{x}_{s^})}{d\mathbf{x}_{s^}}}{\frac{dg(\mathbf{x}_{s^})}{d\mathbf{x}_{s^}} \frac{\partial \mathbf{x}_{s^}}{\partial s^}} \right) \frac{\partial \mathbf{x}_{s^}}{\partial \mathbf{v}_0} $$
    where $\frac{dg(\mathbf{x}_{s^})}{d\mathbf{x}_{s^}}$ is the gradient of the implicit surface function $\nabla g(\mathbf{x}_{s^})$ [S1].

3.  **Compute Partial Derivatives via Symplectic Integration:**
    The terms $\frac{\partial \mathbf{x}_{s^}}{\partial \mathbf{v}_0}$ and $\frac{\partial \mathbf{v}_{s^}}{\partial \mathbf{v}_0}$ are computed by integrating the variational equations alongside the ray propagation. Initialize $\frac{\partial \mathbf{x}_0}{\partial \mathbf{v}_0} = I_{3 \times 3}$ and $\frac{\partial \mathbf{v}_0}{\partial \mathbf{v}_0} = I_{3 \times 3}$ [S1]. For each integration step of size $s$ (geometric distance):
    *   Update velocity derivative:
        $$ \frac{\partial \mathbf{v}_{s}}{\partial \mathbf{v}_0} \leftarrow \frac{\partial \mathbf{v}_{s}}{\partial \mathbf{v}_0} + 0.5s H_n(\mathbf{x}) \frac{\partial \mathbf{x}_{s}}{\partial \mathbf{v}_0} $$
    *   Update position derivative:
        $$ \frac{\partial \mathbf{x}_{s}}{\partial \mathbf{v}_0} \leftarrow \frac{\partial \mathbf{x}_{s}}{\partial \mathbf{v}_0} + s \left( -\frac{\mathbf{v}_s \nabla n(\mathbf{x})}{n(\mathbf{x})^2} \frac{\partial \mathbf{x}_{s}}{\partial \mathbf{v}_0} + \frac{1}{n(\mathbf{x})} \frac{\partial \mathbf{v}_{s}}{\partial \mathbf{v}_0} \right) $$
    *   Update position: $\mathbf{x}_{s} \leftarrow \mathbf{x}_{s} + s \frac{\mathbf{v}_s}{n(\mathbf{x})}$
    *   Update velocity: $\mathbf{v}_{s} \leftarrow \mathbf{v}_{s} + 0.5s \nabla n(\mathbf{x})$
    *(Note: The source pseudocode interleaves these updates in a symplectic leapfrog-like manner. The exact order in the source is: update $\mathbf{v}$ derivative, update $\mathbf{x}$ derivative, update $\mathbf{x}$, update $\mathbf{v}$ derivative again, update $\mathbf{v}$. We follow the source's specific sequence for the derivatives)* [S1].

4.  **Optimize:**
    Use the BFGS algorithm to minimize $\mathcal{L}(\mathbf{v}_0)$ using the computed gradient. Repeat steps 2-3 for each BFGS iteration until convergence [S1].

**Pitfalls:**
*   **Undefined $s^*$ for non-solutions:** The geometric length $s^*$ is only strictly defined at the solution where $\mathbf{x}_{s^} = \mathbf{y}$. For arbitrary $\mathbf{v}_0$ during optimization, the ray may not hit $\mathbf{y}$. The algorithm mitigates this by assuming $\mathbf{x}_{s^}$ is the intersection with the boundary $g(\mathbf{x})=0$, making $s^*$ a well-defined function of $\mathbf{v}_0$ for all iterations [S1].
*   **Precision:** Computing $\frac{ds^*}{d\mathbf{v}_0}$ involves division by $\frac{dg(\mathbf{x}_{s^})}{d\mathbf{x}_{s^}} \frac{\partial \mathbf{x}_{s^}}{\partial s^}$. If the ray is tangent to the surface, this denominator approaches zero, causing numerical instability [S1].

## Subtopics

### [Multiple direct connections estimation](23243-multiple-direct-connections-estimation.md)
Multiple direct connections estimation is an unbiased Monte Carlo technique for computing the total throughput when multiple stationary refractive paths exist between two points in a heterogeneous medium. It extends the Type-1 Direct Connection Algorithm by estimating the inverse probability of sampling each stationary path, ensuring that the sum of contributions from all possible paths is correctly accounted for without bias. Unlike biased estimators that assume a single dominant path, this method handles path multiplicity by resampling initial velocities until a new distinct path is found or convergence fails.

## Operating characteristics
*   **Complexity:** The method requires iterative optimization (BFGS). Each iteration involves a full ray trace with variational integration, which is $O(N)$ where $N$ is the number of integration steps. The number of BFGS iterations depends on the complexity of the refractive index field and the initial guess.
*   **Performance:** Type-1 connections converge faster than Type-2 connections because the boundary constraint provides a more well-behaved loss function. The source notes that using Type-1 connections increases the number of converged cases by 10% compared to Type-2, reducing the need for reinitialization [S1].
*   **Strengths:** Handles heterogeneous media exactly (unbiased). Efficient for connections involving medium boundaries.
*   **Limitations:** Requires the implicit surface representation $g(\mathbf{x})$. Fails or converges poorly if the ray is tangent to the boundary. Computationally expensive due to the need for gradient computation and optimization.

## Background
The Type-1 Direct Connection Algorithm was introduced in the context of path tracing estimators for refractive radiative transfer [S1]. It addresses the boundary value problem of finding a curve satisfying Hamilton's equations between two vertices, one on a medium boundary and one inside. It is part of a broader framework that includes Type-2 connections (both vertices inside) and multiple direct connections estimation [S1].

## Sources
[S1] Path tracing estimators for refractive radiative transfer, https://doi.org/10.1145/3414685.3417793, fetched 2026-10-29
