# Analytic Gradient Computation

Also called: Analytic Basis Gradients, Analytic Derivatives of Basis Reconstruction

## Overview
Analytic Gradient Computation computes gradient vectors for local shading (BRDF) in volume rendering using analytic derivatives of the basis reconstruction function, avoiding additional samples or ray traversals [S1]. It sets itself apart from [Gradient Computation via Finite Differences](1077-gradient-computation-via-finite-differences.md) by requiring no additional sample evaluations or ray traversals, relying solely on the data values already loaded for the scalar reconstruction [S1].

## How it works
**Inputs and Outputs:**
- Input: The scalar volume data values $C_v$ and basis function parameters (cell width $C_w$, control point coordinates $C_p$) used for the current sample evaluation at position $p$ [S1].
- Output: The gradient vector $\nabla \hat{H}(p) = [\frac{\partial \hat{H}}{\partial x}, \frac{\partial \hat{H}}{\partial y}, \frac{\partial \hat{H}}{\partial z}]$ [S1].

**Data Structures:**
The method operates on the same buffers used for scalar reconstruction. No additional storage is required for gradient samples. The gradient is computed per sample point during ray traversal [S1].

**Formulas:**
The gradient of the reconstructed scalar field $\hat{H}(p)$ is derived from the quotient rule applied to the basis reconstruction equation [S1]. For the x-component:

$$
\frac{\partial \hat{H}(p)}{\partial x} = \frac{\sum_C \hat{H}_C(p) \sum_C \frac{\partial \hat{H}_C(p)}{\partial x} C_v - \sum_C \hat{H}_C(p) C_v \sum_C \frac{\partial \hat{H}_C(p)}{\partial x}}{(\sum_C \hat{H}_C(p))^2}
$$

The partial derivative of the individual basis function component $\hat{H}_C(p)$ with respect to $x$ is:

$$
\frac{\partial \hat{H}_C(p)}{\partial x} = \hat{h}\left(\frac{|C_{py} - p_y|}{C_w}\right) \hat{h}\left(\frac{|C_{pz} - p_z|}{C_w}\right) \chi(x) \frac{1}{C_w}
$$

Where:
- $\hat{h}(\cdot)$ is the 1D basis function component [S1].
- $C_w$ is the cell width [S1].
- $C_{py}, C_{pz}$ are the y and z coordinates of the control point $C$ [S1].
- $\chi(t)$ is the sign function defined as:
  $$
  \chi(t) = \begin{cases} -1, & \text{if } C_{pt} - p_t \geq 0 \\ 1, & \text{otherwise} \end{cases}
  $$
  In the context of $\frac{\partial \hat{H}}{\partial x}$, the argument $t$ corresponds to the x-dimension relative to the control point, i.e., $C_{px} - p_x$ [S1]. The source text explicitly writes $\chi(x)$ in the derivative formula but defines $\chi(t)$ with a generic argument [S1]. The sign depends on the relative position of the sample point to the control point in the dimension being differentiated [S1].

**Steps:**
1. For each contributing control point $C$, evaluate the 1D basis function $\hat{h}$ for the absolute normalized distances in all three dimensions: $h_x = \hat{h}(|C_{px} - p_x|/C_w)$, $h_y = \hat{h}(|C_{py} - p_y|/C_w)$, $h_z = \hat{h}(|C_{pz} - p_z|/C_w)$ [S1].
2. Compute the basis function value $\hat{H}_C(p) = h_x h_y h_z$ and accumulate $S_H = \sum \hat{H}_C(p)$ and $S_{Hv} = \sum \hat{H}_C(p) C_v$ [S1].
3. Compute the partial derivative $\frac{\partial \hat{H}_C(p)}{\partial x}$ using $h_y$, $h_z$, and $\chi(C_{px} - p_x)$, then accumulate $S_{dHx} = \sum \frac{\partial \hat{H}_C(p)}{\partial x}$ and $S_{dHx\_v} = \sum \frac{\partial \hat{H}_C(p)}{\partial x} C_v$ [S1]. Repeat for y and z dimensions to get $S_{dHy}, S_{dHy\_v}, S_{dHz}, S_{dHz\_v}$ [S1].
4. Compute the final gradient component using the quotient rule: $\frac{\partial \hat{H}}{\partial x} = \frac{S_H S_{dHx\_v} - S_{Hv} S_{dHx}}{S_H^2}$ [S1]. Repeat for y and z [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_analytic_gradient(p, control_points, C_w, basis_func_h):
    sum_H = 0.0
    sum_Hv = 0.0
    sum_dHx = 0.0
    sum_dHx_v = 0.0
    sum_dHy = 0.0
    sum_dHy_v = 0.0
    sum_dHz = 0.0
    sum_dHz_v = 0.0

    for C in control_points:
        dx = C.px - p.x
        dy = C.py - p.y
        dz = C.pz - p.z

        h_x = basis_func_h(abs(dx) / C_w)
        h_y = basis_func_h(abs(dy) / C_w)
        h_z = basis_func_h(abs(dz) / C_w)

        H_C = h_x * h_y * h_z
        sum_H += H_C
        sum_Hv += H_C * C.v

        chi_x = -1 if dx >= 0 else 1
        chi_y = -1 if dy >= 0 else 1
        chi_z = -1 if dz >= 0 else 1

        dH_C_x = h_y * h_z * chi_x * (1.0 / C_w)
        sum_dHx += dH_C_x
        sum_dHx_v += dH_C_x * C.v

        dH_C_y = h_x * h_z * chi_y * (1.0 / C_w)
        sum_dHy += dH_C_y
        sum_dHy_v += dH_C_y * C.v

        dH_C_z = h_x * h_y * chi_z * (1.0 / C_w)
        sum_dHz += dH_C_z
        sum_dHz_v += dH_C_z * C.v

    if sum_H == 0:
        return (0.0, 0.0, 0.0)

    dH_dx = (sum_H * sum_dHx_v - sum_Hv * sum_dHx) / (sum_H * sum_H)
    dH_dy = (sum_H * sum_dHy_v - sum_Hv * sum_dHy) / (sum_H * sum_H)
    dH_dz = (sum_H * sum_dHz_v - sum_Hv * sum_dHz) / (sum_H * sum_H)

    return (dH_dx, dH_dy, dH_dz)
```

**Parameters:**
- $C_w$: Cell width. Typical values depend on the grid resolution and level of detail [S1].
- Basis function $\hat{h}$: Typically a cubic B-spline or similar smooth function [G].

**Edge Cases and Pitfalls:**
- Analytic gradients are not continuously differentiable, leading to shading artifacts at cell and level boundaries [S1].
- Division by zero if $\sum \hat{H}_C(p) = 0$; guard against this [G].
- Precision issues near cell boundaries where the sign function $\chi$ changes [S1].

## Subtopics

### [Analytic Gradient Computation via Box-Spline Derivatives](3199-analytic-gradient-computation-via-box-spline-derivatives.md)
Computes exact gradient vectors for local shading by evaluating analytic derivatives of the underlying box-spline basis function. It sets itself apart from this method by specifically exploiting the symmetry and structure of Face-Centered Cubic (FCC) box-spluses to reuse the 19 samples already fetched for scalar evaluation, whereas finite difference methods require six additional full spline evaluations. Choose it when rendering FCC-structured data to maximize sample reuse.

## Operating characteristics
- **Time Complexity:** $O(1)$ per sample, same as scalar evaluation. No additional ray traversals or sample fetches [S1].
- **Memory Complexity:** $O(1)$ additional memory per sample for gradient components. No extra buffers for gradient samples [S1].
- **Performance:** Much faster than both central difference variants, which require 6 additional basis reconstruction samples and ray traversals [S1].
- **Quality:** Provides superior quality to clamped central differences. May be slightly worse than accurate central differences due to discontinuities in the derivative at boundaries, but the trade-off favors speed [S1].
- **When to use:** When performance is critical and the discontinuities at cell boundaries are acceptable or mitigated by other means. Default in the ExaBricks renderer [S1].
- **When not to use:** When continuous gradients are strictly required and performance is not a constraint [S1].

## Background
Originated in the context of ray tracing structured AMR data using ExaBricks [S1]. The method was introduced to enable efficient local shading with BRDFs without the prohibitive cost of additional samples required by finite difference methods [S1]. It leverages the mathematical structure of the basis reconstruction function to provide analytic derivatives [S1].

## Sources
[S1] Ray Tracing Structured AMR Data Using ExaBricks, https://www.sci.utah.edu/~wald/Publications/2020/exabrick/exabrick.pdf, fetched 2026-10-30
