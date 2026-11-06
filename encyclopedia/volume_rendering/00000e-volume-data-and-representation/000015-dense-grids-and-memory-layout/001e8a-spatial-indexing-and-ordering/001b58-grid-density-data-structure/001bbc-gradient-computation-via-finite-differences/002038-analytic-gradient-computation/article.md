# Analytic Gradient Computation

Also called: Analytic Basis Gradients, Analytic Derivatives of Basis Reconstruction

## Overview
Analytic Gradient Computation computes the 3D gradient vector for local shading (BRDF) in volume rendering by evaluating the exact first-order partial derivatives of the basis reconstruction function [S1]. It sets itself apart from [Gradient Computation via Finite Differences](1077-gradient-computation-via-finite-differences.md) by requiring no additional memory accesses or ray traversals, relying solely on the data values already loaded for the scalar evaluation [S1]. Unlike [Gradient Map Generation](1204-gradient-map-generation.md) which precomputes gradients, this method computes them on-the-fly during ray marching [S1].

## How it works
**Inputs:**
- Sample point $p = (p_x, p_y, p_z)$ in world space.
- Cell width $C_w$.
- Cell center coordinates $C_p = (C_{px}, C_{py}, C_{pz})$.
- Basis function component $\hat{h}(\cdot)$.
- Scalar data values $C_v$ and basis weights $\hat{H}_C(p)$ for all contributing cells $C$.

**Outputs:**
- Gradient vector $\nabla B(p) = (\frac{\partial B(p)}{\partial x}, \frac{\partial B(p)}{\partial y}, \frac{\partial B(p)}{\partial z})$.

**Data Structures:**
- The method operates on the scalar reconstruction result $B(p)$ and its components. No new storage is required beyond registers for the gradient components.

**Steps:**
1. Compute the scalar reconstruction $B(p)$ and the weighted sum of basis derivatives $\sum_C \frac{\partial \hat{H}_C(p)}{\partial x} C_v$ for each axis. This uses the same data values loaded for the scalar evaluation [S1].
2. Compute the partial derivative of the basis function $\frac{\partial \hat{H}_C(p)}{\partial x}$ for each cell $C$ using the analytic formula below.
3. Compute the partial derivative of the reconstructed scalar field $B(p)$ using the quotient rule applied to the basis reconstruction equation [S1]:

$$
\frac{\partial B(p)}{\partial x} = \frac{\sum_C \hat{H}_C(p) \sum_C \frac{\partial \hat{H}_C(p)}{\partial x} C_v - \sum_C \hat{H}_C(p) C_v \sum_C \frac{\partial \hat{H}_C(p)}{\partial x}}{(\sum_C \hat{H}_C(p))^2}
$$

4. Repeat Step 3 for $y$ and $z$ axes to obtain the full gradient vector.

**Formulas:**
The partial derivative of the basis function component $\hat{H}_C(p)$ with respect to $x$ is:

$$
\frac{\partial \hat{H}(p)}{\partial x} = \hat{h}\left(\frac{|C_{py} - p_y|}{C_w}\right) \hat{h}\left(\frac{|C_{pz} - p_z|}{C_w}\right) \chi(x) \frac{1}{C_w}
$$

The sign function $\chi(t)$ is defined as:

$$
\chi(t) = \begin{cases} -1, & \text{if } C_{pt} - p_t \geq 0 \\ 1, & \text{otherwise} \end{cases}
$$

**Parameters:**
- $C_w$: Cell width. Typical values depend on the specific AMR level and region resolution [S1].

**Edge Cases and Pitfalls:**
- The analytic gradient is not continuously differentiable. This leads to shading artifacts at cell and level boundaries, whereas central differences are continuous but expensive [S1].
- The sign function $\chi(t)$ introduces discontinuities in the derivative at the cell boundaries ($C_{pt} - p_t = 0$) [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_analytic_gradient(p, C_p, C_w, basis_weights, data_values):
    """
    Computes the gradient of the basis reconstruction function analytically.
    
    Args:
        p: Sample point (x, y, z)
        C_p: Cell center coordinates (C_px, C_py, C_pz) for each contributing cell
        C_w: Cell width (scalar)
        basis_weights: List of basis function values H_C(p) for each cell
        data_values: List of scalar data values C_v for each cell
        
    Returns:
        Gradient vector (dx, dy, dz)
    """
    num_cells = len(basis_weights)
    
    sum_H = 0.0
    sum_H_Cv = 0.0
    sum_dH_dx_Cv = 0.0
    sum_dH_dy_Cv = 0.0
    sum_dH_dz_Cv = 0.0
    
    sum_H_dH_dx = 0.0
    sum_H_dH_dy = 0.0
    sum_H_dH_dz = 0.0
    
    for i in range(num_cells):
        H = basis_weights[i]
        Cv = data_values[i]
        
        # Compute basis derivative components
        # dH/dx = h(|C_py - p_y|/C_w) * h(|C_pz - p_z|/C_w) * chi(x) * (1/C_w)
        # Note: The formula in [S1] shows dH/dx depends on y and z distances via h, 
        # and x distance via chi(x). This assumes a separable basis function structure.
        
        dist_y = abs(C_p[i].y - p.y)
        dist_z = abs(C_p[i].z - p.z)
        
        h_y = basis_function_component(dist_y / C_w)
        h_z = basis_function_component(dist_z / C_w)
        
        # Chi function for x
        if C_p[i].x - p.x >= 0:
            chi_x = -1
        else:
            chi_x = 1
            
        dH_dx = h_y * h_z * chi_x * (1.0 / C_w)
        
        # Chi function for y
        if C_p[i].y - p.y >= 0:
            chi_y = -1
        else:
            chi_y = 1
            
        dist_x = abs(C_p[i].x - p.x)
        dist_z_y = abs(C_p[i].z - p.z)
        h_x = basis_function_component(dist_x / C_w)
        h_z_y = basis_function_component(dist_z_y / C_w)
        
        dH_dy = h_x * h_z_y * chi_y * (1.0 / C_w)
        
        # Chi function for z
        if C_p[i].z - p.z >= 0:
            chi_z = -1
        else:
            chi_z = 1
            
        dist_x_z = abs(C_p[i].x - p.x)
        dist_y_z = abs(C_p[i].y - p.y)
        h_x_z = basis_function_component(dist_x_z / C_w)
        h_y_z = basis_function_component(dist_y_z / C_w)
        
        dH_dz = h_x_z * h_y_z * chi_z * (1.0 / C_w)
        
        # Accumulate sums for the quotient rule
        sum_H += H
        sum_H_Cv += H * Cv
        sum_dH_dx_Cv += dH_dx * Cv
        sum_dH_dy_Cv += dH_dy * Cv
        sum_dH_dz_Cv += dH_dz * Cv
        
        sum_H_dH_dx += H * dH_dx
        sum_H_dH_dy += H * dH_dy
        sum_H_dH_dz += H * dH_dz
        
    # Denominator
    denom = sum_H * sum_H
    
    if denom == 0:
        return (0.0, 0.0, 0.0)
        
    # Apply quotient rule
    dx = (sum_H_dH_dx * sum_H_Cv - sum_H_Cv * sum_dH_dx_Cv) / denom
    dy = (sum_H_dH_dy * sum_H_Cv - sum_H_Cv * sum_dH_dy_Cv) / denom
    dz = (sum_H_dH_dz * sum_H_Cv - sum_H_Cv * sum_dH_dz_Cv) / denom
    
    return (dx, dy, dz)

def basis_function_component(t):
    """
    Placeholder for the specific basis function component h(t).
    The exact form of h(t) depends on the basis reconstruction method used.
    """
    # Not given in the sources. Typically a box spline or similar interpolant.
    return 0.0 # [G]
```

## Subtopics

### [Analytic Gradient Computation via Box-Spline Derivatives](3199-analytic-gradient-computation-via-box-spline-derivatives.md)
This method computes exact gradient vectors for local shading in volume rendering by evaluating analytic derivatives of the underlying box-spline basis function, avoiding the inaccuracies and extra memory bandwidth of finite differences. It sets itself apart from [Analytic Gradient Computation](2167-analytic-gradient-computation.md) by specifically exploiting the symmetry and structure of Face-Centered Cubic (FCC) box-splines to reuse the 19 samples already fetched for scalar evaluation, whereas finite difference methods require six additional full spline evaluations.

### [Analytic Gradient Computation for Triangle Distributions](5829-analytic-gradient-computation-for-triangle-distributions.md)
This method computes the exact Fourier Transform $\hat{u}_\mathcal{T}$ and its spatial gradients for a single triangle $\mathcal{T}$ defined by vertices $(\mathbf{v}_1, \mathbf{v}_2, \mathbf{v}_3)$. It provides analytic derivatives of the basis reconstruction function, avoiding additional samples or ray traversals [G].

## Operating characteristics
- **Time Complexity:** $O(N)$ where $N$ is the number of contributing cells per sample. No additional ray traversals or memory accesses are required beyond the scalar evaluation [S1].
- **Memory Complexity:** $O(1)$ additional memory per sample, as it reuses existing data values [S1].
- **Measured Performance:** Analytic gradients are much faster to compute than both central difference variants [S1].
- **Strengths:** Superior quality compared to clamped central differences; no additional memory bandwidth cost [S1].
- **Limitations:** Not continuously differentiable, leading to shading artifacts at cell and level boundaries [S1].
- **When to use:** When performance is critical and the discontinuities at cell boundaries are acceptable or mitigated by other means. Used by default in the ExaBricks renderer [S1].
- **When not to use:** When continuous gradients are required to avoid shading artifacts, in which case central differences may be preferred despite the cost [S1].

## Background
- **Origin:** The method was introduced in the ExaBricks renderer for ray tracing structured AMR data [S1].
- **History:** It was developed as an alternative to central differencing to avoid the cost of additional samples and ray traversals [S1].
- **Usage:** Used in the ExaBricks renderer for local shading with BRDFs [S1].

## Sources
[S1] Ray Tracing Structured AMR Data Using ExaBricks, https://www.sci.utah.edu/~wald/Publications/2020/exabrick/exabrick.pdf, fetched 2026-10-30
