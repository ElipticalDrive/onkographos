# Cylindrical Symmetry Projector

Also called: Abel Transform, Abel Transform Discretization

## Overview
The Cylindrical Symmetry Projector is a forward and back projector pair designed for objects possessing cylindrical symmetry, serving as a specialized case of the Forward Projection (aTx) method [S2, S3, unofficial]. It solves the problem of computing line integrals through axisymmetric volumes by reducing the 3D integration to a 1D radial problem, specifically implementing the Abel Transform for parallel-beam geometries [S2, S3, unofficial]. Unlike general 3D projectors, it exploits rotational invariance to achieve significant computational efficiency, distinguishing it from siblings like Analytic 3D Gaussian Ray Tracing for Tomography which handle arbitrary 3D distributions without symmetry assumptions [S2, S3, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A radial extinction coefficient profile $\alpha(r)$ defined for $r \in [0, r_{max}]$.
*   **Output:** The total optical depth $\delta_{max}(x)$ for a set of projection rays at impact parameter $x$.

**Data Structures:**
*   **Radial Profile:** A vector $\alpha$ of size $N_r$, representing $\alpha(r_j)$ at discrete radii $r_j$.
*   **Projection Vector:** A vector $\Delta$ of size $N_x$, representing $\delta_{max}(x_i)$ at discrete impact parameters $x_i$.
*   **Kernel Matrix:** A matrix $K$ of size $N_x \times N_r$, where element $K_{ij}$ represents the geometric path length of the $i$-th ray passing through the $j$-th concentric ring [S1, unofficial].

**Steps:**
1.  **Discretization:** Discretize the spray cross-section into $N_r$ concentric rings to approximate the continuous integral as a finite sum [S1, unofficial].
2.  **Kernel Construction:** Compute the geometric kernel matrix $K$. The element $K_{ij}$ corresponds to the path length of the $i$-th ray passing through the $j$-th ring [S1, unofficial].
3.  **Forward Projection:** Compute the total optical depth vector $\Delta$ using the linear algebraic form:
    $$ \Delta = K A $$
    where $A$ is the vector of unknown radial extinction coefficients $\alpha(r)$ [S1, unofficial].
4.  **Back Projection:** To reconstruct $\alpha(r)$ from measured $\delta_{max}(x)$, solve the linear system $K A = \Delta$ for $A$. This typically involves inverting or pseudo-inverting the kernel matrix $K$, potentially with regularization if the system is ill-conditioned [S1, unofficial].

**Formulas:**
Under axisymmetry, the total optical depth $\delta_{max}(x)$ is the Abel transform of the local extinction-coefficient profile $\alpha(r)$ [S1, unofficial]:
$$ \delta_{max}(x) = 2 \int_{x}^{r_{max}} \frac{\alpha(r) r}{\sqrt{r^2 - x^2}} dr $$
This integral is approximated by the discrete sum:
$$ \delta_{max}(x_i) = \sum_{j=1}^{N_r} K_{ij} \alpha(r_j) $$
where $K_{ij}$ encodes the integral of the geometric factor over the $j$-th ring for the $i$-th ray [S1, unofficial].

**Parameters:**
*   $r_{max}$: Maximum radius of the object.
*   $x$: Impact parameter (distance from the axis of symmetry to the ray).
*   $N_r, N_x$: Number of radial bins and projection samples.

**Edge Cases and Pitfalls:**
*   **Singularity:** The kernel has a singularity at $r=x$. Numerical integration schemes must handle this carefully [S1, unofficial].
*   **Ill-Conditioning:** The Abel transform is an ill-posed inverse problem. Small errors in $\delta_{max}(x)$ can lead to large errors in $\alpha(r)$ without regularization [S1, unofficial].
*   **Boundary Conditions:** The value of $\alpha(r)$ at $r_{max}$ is assumed to be zero or negligible [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def cylindrical_symmetry_projector(alpha, r_grid, x_grid):
    """
    Forward projection using Cylindrical Symmetry (Abel Transform).
    
    Args:
        alpha: Array of extinction coefficients alpha(r) at radii r_grid.
        r_grid: Array of radial positions r_j.
        x_grid: Array of impact parameters x_i.
        
    Returns:
        delta_max: Array of total optical depths delta_max(x_i).
    """
    N_r = len(r_grid)
    N_x = len(x_grid)
    delta_max = zeros(N_x)
    
    # Precompute kernel matrix K
    K = zeros((N_x, N_r))
    for i in range(N_x):
        x_i = x_grid[i]
        for j in range(N_r):
            r_j = r_grid[j]
            # Path length through ring j for ray at x_i
            # This is the geometric kernel element K_ij
            # For a thin ring approximation, this might be computed analytically
            # or via numerical integration of the Abel kernel over the ring width
            if r_j >= x_i:
                # Simplified geometric path length for a thin ring at r_j
                # Exact formula depends on ring discretization scheme
                # Common approximation: 2 * r_j / sqrt(r_j^2 - x_i^2) * dr_j
                # But K_ij should represent the integral over the ring
                # Here we assume a precomputed K or a helper function
                K[i, j] = compute_path_length(x_i, r_j, r_grid)
            else:
                K[i, j] = 0.0
                
    # Matrix-vector multiplication: Delta = K * A
    for i in range(N_x):
        sum_val = 0.0
        for j in range(N_r):
            sum_val += K[i, j] * alpha[j]
        delta_max[i] = sum_val
        
    return delta_max

def compute_path_length(x, r, r_grid):
    """
    Computes the geometric kernel element K_ij for ray at x and ring at r.
    This is a placeholder for the specific discretization scheme.
    [G] In practice, this involves integrating 2*r/sqrt(r^2-x^2) over the ring width.
    """
    # Example: Simple approximation for a ring centered at r with width dr
    dr = r_grid[1] - r_grid[0] if len(r_grid) > 1 else 1.0
    if r > x:
        return 2.0 * r / sqrt(r**2 - x**2) * dr
    else:
        return 0.0
```

## Subtopics

### Abel Transform
The specific mathematical operation applied in parallel-beam geometries. It is the core of the Cylindrical Symmetry Projector. Choose it when the object is strictly axisymmetric and the projection geometry is parallel. Link: [Abel Transform](abel-transform.md).

### Radial Discretization
The method of dividing the radial profile into concentric rings. Choose finer discretization for higher accuracy but increased computational cost. Link: [Radial Discretization](radial-discretization.md).

## Operating characteristics
*   **Time Complexity:** $O(N_x N_r)$ for the forward projection if the kernel matrix is precomputed. The back projection (inversion) depends on the solver used, typically $O(N_r^2)$ or $O(N_r \log N_r)$ with fast algorithms.
*   **Memory Complexity:** $O(N_x N_r)$ to store the kernel matrix $K$.
*   **Strengths:** Extremely efficient for axisymmetric objects; reduces 3D problem to 1D.
*   **Limitations:** Only applicable to objects with cylindrical symmetry; sensitive to noise in the input data due to the ill-posed nature of the inverse Abel transform.
*   **When to use:** When the object is known to be axisymmetric and high performance is required.
*   **When not to use:** For arbitrary 3D objects or objects with significant asymmetry.

## Background
The Cylindrical Symmetry Projector, specifically the Abel Transform, has a long history in physics and engineering, particularly in plasma physics and combustion diagnostics [S1, S2, S3, unofficial]. It was implemented as part of a differentiable forward projector for X-ray CT, allowing for gradient-based optimization in tomographic reconstruction [S2, S3, unofficial]. The discretization approach described here follows the work of Champley & Maddox (2021) [S2, S3, unofficial].

## Sources
[S1] Tomographic Reconstruction of Infrared Radiance from Axisymmetric Sprays in a Hot Environment Using Multi-Parameter Regularization, https://doi.org/10.21203/rs.3.rs-10907544/v1, fetched 2026-10-26 (unofficial)
[S2] PDF Differentiable Forward Projector for X-ray Computed Tomography, https://differentiable.xyz/papers-2023/paper_48.pdf, fetched 2026-10-29 (unofficial)
[S3] Differentiable Forward Projector for X-ray Computed ..., https://openreview.net/forum?id=dX8khDFGHv, fetched 2026-11-06 (unofficial)
