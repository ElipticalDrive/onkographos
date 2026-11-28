# Edge Point Re-parametrization

Also called: Implicit Surface Deformation via Re-parametrization, Learned Surface Displacement

## Overview
Edge Point Re-parametrization provides an explicit, closed-form re-parametrization of the surface point $x$ for implicit surfaces, enabling gradients to flow through surface geometry during inverse rendering optimization [S1]. It solves the problem that the root of $S(x)=0$ is not directly differentiable in standard automatic differentiation frameworks when $x$ is the output of a solver [S1].

## How it works
**Inputs and Outputs**
- Input: An initial edge point $x$, the SDF function $S_{\Theta_s}$ parameterized by $\Theta_s$, and the surface normal $n = \nabla S_{\Theta_s}(x)$ [S1].
- Output: A re-parametrized surface point $x_{\Theta_s}$ that lies on the zero-level set of $S_{\Theta_s}$ and is differentiable with respect to $\Theta_s$ [S1].

**Data Structures**
- $x$: 3D vector $(x, y, z)$ [S1].
- $n$: 3D vector $(n_x, n_y, n_z)$, used in the dot product $n^T n$ [S1].
- $S_{\Theta_s}(x)$: Scalar SDF value [S1].
- $\Theta_s$: Parameters of the implicit surface representation (e.g., MLP weights) [S1].

**Steps**
1. **Define Deformation Direction**: The surface point moves along the surface normal $n$ from the initial point $x$. The deformed point is defined as $x_{\Theta_s} = x + t_{\Theta_s} \cdot n$, where $t_{\Theta_s}$ is a scalar displacement [S1].
2. **Initial Condition**: At initialization $\Theta_s = \Theta_s^{(0)}$, the displacement is zero, ensuring $x_{\Theta_s} = x$ [S1].
3. **Zero-Level Set Constraint**: The deformed point must satisfy $S_{\Theta_s}(x_{\Theta_s}) = 0$ [S1].
4. **Implicit Differentiation**: Differentiating the constraint with respect to $\Theta_s$ yields $\frac{\partial S}{\partial \Theta_s}(x_{\Theta_s}) + \left( \frac{\partial S}{\partial x_{\Theta_s}} \right)^T \frac{\partial x_{\Theta_s}}{\partial \Theta_s} = 0$ [S1].
5. **Evaluate at Initialization**: At $\Theta_s = \Theta_s^{(0)}$, the gradient of the displacement is $\frac{\partial t}{\partial \Theta_s} \big|_{\Theta_s = \Theta_s^{(0)}} = -\frac{1}{n^T n} \cdot \frac{\partial S}{\partial \Theta_s}(x) \big|_{\Theta_s = \Theta_s^{(0)}}$ [S1].
6. **First-Order Approximation**: The scalar displacement is approximated as $t_{\Theta_s} \approx -\frac{1}{n^T n} \cdot S_{\Theta_s}(x)$ [S1].
7. **Final Re-parametrization**: Substituting $t_{\Theta_s}$ back yields $x_{\Theta_s} = x - \frac{n}{n^T n} \cdot S_{\Theta_s}(x)$ [S1].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def edge_point_reparametrization(x, S_val, n):
    """
    Computes the re-parametrized surface point x_theta.
    Args:
        x: Initial 3D point (tensor).
        S_val: SDF value at x, S_theta(x) (scalar tensor).
        n: Surface normal vector (3D vector tensor).
    Returns:
        x_theta: Re-parametrized point on the zero-level set.
    """
    n_dot_n = n[0]*n[0] + n[1]*n[1] + n[2]*n[2]
    if n_dot_n < 1e-8:
        return x
    t = -S_val / n_dot_n
    x_theta = x + t * n
    return x_theta
```

**Parameters**
- $x$: Initial intersection estimate [S1].
- $S_{\Theta_s}(x)$: SDF value [S1].
- $n$: Normal vector $\nabla S_{\Theta_s}(x)$ [S1].

**Edge Cases and Pitfalls**
- **Degenerate Normals**: If $n$ is zero, $n^T n$ is zero, leading to division by zero. A guard is required [G].
- **First-Order Approximation**: The method provides an unbiased first-order gradient with respect to $\Theta_s$, suitable for gradient-based optimizers, but may not capture higher-order surface curvature effects [S1].
- **Initialization**: The method assumes the initial point $x$ is close enough to the surface such that the first-order approximation is valid [G].

## Subtopics

### [Differentiable Surface Point Reparameterization](15363-differentiable-surface-point-reparameterization.md)
Reparameterizes the surface intersection equation to make the surface point differentiable with respect to the SDF MLP parameters. It is chosen when differentiable rendering of complex materials and geometry is required simultaneously. [Differentiable Surface Point Reparameterization](15363-differentiable-surface-point-reparameterization.md)

## Operating characteristics
**Time and Memory Complexity**
- Time: $O(1)$ per ray, assuming $S_{\Theta_s}(x)$ and $\nabla S_{\Theta_s}(x)$ are computed in $O(1)$ [G].
- Memory: $O(1)$ additional memory per ray [G].

**Measured Performance**
- Not given in the sources.

**Scaling**
- Scales linearly with the number of rays [G].

**Strengths and Limitations**
- **Strengths**: Enables gradient-based optimization of implicit surfaces in inverse rendering [S1].
- **Limitations**: First-order approximation may be inaccurate for large displacements or highly curved surfaces [S1].

**When to Use**
- Use when optimizing neural implicit surfaces (SDFs) for inverse rendering tasks where geometry needs to be updated via gradient descent [S1].

**When Not to Use**
- Do not use for static rendering where geometry is fixed [G]. Do not use if high-order curvature accuracy is critical and the first-order approximation is insufficient [S1].

## Background
Originated in the context of inverse rendering and neural implicit surface optimization [S1]. The method was introduced in the IRON paper (2022) to enable differentiable ray-surface intersection for optimizing Neural SDFs and materials from photometric images [S1].

## Sources
[S1] IRON: Inverse Rendering by Optimizing Neural SDFs and Materials from Photometric Images, https://doi.org/10.1109/cvpr52688.2022.00548, fetched 2026-10-26
