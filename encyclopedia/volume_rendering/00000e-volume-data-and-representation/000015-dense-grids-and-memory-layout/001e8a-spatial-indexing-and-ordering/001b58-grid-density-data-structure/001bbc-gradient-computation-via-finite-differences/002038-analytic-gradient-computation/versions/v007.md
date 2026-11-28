# Analytic Gradient Computation

Also called: Analytic Basis Gradients, Analytic Derivatives of Basis Reconstruction

## Overview
Analytic Gradient Computation computes the 3D gradient vector for local shading (BRDF) by deriving exact derivatives of the basis reconstruction function, avoiding the additional memory accesses and ray traversals required by finite difference methods [S1]. It is distinguished from its parent method, [Gradient Computation via Finite Differences](1077-gradient-computation-via-finite-differences.md), by calculating gradients from existing data values rather than sampling offset positions, and from [Analytic Gradient Computation via Box-Spline Derivatives](3199-analytic-gradient-computation-via-box-spline-derivatives.md) by its general applicability to standard basis reconstruction without relying on specific FCC lattice symmetries [S1].

## How it works
**Inputs and Outputs:**
- Input: The scalar volume data values $C_v$, the basis reconstruction function $\hat{H}_C(p)$, the cell width $C_w$, and the sample position $p$ [S1].
- Output: The gradient vector $\nabla B(p) = [\frac{\partial B}{\partial x}, \frac{\partial B}{\partial y}, \frac{\partial B}{\partial z}]$ [S1].

**Data Structures:**
- $C_v$: Scalar value associated with cell $C$ [S1].
- $C_{py}$, $C_{pz}$: Coordinates of the cell center relative to the sample point in the y and z dimensions [S1].
- $C_w$: Width of the cell [S1].
- $\hat{h}(\cdot)$: The 1D basis function component [S1].

**Formulas:**
The gradient of the reconstructed signal $B(p)$ is computed using the quotient rule on the basis reconstruction sum [S1]:

$$
\frac{\partial B(p)}{\partial x} = \frac{\sum_C \hat{H}_C(p) \sum_C \frac{\partial \hat{H}_C(p)}{\partial x} C_v - \sum_C \hat{H}_C(p) C_v \sum_C \frac{\partial \hat{H}_C(p)}{\partial x}}{(\sum_C \hat{H}_C(p))^2}
$$

The partial derivative of the basis function component $\hat{H}_C(p)$ with respect to $x$ is [S1]:

$$
\frac{\partial \hat{H}(p)}{\partial x} = \hat{h}\left(\frac{|C_{py} - p_y|}{C_w}\right) \hat{h}\left(\frac{|C_{pz} - p_z|}{C_w}\right) \chi(x) \frac{1}{C_w}
$$

The sign function $\chi(t)$ is defined as [S1]:

$$
\chi(t) = \begin{cases} -1, & \text{if } C_{p_t} - p_t \geq 0 \\ 1, & \text{otherwise} \end{cases}
$$

**Steps:**
1. For the current sample position $p$, identify all contributing cells $C$ within the basis support [S1].
2. For each cell $C$, compute the basis weight $\hat{H}_C(p)$ and retrieve the data value $C_v$ [S1].
3. Compute the partial derivative of the basis function $\frac{\partial \hat{H}_C(p)}{\partial x}$ by reusing the basis function evaluations from the y and z dimensions and applying the sign function $\chi(x)$ scaled by $1/C_w$ [S1].
4. Accumulate the sums required for the quotient rule: $S_H = \sum_C \hat{H}_C(p)$, $S_{HV} = \sum_C \hat{H}_C(p) C_v$, $S_{dH} = \sum_C \frac{\partial \hat{H}_C(p)}{\partial x}$, and $S_{dHV} = \sum_C \frac{\partial \hat{H}_C(p)}{\partial x} C_v$ [S1].
5. Compute the gradient component $\frac{\partial B(p)}{\partial x}$ using the quotient rule formula [S1].
6. Repeat steps 3–5 for the $y$ and $z$ dimensions to obtain the full gradient vector [S1].

**Parameters:**
- $C_w$: Cell width, determined by the spatial indexing and level structure [S1].

**Edge Cases and Pitfalls:**
- Analytic gradients are not continuously differentiable, leading to shading artifacts at cell and level boundaries [S1].
- The method assumes the basis reconstruction function is differentiable; if the underlying basis function has discontinuous derivatives, the resulting gradient will also be discontinuous [S1][S3, unofficial].

## Subtopics

### [Analytic Gradient Computation via Box-Spline Derivatives](3199-analytic-gradient-computation-via-box-spline-derivatives.md)
This method computes the exact gradient vector for volume rendering shading by deriving analytic derivatives of the underlying box-spline basis function, avoiding the approximation errors and additional sampling costs of finite differences. It is distinguished from its parent method, [Analytic Gradient Computation](2167-analytic-gradient-computation.md), by specifically leveraging the symmetry and support properties of Face-Centered Cubic (FCC) box-splines to reuse existing texture fetches. Unlike [Analytic Gradient Computation for Triangle Distributions](5829-analytic-gradient-computation-for-triangle-distributions.md), which handles triangular supports, this method operates on the hexagonal/triangular lattice structure of FCC grids. Choose this variant when rendering on FCC grids to maximize fetch reuse.

### [Analytic Gradient Computation for Triangle Distributions](5829-analytic-gradient-computation-for-triangle-distributions.md)
This method computes the exact Fourier Transform $\hat{u}_\mathcal{T}$ and its spatial gradients for a single triangle $\mathcal{T}$ defined by vertices $(\mathbf{v}_1, \mathbf{v}_2, \mathbf{v}_3)$. It provides analytic derivatives of the basis reconstruction function, avoiding additional samples or ray traversals [G]. It differs from sibling methods like [Analytic Gradient Computation via Box-Spline Derivatives](3199-analytic-gradient-computation-via-box-spline-derivatives.md) by operating directly on mesh geometry rather than grid-based basis functions. Choose this variant when shading triangular primitives or unstructured meshes.

### Isosurface Rendering with Analytic Gradient
A visualization mode that performs lighting calculations based on the data gradient, which is analytically evaluated in parallel to the data value [S2]. The gradient $\nabla f(x)$ is calculated as [S2]:
$$
\nabla f (x) = - \sum_{i=1}^{M} w_i \frac{x - \mu_i}{2\sigma_i} e^{-\frac{\|x-\mu_i\|^2}{2\sigma_i^2}}
$$
where $w_i$, $\mu_i$, and $\sigma_i$ are the weight, center, and width of the $i$-th Gaussian radial basis function, and $M$ is the number of RBF functions [S2].

### Congruent Gradient of Reconstructed Data
The volume gradient calculated as the gradient of the reconstructed continuous data field [S3, unofficial]. It is calculated by applying the reconstruction filter in two directions and the derivative of the reconstruction filter in the third direction [S3, unofficial].
1. Apply the reconstruction filter in two dimensions (e.g., y and z) [S3, unofficial].
2. Use the derivative of the reconstruction filter to calculate the gradient in the third dimension (e.g., x) [S3, unofficial].
3. For linear reconstruction, the x-gradient is the difference of two bi-linearly interpolated planes: $G_x = L_2(a_2,b_2,c_2,d_2,y,z) - L_2(a_1,b_1,c_1,d_1,y,z)$ [S3, unofficial].
If a reconstruction filter with a continuous derivative is not used, the congruent volume gradient will be discontinuous, leading to visible artefacts [S3, unofficial]. For linear interpolation, the gradient is piece-wise constant in the axis direction, making it discontinuous [S3, unofficial].

## Operating characteristics
- **Time Complexity:** O(1) per sample relative to additional memory accesses, as it uses only existing data values loaded for the original sample evaluation [S1].
- **Memory Complexity:** No additional memory buffers required for gradient storage if computed on-the-fly [S1].
- **Measured Performance:** Much faster than both central difference variants (clamped and accurate) [S1].
- **Strengths:** Superior quality to clamped central differences; avoids additional ray traversals and memory accesses [S1].
- **Limitations:** Not continuously differentiable, leading to shading artifacts at cell and level boundaries [S1].
- **When to use:** When performance is critical and the discontinuities at cell boundaries are acceptable or mitigated by other means [S1].
- **When not to use:** When continuous gradients are required for smooth shading, in which case central differences or higher-order reconstruction filters with continuous derivatives should be used [S1][S3, unofficial].

## Background
Originated in the context of ray tracing structured AMR data using ExaBricks, where local shading with BRDFs required efficient gradient computation [S1]. The method was developed to address the high cost of central differencing, which requires six additional basis reconstruction samples and ray traversals [S1]. It is used in volume rendering renderers that employ basis reconstruction methods for scalar field visualization [S1].

## Sources
[S1] PDF Ray Tracing Structured AMR Data Using ExaBricks, https://www.sci.utah.edu/~wald/Publications/2020/exabrick/exabrick.pdf, fetched 2026-10-30
[S2] Interactively Visualizing Procedurally Encoded Scalar Fields, https://doi.org/10.2312/vissym/vissym04/035-044, fetched 2026-10-26
[S3] Iso-Surface Volume Rendering : speed and accuracy for medical applications, https://doi.org/10.3990/1.9789036513982, fetched 2026-10-26 (unofficial)
