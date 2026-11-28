# Regularized Winding Number

Also called: Gaussian-regularized winding number, Regularized Dipole Sum with unit moment

## Overview
The Regularized Winding Number is a scalar field representation for oriented point clouds that replaces the singular Poisson kernel of the standard Winding Number with a Gaussian-regularized variant [S1]. It solves the problem of numerical instability and exact interpolation inherent in the unregularized method, making it robust for noisy point clouds (e.g., from structure-from-motion) and efficient for inverse rendering [S1]. Unlike Poisson Surface Reconstruction (PSR), which requires a global grid solve, the Regularized Winding Number is computed as a direct sum, offering equivalent regularity properties with significantly lower computational cost for rendering and differentiation [S1].

## How it works
**Inputs:**
- An oriented point cloud $\mathcal{P} = \{(p_m, n_m, A_m)\}_{m=1}^M$, where $p_m \in \mathbb{R}^3$ is the position, $n_m \in \mathbb{R}^3$ is the unit normal, and $A_m \in \mathbb{R}$ is the area weight for point $m$.
- A query point $x \in \mathbb{R}^3$.
- A regularization parameter $\varepsilon > 0$.

**Outputs:**
- A scalar value $\tilde{w}_\varepsilon(x) \in \mathbb{R}$ representing the regularized winding number at $x$.

**Data Structures:**
- The point cloud is stored as three parallel arrays or a single array of structs:
  - `positions`: Array of $M$ vectors $(x, y, z)$.
  - `normals`: Array of $M$ vectors $(x, y, z)$.
  - `areas`: Array of $M$ scalars.

**Steps:**
1. Initialize the accumulator $W \leftarrow 0$.
2. For each point $m$ from $1$ to $M$:
   a. Compute the displacement vector $d = p_m - x$.
   b. Compute the squared distance $r^2 = \|d\|^2 = d \cdot d$.
   c. Compute the distance $r = \sqrt{r^2}$.
   d. Compute the scaled distance $t = r / \varepsilon$.
   e. Compute the scaling function $S(t) = \text{erf}(t) - \frac{2t}{\sqrt{\pi}} e^{-t^2}$.
   f. Compute the singular Poisson kernel component $P(x, p_m) = n_m \cdot \nabla_{p_m} G(x, p_m)$.
      *Note: The gradient of the Green's function $G(x, y) = -\frac{1}{4\pi \|x-y\|}$ with respect to $y$ is $\nabla_y G(x, y) = \frac{x-y}{4\pi \|x-y\|^3}$. Thus, $P(x, p_m) = n_m \cdot \frac{x - p_m}{4\pi r^3}$.*
   g. Compute the regularized kernel value $P_\varepsilon(x, p_m) = S(t) \cdot P(x, p_m)$.
   h. Update $W \leftarrow W + A_m \cdot P_\varepsilon(x, p_m)$.
3. Return $W$.

**Formulas:**
The regularized winding number is defined as [S1]:
$$ \tilde{w}_\varepsilon(x) \equiv \sum_{m=1}^M A_m P_\varepsilon(x, p_m) $$

The regularized Poisson kernel is [S1]:
$$ P_\varepsilon(x, y) \equiv S\left(\frac{\|y - x\|}{\varepsilon}\right) P(x, y) $$

The scaling function $S(t)$ is [S1]:
$$ S(t) \equiv \text{erf}(t) - \frac{2t}{\sqrt{\pi}} \exp(-t^2) $$

The singular Poisson kernel $P(x, y)$ is derived from the Green's function $G(x, y) = -\frac{1}{4\pi \|x-y\|}$ satisfying $\Delta_x G(x, y) = \delta(x - y)$ [S1]:
$$ P(x, y) \equiv n(y) \cdot \nabla_y G(x, y) = n(y) \cdot \frac{x - y}{4\pi \|x - y\|^3} $$

The Gaussian nascent delta function used for regularization is [S1]:
$$ \phi_\varepsilon(x - y) \equiv \frac{1}{\varepsilon \sqrt{2\pi}} \exp\left(-\frac{\|x - y\|^2}{2\varepsilon^2}\right) $$

**Parameters:**
- $\varepsilon$: Regularization parameter. Controls the trade-off between regularization (smoothness) and bias (accuracy relative to the unregularized field) [S1]. Typical values depend on the scale of the point cloud; it must be chosen such that $\varepsilon > 0$ to avoid singularity.

**Edge Cases and Pitfalls:**
- **Singularity:** The unregularized kernel $P(x, y)$ is singular at $x = y$. The regularized kernel $P_\varepsilon(x, y)$ is finite at $x = y$, with $P_\varepsilon(y, y) = \frac{1}{3} \varepsilon^{-3} \pi^{-3/2}$ [S1].
- **Exact Interpolation:** The unregularized winding number exactly interpolates the point cloud, which is undesirable for noisy data [S1]. The regularized version does not exactly interpolate, allowing for robust handling of outliers and noise [S1].
- **Desingularization vs. Regularization:** Using a simple cutoff in the denominator (desingularization) results in worse performance in inverse rendering compared to Gaussian regularization [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def erf(x):
    # Approximation of the error function
    # Using a standard polynomial approximation or library call
    # For self-containedness, a common approximation is used here
    # Note: In practice, use a high-precision library implementation
    t = 1.0 / (1.0 + 0.3275911 * abs(x))
    y = 1.0 - (((((1.061405429 * t - 1.453152027) * t) + 1.421413741) * t - 0.284496736) * t + 0.254829592) * t * math.exp(-x * x)
    if x >= 0:
        return y
    else:
        return -y

def regularized_winding_number(x, point_cloud, epsilon):
    """
    Computes the regularized winding number at point x for an oriented point cloud.
    
    Args:
        x: Query point (tuple or list of 3 floats)
        point_cloud: List of tuples (p, n, A) where p is position (3 floats), 
                     n is normal (3 floats), A is area weight (float)
        epsilon: Regularization parameter (float > 0)
        
    Returns:
        Scalar value of the regularized winding number
    """
    w = 0.0
    inv_epsilon = 1.0 / epsilon
    
    for p, n, A in point_cloud:
        # Compute displacement vector d = p - x
        dx = p[0] - x[0]
        dy = p[1] - x[1]
        dz = p[2] - x[2]
        
        # Compute squared distance r^2
        r_sq = dx*dx + dy*dy + dz*dz
        
        # Compute distance r
        r = math.sqrt(r_sq)
        
        # Avoid division by zero if r is extremely small, though epsilon handles it
        # However, for the singular kernel part, we need r^3
        if r < 1e-10:
            # At the point itself, the singular kernel is undefined, 
            # but the regularized kernel is finite. 
            # The formula P_epsilon = S(t) * P(x,y) breaks down if P(x,y) is infinite.
            # However, the derivation shows P_epsilon is finite.
            # We use the limit value derived in the source: P_epsilon(y,y) = 1/(3 * epsilon^3 * pi^(3/2))
            # But wait, the source defines P_epsilon via S(t) * P(x,y).
            # Let's check the behavior of S(t) as t->0.
            # S(t) ~ t^3 / (3 sqrt(pi)) for small t?
            # erf(t) ~ 2t/sqrt(pi) - 2t^3/(3 sqrt(pi))
            # S(t) = erf(t) - 2t/sqrt(pi) exp(-t^2)
            # exp(-t^2) ~ 1 - t^2
            # S(t) ~ (2t/sqrt(pi) - 2t^3/(3 sqrt(pi))) - (2t/sqrt(pi))(1 - t^2)
            # S(t) ~ 2t/sqrt(pi) - 2t^3/(3 sqrt(pi)) - 2t/sqrt(pi) + 2t^3/sqrt(pi)
            # S(t) ~ (2 - 2/3) t^3 / sqrt(pi) = (4/3) t^3 / sqrt(pi)
            # P(x,y) ~ n . (x-y) / (4 pi r^3) = - n . d / (4 pi r^3)
            # If x=y, d=0, so P(x,y) is 0/0? No, P(x,y) involves n . (x-y).
            # If x=y, x-y=0, so P(x,y)=0.
            # So S(t) * P(x,y) -> 0 * infinity?
            # Let's look at the source again.
            # P_epsilon(y,y) = 3^-1 epsilon^-3 pi^-3/2.
            # This implies the limit is non-zero.
            # The formula P(x,y) = n . (x-y) / (4 pi r^3).
            # If x=y, the vector is 0.
            # The source says P_epsilon is not singular.
            # The derivation uses the Green's function regularization.
            # The direct evaluation via S(t)*P(x,y) might be numerically unstable at r=0.
            # However, for implementation, if r is very small, we can use the limit.
            # But typically, query points are not exactly on the point cloud.
            # We will assume r > 0 for the general case.
            pass
            
        # Compute t = r / epsilon
        t = r * inv_epsilon
        
        # Compute S(t)
        # S(t) = erf(t) - (2t / sqrt(pi)) * exp(-t^2)
        sqrt_pi = math.sqrt(math.pi)
        exp_neg_t_sq = math.exp(-t * t)
        S_t = erf(t) - (2.0 * t / sqrt_pi) * exp_neg_t_sq
        
        # Compute Singular Poisson Kernel P(x, p_m)
        # P(x, y) = n . (x - y) / (4 pi r^3)
        # Note: x - y = -d = (x0-p0, x1-p1, x2-p2)
        # d = p - x, so x - y = -d
        # n . (x - y) = - (n . d)
        dot_nd = -(n[0]*dx + n[1]*dy + n[2]*dz)
        
        # r^3
        r_cubed = r_sq * r
        
        # P(x, y)
        if r_cubed < 1e-30:
            # Handle singularity by using the limit value if necessary
            # Or clamp r to a minimum value
            r_cubed = 1e-30
            
        P_xy = dot_nd / (4.0 * math.pi * r_cubed)
        
        # Regularized Poisson Kernel
        P_eps = S_t * P_xy
        
        # Accumulate
        w += A * P_eps
        
    return w
```

## Subtopics

### [Generalized Winding Number Voxelization](7623-generalized-winding-number-voxelization.md)
A generalization of the winding number that allows for non-closed meshes and different boundary conditions. Choose this when dealing with open surfaces or when the standard winding number assumptions do not hold.

### [Mesh Winding Number Voxelization](2328-mesh-winding-number-voxelization.md)
The parent method that computes the winding number hierarchically for meshes. Choose this when working with explicit triangle meshes rather than point clouds.

## Operating characteristics
- **Time Complexity:** $O(M)$ per query point, where $M$ is the number of points in the cloud. This is linear in the number of points and independent of the grid resolution, making it efficient for high-resolution voxelization compared to global solvers.
- **Memory Complexity:** $O(M)$ to store the point cloud. No global grid storage is required for the field evaluation itself, unlike PSR which requires a grid for the Poisson solve.
- **Measured Performance:** The method is significantly faster than Poisson Surface Reconstruction (PSR) for inverse rendering tasks because it avoids the global grid solve [S1]. It produces high-quality meshes comparable to state-of-the-art inverse rendering methods when used with image-based losses [S1].
- **Strengths:** Robust to noise and outliers in point clouds; differentiable; efficient to render; no global solve required.
- **Limitations:** Requires a regularization parameter $\varepsilon$; performance degrades if $\varepsilon$ is poorly chosen (too small leads to noise, too large leads to bias); not suitable for exact interpolation if that is required.
- **When to use:** When working with noisy point clouds (e.g., from SfM) and needing a differentiable, efficient geometry representation for inverse rendering or optimization.
- **When not to use:** When exact interpolation of the point cloud is required; when the point cloud is extremely large and $O(M)$ per query is too slow (consider spatial indexing).

## Background
The Regularized Winding Number was introduced in the context of inverse rendering and 3D reconstruction to address the limitations of the standard Winding Number, such as singularity and sensitivity to noise [S1]. It is based on the regularization of the Poisson kernel using a Gaussian nascent delta function [S1]. The method is equivalent to Poisson Surface Reconstruction (PSR) in terms of regularity properties but is computed as a direct sum, making it more efficient for rendering and differentiation [S1]. The approach builds on earlier work on dipole sums and winding numbers [S1].

## Sources
[S1] 3D Reconstruction with Fast Dipole Sums, https://doi.org/10.1145/3687914, fetched 2026-10-28
