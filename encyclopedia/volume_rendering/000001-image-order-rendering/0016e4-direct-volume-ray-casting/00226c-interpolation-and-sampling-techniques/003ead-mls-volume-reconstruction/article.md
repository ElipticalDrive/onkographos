# MLS Volume Reconstruction

Also called: MLS Ray Casting, Moving Least Squares Ray Casting, Moving Least Squares Volume Rendering, Volume MLS Ray Casting

## Overview
MLS Volume Reconstruction is a volume rendering technique that reconstructs a smooth scalar field from discrete samples (regular or irregular grids) using Moving Least Squares (MLS) approximation, allowing for adaptive preintegration to handle the computational cost of the scalar field evaluation [S1, unofficial]. Unlike Radial Basis Functions (RBFs), which require solving a global system of equations, MLS solves a local weighted least squares problem at each query point, making it more efficient for large datasets [S1, unofficial]. It provides a unifying framework that subsumes convolution (equivalent to MLS with a constant basis) and allows for tunable smoothing and interpolation control [S1, unofficial].

## How it works
**Inputs:**
- A set of $n$ data points $\mathbf{x}_i$ with associated scalar values $f_i$ (and optionally gradients).
- A ray origin $\mathbf{o}$ and direction $\mathbf{d}$.
- Transfer function parameters (color $c$, opacity $\tau$).
- MLS parameters: basis type, support size $h$, power factor $\alpha$, number of neighbors $k$.

**Outputs:**
- Final pixel color $C$ and opacity $\alpha$.

**Data Structures:**
- **Spatial Index:** A hierarchical data structure (e.g., CPU kd-tree) to efficiently query neighbors. For irregular datasets, samples are referenced by all leaves intersecting their corresponding ellipsoid (or ball) support [S1, unofficial].
- **Preintegration Table:** A 3D lookup table indexed by $f(d_i)$, $f(d_{i+1})$, and interval length $l = d_{i+1} - d_i$, storing approximated color $\tilde{C}$ and opacity $\tilde{\alpha}$ [S1, unofficial].

**Step 1: Neighbor Query**
For a query point $\mathbf{x}$ on the ray, find all sample points $\mathbf{x}_i$ such that the weight $w_i(\mathbf{x}) > 0$. This is done by querying the spatial index for points whose support regions (ellipsoids or balls) contain $\mathbf{x}$ [S1, unofficial].

**Step 2: MLS Coefficient Calculation**
Construct the local approximation by solving the weighted least squares problem. To improve stability and performance:
1. Shift the basis origin to the evaluation point $\mathbf{x}$, i.e., evaluate basis functions at $\mathbf{x}_i - \mathbf{x}$ [S1, unofficial].
2. Compute the weighted covariance matrix $\mathbf{A}(\mathbf{x}) = \mathbf{B}^T \mathbf{W}(\mathbf{x}) \mathbf{B}$, where $\mathbf{B}$ contains basis evaluations and $\mathbf{W}(\mathbf{x})$ is a diagonal matrix of weights [S1, unofficial].
3. Perform an $\mathbf{LDL}^T$ decomposition of $\mathbf{A}(\mathbf{x})$ to avoid square roots [S1, unofficial].
4. Solve for coefficients $\mathbf{c}_{\mathbf{x}}$:
   $$ \mathbf{c}_{\mathbf{x}} = \mathbf{L}^{-T}\mathbf{D}^{-1}\mathbf{L}^{-1}\left(\mathbf{B}^T\mathbf{W}(\mathbf{x})\mathbf{f}\right) $$
   Since the basis is evaluated at the origin ($\mathbf{x}$), only the constant coefficient $c_1$ of $\mathbf{c}_{\mathbf{x}}$ is strictly needed for the function value $f(\mathbf{x}) = \mathbf{b}(\mathbf{x})^T \mathbf{c}_{\mathbf{x}}$ if $\mathbf{b}(\mathbf{x}) = [1, 0, 0, 0]^T$ for a linear basis shifted to origin [S1, unofficial].

**Step 3: Gradient Computation**
Compute the gradient $\nabla f(\mathbf{x})$ for shading and adaptive sampling. The local approximate gradient is recommended for efficiency:
$$ \frac{\partial f(\mathbf{x})}{\partial x_k} \approx \frac{\partial \mathbf{b}(\mathbf{x})}{\partial x_k} \mathbf{c}_{\mathbf{x}} $$
This is computationally cheaper than the analytic gradient and produces indistinguishable results for most models [S1, unofficial].

**Step 4: Adaptive Preintegration**
Integrate along the ray using adaptive subdivision to handle extrema and varying opacity:
1. Start with a coarse interval $[d_i, d_{i+1}]$ of length $l$.
2. Look up $\tilde{C}_1$ and $\tilde{\alpha}_1$ from the preintegration table for the full interval [S1, unofficial].
3. Subdivide the interval into $[d_i, d_{i+1/2}]$ and $[d_{i+1/2}, d_{i+1}]$ (length $l/2$).
4. Look up $\tilde{C}_{sub}$ and $\tilde{\alpha}_{sub}$ for the subintervals.
5. Compute the subdivided color $C_2$ and opacity $\tilde{\alpha}_{sub\_total}$:
   $$ C_2 = \tilde{C}_{i,i+1/2,l/2} + (1 - \tilde{\alpha}_{i,i+1/2,l/2})\tilde{C}_{i+1/2,i+1,l/2} $$
   $$ \tilde{\alpha}_{sub\_total} = \tilde{\alpha}_{i,i+1/2,l/2} + (1 - \tilde{\alpha}_{i,i+1/2,l/2})\tilde{\alpha}_{i+1/2,i+1,l/2} $$
   *Note: Subdivide the first half first to determine $\alpha_{i+1/2}$ for the termination check of the second half* [S1, unofficial].
6. Check termination criteria:
   $$ (1 - \alpha_i)|C_1 - C_2|/|C_1| < \varepsilon $$
   where $|C| = \sqrt{C.r^2 + C.g^2 + C.b^2}$ [S1, unofficial].
7. If not converged, recursively subdivide. If converged, update the accumulated color and opacity:
   $$ \alpha_{i+1} = \alpha_i + (1 - \alpha_i)\tilde{\alpha}_{sub\_total} $$
   $$ C_{i+1} = C_i + (1 - \alpha_i)C_2 $$

**Parameters:**
- **Basis:** Linear $\mathbf{b}(\mathbf{x}) = (1, x, y, z)^T$ is recommended. Quadratic bases increase cost by >2x with little visual gain [S1, unofficial].
- **Weighting Power $\alpha$:** $\alpha=4$ is recommended, guaranteeing $C^3$ continuity [S1, unofficial].
- **Support Size $h$:** Radius within which neighbors influence reconstruction. For irregular data, anisotropic ellipsoidal supports $H_i$ derived from covariance analysis of one-ring neighborhoods are used [S1, unofficial].
- **Neighbors $k$:** Typically 8 to 20 [S1, unofficial].

**Pitfalls:**
- **Boundary Divergence:** MLS reconstruction can diverge during extrapolation at boundaries, causing artifacts [S1, unofficial].
- **Flat Spot:** Using a constant basis with interpolating MLS results in zero gradient at data points ($\nabla f(\mathbf{x}_i) = 0$). Use a higher-order basis to avoid this [S1, unofficial].
- **Neighborhood Query Cost:** For irregular datasets, finding neighbors is the most expensive part of the evaluation [S1, unofficial].

## Subtopics

### Fast and Robust MLS Evaluation
A variant focusing on numerical stability and speed. It shifts the basis origin to the evaluation point to avoid large numbers and improves stability. It exploits the fact that the basis is only evaluated at the origin to compute only the constant coefficient of the solution vector. It uses a CPU kd-tree built recursively until node size is smaller than the average radius of samples, referencing samples by all intersecting leaves [S1, unofficial].

## Operating characteristics
- **Time Complexity:** $O(m \cdot p \cdot (k + \log(n)))$ for irregular datasets, where $m$ is pixels, $p$ is evaluations per ray, $k$ is neighbors, and $n$ is dataset size [S1, unofficial].
- **FLOPs:** One MLS evaluation with a linear basis requires $39k + 56$ FLOPs (including free gradient). This is usually less than three times that of trilinear interpolation [S1, unofficial].
- **Memory:** Depends on the spatial index (kd-tree) and preintegration table size.
- **Strengths:** Handles irregular and regular grids uniformly; high-quality shading via accurate gradients; adaptive preintegration reduces evaluations in smooth regions [S1, unofficial].
- **Limitations:** Boundary artifacts; expensive neighborhood queries for irregular data; limited GPU flexibility for adaptive schemes [S1, unofficial].
- **When to use:** High-quality visualization of irregular or scattered volumetric data where smooth gradients are critical [S1, unofficial].

## Background
MLS Volume Reconstruction is based on Moving Least-Squares methods, with convergence theory and error bounds cited from Levin and others [S1, unofficial]. It was introduced to volume rendering to provide a unified framework for arbitrary volumetric datasets, combining well-understood mathematical foundations with tunable smoothing [S1, unofficial]. It subsumes convolution and inverse distance weighting as special cases [S1, unofficial].

## Sources
[S1] Volume MLS Ray Casting, https://miriah.github.io/publications/volume_mls.pdf, fetched 2026-10-29 (unofficial)
