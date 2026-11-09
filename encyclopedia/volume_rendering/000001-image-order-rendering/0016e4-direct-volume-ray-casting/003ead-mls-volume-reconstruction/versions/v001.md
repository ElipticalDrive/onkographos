# MLS Volume Reconstruction

Also called: MLS Ray Casting, Moving Least Squares Ray Casting, Moving Least Squares Volume Rendering, Volume MLS Ray Casting

## Overview
MLS Volume Reconstruction is a volume rendering technique that reconstructs a smooth scalar field from discrete samples (regular or irregular grids) using Moving Least Squares (MLS) approximation, allowing for adaptive preintegration to handle the computational cost of the scalar field evaluation [S1, unofficial]. It solves the problem of high-quality visualization of arbitrary volumetric datasets by providing a unified mathematical framework for function reconstruction and shading, subsuming convolution as a special case [S1, unofficial]. Unlike Radial Basis Functions (RBFs), MLS does not require solving a global system of equations, making it more efficient for large datasets [S1, unofficial].

## How it works
The method operates on the fundamental loop structure of [Direct volume ray casting](19-direct-volume-ray-casting.md). The core computational engine consists of four sequential stages applied to each ray [S5, S10, S37, unofficial]:

1.  **Ray Generation and Discretization**: The ray segment $[t_{start}, t_{end}]$ intersecting the volume is determined, typically by intersecting the ray with the volume's bounding box [S1, unofficial][S8, S16, unofficial]. The segment is discretized into $N$ samples at equidistant intervals $\Delta t = (t_{end} - t_{start}) / N$ [S5, S10, S17, unofficial].
2.  **Sampling and Interpolation**: At each sample point $\mathbf{p}_i = r(t_i)$, the scalar value $V(\mathbf{p}_i)$ is retrieved from the volume data. This is performed by evaluating the MLS approximation function at $\mathbf{p}_i$ [S1, unofficial].
3.  **Classification (Transfer Function)**: The scalar value $V(\mathbf{p}_i)$ is mapped to optical properties: color $C_i$ and opacity $\alpha_i$ [S2, S2789]. The opacity is often corrected for the step size $\Delta t$ to ensure physical consistency, using the Beer-Lambert law approximation:
    $$ \alpha_i = 1 - e^{-\sigma_i \Delta t} $$
    where $\sigma_i$ is the extinction coefficient derived from the scalar value [S24, S30, unofficial].
4.  **Compositing**: The samples are accumulated from front to back (closest to the eye to furthest) using the front-to-back alpha compositing equation [S1, unofficial][S2, S13, unofficial][S61]:
    $$ C_{out} = C_{in} + (1 - \alpha_{in}) \cdot C_{sample} \cdot \alpha_{sample} $$
    $$ \alpha_{out} = \alpha_{in} + (1 - \alpha_{in}) \cdot \alpha_{sample} $$
    where $C_{in}$ and $\alpha_{in}$ are the accumulated color and opacity from previous samples, and $C_{sample}$ and $\alpha_{sample}$ are the color and opacity of the current sample [S52, S60, unofficial]. The process initializes with $C_{in} = 0$ and $\alpha_{in} = 0$ [S15, S46, unofficial].

### MLS Function Reconstruction
The scalar field $f(\mathbf{x})$ is reconstructed locally at each evaluation point $\mathbf{x}$ by minimizing the weighted sum of squared errors [S1, unofficial]:
$$ g_{\mathbf{x}} = \operatorname{argmin}_g \sum_i w_i(\mathbf{x}) |g(\mathbf{x}_i) - f_i|^2 $$
where $w_i(\mathbf{x})$ is the weight of sample $p_i$ for evaluation point $\mathbf{x}$, and $f_i$ is the data value at $\mathbf{x}_i$ [S1, unofficial]. Assuming $g_{\mathbf{x}}$ is a linear combination of $k$ basis functions $\mathbf{b}(\mathbf{x}) = [b_1(\mathbf{x}), \dots, b_k(\mathbf{x})]^T$ with coefficients $\mathbf{c}_{\mathbf{x}}$, the explicit MLS approximating function is:
$$ f(\mathbf{x}) = \mathbf{b}(\mathbf{x})^T (\mathbf{B}^T \mathbf{W}(\mathbf{x}) \mathbf{B})^{-1} \mathbf{B}^T \mathbf{W}(\mathbf{x}) \mathbf{f} $$
where $\mathbf{B}$ is the matrix of basis functions evaluated at data points, $\mathbf{W}(\mathbf{x})$ is the diagonal matrix of weights, and $\mathbf{f}$ is the vector of data values [S1, unofficial]. The matrix $\mathbf{B}^T \mathbf{W}(\mathbf{x}) \mathbf{B}$ must be invertible, implying at least $k$ weights must be nonzero [S1, unofficial].

#### Basis Functions
The paper advocates using a linear basis $\mathbf{b}(\mathbf{x}) = (1, x, y, z)^T$. Quadratic bases increase computational burden by more than a factor of two with little improvement [S1, unofficial].

#### Weighting Functions
Weights typically decrease with distance from $\mathbf{x}$ [S1, unofficial].
*   **Uniform datasets**: Use a compactly supported polynomial weight:
    $$ w_i(\mathbf{x}) = \theta \left( \frac{\|\mathbf{x} - \mathbf{x}_i\|}{h} \right) $$
    where $h$ is the support size, and $\theta(x)$ is:
    $$ \theta(x) = \begin{cases} (1-x^2)^\alpha & \text{if } x < 1 \\ 0 & \text{otherwise.} \end{cases} $$
    The power factor $\alpha$ controls the speed of decrease. $\alpha=4$ is recommended as a compromise, guaranteeing $C^3$ reconstruction continuity [S1, unofficial].
*   **Irregular datasets**: Use anisotropic weights with ellipsoidal support $H_i$:
    $$ w_i(\mathbf{x}) = \boldsymbol{\theta} \left( \|\mathbf{H}_i(\mathbf{x} - \mathbf{x}_i)\| \right) $$
    $H_i$ is obtained from covariance analysis of one-ring neighborhoods. For non-anisotropic irregular data, balls (scalar radius $h_i$) suffice [S1, unofficial].
*   **Interpolating datasets**: Use a weighting function with a singularity for exact interpolation:
    $$ \theta(x) = \begin{cases} \ln(x)^2 - x^2 + 2x - 1 & 0 \leq x \leq 1 \\ 0 & x > 1 \end{cases} $$
    Note: Using a constant basis with interpolating MLS causes the "flat spot phenomenon" where $\nabla f(\mathbf{x}_i) = 0$. This is avoided by using a higher order basis [S1, unofficial].

#### Gradient Computation
High-quality shading requires accurate gradients. Two methods are established [S1, unofficial]:
1.  **Analytic Gradient**: Most accurate.
    $$ \frac{\partial f(\mathbf{x})}{\partial x_k} = \frac{\partial \mathbf{b}(\mathbf{x})^T}{\partial x_k} \mathbf{c}_{\mathbf{x}} - \mathbf{b}(\mathbf{x})^T \mathbf{A}(\mathbf{x})^{-1} \left( \frac{\partial \mathbf{A}(\mathbf{x})}{\partial x_k} \mathbf{c}_{\mathbf{x}} + \mathbf{B}^T \frac{\partial \mathbf{W}(\mathbf{x})}{\partial x_k} \mathbf{f} \right) $$
    where $\mathbf{A}(\mathbf{x}) = \mathbf{B}^T \mathbf{W}(\mathbf{x}) \mathbf{B}$ [S1, unofficial].
2.  **Local Approximate Gradient**: Computationally efficient alternative.
    $$ \frac{\partial f(\mathbf{x})}{\partial x_k} \approx \frac{\partial \mathbf{b}(\mathbf{x})}{\partial x_k} \mathbf{c}_{\mathbf{x}} $$
    Results are indistinguishable from analytic except for high-frequency models. Recommended for general use [S1, unofficial].

#### Fast and Robust MLS Evaluation
To improve stability and performance [S1, unofficial]:
1.  Shift the origin of the basis to the evaluation point $\mathbf{x}$, i.e., $b(\mathbf{x}_i) = b(\mathbf{x}_i - \mathbf{x})$, to avoid large numbers.
2.  Since the basis is only evaluated at $\mathbf{x}$ (the origin), compute only the constant coefficient $c_1$ of $\mathbf{c}_{\mathbf{x}}$, avoiding most calculation.
3.  Solve for coefficients $\mathbf{c}_{\mathbf{x}}$ using LDL decomposition to avoid square roots:
    $$ \mathbf{c}_{\mathbf{x}} = \mathbf{L}^{-T}\mathbf{D}^{-1}\mathbf{L}^{-1}\left(\mathbf{B}^T\mathbf{W}(\mathbf{x})\mathbf{f}\right) $$
    where $\mathbf{B}^T \mathbf{W}(\mathbf{x}) \mathbf{B} = \mathbf{L}\mathbf{D}\mathbf{L}^T$ [S1, unofficial].
4.  Find all neighbor samples $\mathbf{p}_i$ having strictly positive weights using a spatial data structure (e.g., CPU kd-tree). A sample is referenced by all leaves intersecting its corresponding ellipsoid. The kd-tree is built recursively until node size is smaller than the average radius of the node's samples [S1, unofficial].

### Adaptive Preintegration
The MLS framework incurs expensive scalar field evaluations. Adaptive preintegration modifies interval lengths along rays, focusing computation where the volume integral changes fastest [S1, unofficial].

#### Preintegration Table Lookup
Assuming $f(\mathbf{x}(\lambda))$ is linear on interval $[d_i, d_{i+1}]$, the approximated color $\tilde{C}$ and opacity $\tilde{\alpha}$ are looked up in a 3D table indexed by $f(d_i)$, $f(d_{i+1})$, and length $l = d_{i+1} - d_i$ [S1, unofficial].

#### Recursive Subdivision Algorithm
1.  Start with coarse step size $l$.
2.  Compute $C_1$ and $\tilde{\alpha}$ from preintegration tables for interval $[d_i, d_{i+1}]$.
3.  Subdivide interval into $[d_i, d_{i+1/2}]$ and $[d_{i+1/2}, d_{i+1}]$.
4.  Compute $C_2$ and $\tilde{\alpha}$ for the two subintervals.
5.  Check termination criteria: $(1 - \alpha_i)|C_1 - C_2|/|C_1| < \varepsilon$.
6.  If not converged, recursively subdivide.
7.  Important: Subdivide $[d_i, d_{i+1/2}]$ first to know $\alpha_{i+1/2}$ before subdividing $[d_{i+1/2}, d_{i+1}]$ [S1, unofficial].

#### Formulas
*   **Interval Opacity**: $\alpha_{[d_i, d_{i+1}]} = 1 - \exp\left(-\int_{d_i}^{d_{i+1}} \tau(f(\mathbf{x}(\lambda))) d\lambda\right)$ [S1, unofficial].
*   **Interval Color**: $C_{[d_i, d_{i+1}]} = \int_{d_i}^{d_{i+1}} c(f(\mathbf{x}(\lambda))) \tau(f(\mathbf{x}(\lambda))) e^{\int_{d_i}^{\lambda} \tau(f(\mathbf{x}(\lambda'))) d\lambda'} d\lambda$ [S1, unofficial].
*   **Recursive Updates**:
    $$ \alpha_{i+1} = \alpha_i + (1 - \alpha_i)\alpha_{[d_i, d_{i+1}]} $$
    $$ C_{i+1} = C_i + (1 - \alpha_i)C_{[d_i, d_{i+1}]} $$
    [S1, unofficial].
*   **Initial Color Approximation**: $C_1 = \tilde{C}_{i,i+1,l}$ [S1, unofficial].
*   **Subdivided Color Approximation**: $C_2 = \tilde{C}_{i,i+1/2,l/2} + (1 - \tilde{\alpha}_{i,i+1/2,l/2})\tilde{C}_{i+1/2,i+1,l/2}$ [S1, unofficial].
*   **Subdivided Opacity Approximation**: $\tilde{\alpha}_{i,i+1,l} \approx \tilde{\alpha}_{i,i+1/2,l/2} + (1 - \tilde{\alpha}_{i,i+1/2,l/2})\tilde{\alpha}_{i+1/2,i+1,l/2}$ [S1, unofficial].
*   **Termination Criteria**: $(1 - \alpha_i)|C_1 - C_2|/|C_1| < \varepsilon$, where $|C| = \sqrt{C.r^2 + C.g^2 + C.b^2}$ [S1, unofficial].

#### Parameters
*   $k$: Number of basis functions. Typically varies between 8 and 20 in practice [S1, unofficial].
*   $h$: Support size. Radius within which neighboring samples influence reconstruction [S1, unofficial].
*   $\alpha$: Weighting function power factor. $\alpha=4$ recommended [S1, unofficial].
*   $p$: Number of function evaluations per ray. Depends on complexity of scalar and transfer functions [S1, unofficial].
*   $m$: Number of pixels in output image [S1, unofficial].
*   $n$: Size of the dataset [S1, unofficial].

#### Edge Cases and Pitfalls
*   **Boundary Behavior**: MLS reconstruction exhibits divergence during extrapolation at boundaries, leading to artifacts [S1, unofficial].
*   **Flat Spot Phenomenon**: When using a constant basis with interpolating MLS, the gradient at data points is zero ($\nabla f(\mathbf{x}_i) = 0$). Avoided by using a higher order basis [S1, unofficial].
*   **Irregular Datasets**: The most expensive part of a function evaluation is the neighborhood query [S1, unofficial].
*   **GPU Limitations**: Limited flexibility of current GPU architectures makes it challenging to take full advantage of gains from adaptive schemes that reduce function evaluations [S1, unofficial].

## Subtopics

### Fast and Robust MLS Evaluation
A variant focusing on numerical stability and computational efficiency during the MLS coefficient calculation. It shifts the basis origin to the evaluation point to avoid large numbers, computes only the constant coefficient of the solution vector, and uses LDL decomposition to solve the linear system without square roots. It relies on a CPU kd-tree for neighbor queries, where samples are referenced by all leaves intersecting their ellipsoidal support. This method is chosen when performance and stability of the local reconstruction are critical, particularly for irregular datasets where neighborhood queries dominate cost [S1, unofficial].

## Operating characteristics
*   **Time Complexity**: Theoretical time complexity for irregular data sets is $O(m \cdot p \cdot (k + \log(n)))$, where $m$ is pixels, $p$ is evaluations per ray, $k$ is neighbors, and $n$ is dataset size [S1, unofficial].
*   **Evaluation Cost**: Runtime of a single function evaluation is $O(k + \log(n))$, where $O(\log(n))$ is the traversal complexity of the hierarchical data structure. In practice, dataset size has little impact; cost is dominated by neighborhood size [S1, unofficial].
*   **Floating Point Operations**: For one function evaluation using MLS Linear Basis: $39k + 56$ flops (including free gradient). For constant basis: $15k + 1$ (function) + $29k + 10$ (gradient) [S1, unofficial].
*   **Performance**: MLS performance is a function of $k$. For $k=8-20$, flops for one MLS evaluation are usually less than three times that of trilinear interpolation [S1, unofficial].
*   **Strengths**: High-quality isosurface shading (specularity) combined with volume integration; handles irregular and regular grids uniformly; tunable smoothing/interpolation; no global system solve required (unlike RBFs) [S1, unofficial].
*   **Limitations**: Boundary divergence artifacts; flat spot phenomenon with constant basis/interpolating weights; expensive neighborhood queries for irregular data; limited GPU efficiency for adaptive schemes [S1, unofficial].
*   **When to Use**: High-quality visualization of arbitrary volumetric datasets (regular or irregular) where shading quality and smooth reconstruction are prioritized over raw speed.
*   **When Not to Use**: When strict interpolation at data points is required with a constant basis (due to flat spots); when GPU adaptive subdivision gains are critical but hardware limits prevent efficient implementation; when boundary artifacts are unacceptable without additional clamping or hull computation [S1, unofficial].

## Background
MLS Volume Reconstruction is based on Moving Least-Squares methods, with convergence theory and error bounds cited from [48, 22] and multivaried nonparametric regression from [40] [S1, unofficial]. It was proposed as a unified framework for generating high-quality visualizations of arbitrary volumetric datasets, subsuming convolution as a special case (convolution is equivalent to MLS with a constant basis function) [S1, unofficial]. It was applied to datasets such as the 'bucky ball' and 'torso model' (irregular), and the Marschner-Lobb dataset (regular 41x41x41 grid), compared against analytic functions and B-spline filters [S1, unofficial]. The method was used for the irregular blunt fin data set with adaptive preintegration [S1, unofficial]. Manuscript received 31 March 2008; accepted 1 August 2008; posted online 19 October 2008 [S1, unofficial].

## Sources
[S1] Volume MLS Ray Casting, https://miriah.github.io/publications/volume_mls.pdf, fetched 2026-10-29 (unofficial)
