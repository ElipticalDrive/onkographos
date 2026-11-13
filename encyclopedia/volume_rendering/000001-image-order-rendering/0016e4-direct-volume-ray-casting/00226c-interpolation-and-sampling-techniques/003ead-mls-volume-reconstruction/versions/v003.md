# MLS Volume Reconstruction

Also called: MLS Ray Casting, Moving Least Squares Ray Casting, Moving Least Squares Volume Rendering, Volume MLS Ray Casting

## Overview
MLS Volume Reconstruction is a volume rendering technique that approximates a continuous scalar field and its gradients from discrete samples (regular or irregular) using Moving Least Squares (MLS) approximation [S1, unofficial]. It solves the problem of high-quality visualization of arbitrary volumetric datasets by providing a unified framework for function reconstruction that supports adaptive preintegration [S1, unofficial]. Unlike Radial Basis Functions (RBFs), MLS does not require solving a global system of equations, making it more efficient for large datasets [S1, unofficial]. It sets itself apart from standard interpolation methods by allowing tunable smoothing and interpolation control through basis and weighting functions, and from standard ray casting by integrating adaptive subdivision based on preintegration tables to handle the computational cost of the scalar field evaluation [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Discrete scalar values $f_i$ at sample positions $\mathbf{x}_i$ (regular or irregular grid); transfer function mapping scalar values to color $c(\cdot)$ and opacity $\tau(\cdot)$; ray origin $\mathbf{o}$ and direction $\mathbf{d}$.
*   **Output:** Final pixel color $I$ and accumulated opacity $\alpha$.

**Data Structures:**
*   **Spatial Index:** A hierarchical data structure (e.g., CPU kd-tree) to find neighbor samples $\mathbf{x}_i$ with non-zero weights for a query point $\mathbf{x}$ [S1, unofficial]. For irregular data, the kd-tree is built recursively until node size is smaller than the average radius of the node's samples. A sample is referenced by all leaves intersecting its corresponding ellipsoid (or ball) [S1, unofficial].
*   **Preintegration Table:** A 3D lookup table indexed by $f(d_i)$, $f(d_{i+1})$, and interval length $l = d_{i+1} - d_i$. It stores approximated color $\tilde{C}$ and opacity $\tilde{\alpha}$ for linear scalar field assumptions over the interval [S1, unofficial].

**Parameters:**
*   **Basis Functions $\mathbf{b}(\mathbf{x})$:** Vector of basis functions. Linear basis $\mathbf{b}(\mathbf{x}) = (1, x, y, z)^T$ is recommended [S1, unofficial].
*   **Support Size $h$:** Radius within which neighbors influence reconstruction [S1, unofficial].
*   **Weighting Power $\alpha$:** Controls weight decrease speed. $\alpha=4$ is recommended for $C^3$ continuity [S1, unofficial].
*   **Number of Neighbors $k$:** Typically 8 to 20 [S1, unofficial].
*   **Error Threshold $\varepsilon$:** Termination criterion for adaptive subdivision [S1, unofficial].

**Step 1: Neighbor Query**
For a query point $\mathbf{x}$ along the ray, find all sample indices $i$ such that the weight $w_i(\mathbf{x}) > 0$. This is done using the spatial index [S1, unofficial].

**Step 2: Weight Computation**
Compute weights $w_i(\mathbf{x})$ for each neighbor.
*   For uniform data:
    $$ w_i(\mathbf{x}) = \theta \left( \frac{\|\mathbf{x} - \mathbf{x}_i\|}{h} \right) $$
    where $\theta(u) = (1-u^2)^\alpha$ if $u < 1$, else 0 [S1, unofficial].
*   For irregular/anisotropic data:
    $$ w_i(\mathbf{x}) = \theta \left( \|\mathbf{H}_i(\mathbf{x} - \mathbf{x}_i)\| \right) $$
    where $\mathbf{H}_i$ is a transformation matrix from covariance analysis of one-ring neighborhoods [S1, unofficial].

**Step 3: MLS Coefficient Calculation**
Solve for the local coefficients $\mathbf{c}_{\mathbf{x}}$ that minimize the weighted sum of squared errors:
$$ \mathbf{c}_{\mathbf{x}} = \operatorname{argmin}_{\mathbf{c}} \sum_i w_i(\mathbf{x}) | \mathbf{b}(\mathbf{x}_i)^T \mathbf{c} - f_i |^2 $$
In matrix form, let $\mathbf{B}$ be the $n \times k$ matrix where $B_{ij} = b_j(\mathbf{x}_i)$, $\mathbf{W}(\mathbf{x})$ be the diagonal matrix of weights, and $\mathbf{f}$ be the vector of data values. The solution is:
$$ \mathbf{c}_{\mathbf{x}} = (\mathbf{B}^T \mathbf{W}(\mathbf{x}) \mathbf{B})^{-1} \mathbf{B}^T \mathbf{W}(\mathbf{x}) \mathbf{f} $$
To improve stability and performance:
1.  Shift the basis origin to $\mathbf{x}$: evaluate $\mathbf{b}(\mathbf{x}_i - \mathbf{x})$ [S1, unofficial].
2.  Use Cholesky-like decomposition $\mathbf{B}^T \mathbf{W}(\mathbf{x}) \mathbf{B} = \mathbf{L} \mathbf{D} \mathbf{L}^T$ (where $\mathbf{L}$ is lower triangular with unit diagonal, $\mathbf{D}$ is diagonal) [S1, unofficial].
3.  Compute $\mathbf{c}_{\mathbf{x}} = \mathbf{L}^{-T} \mathbf{D}^{-1} \mathbf{L}^{-1} (\mathbf{B}^T \mathbf{W}(\mathbf{x}) \mathbf{f})$ [S1, unofficial].
4.  Since the basis is evaluated at the origin ($\mathbf{x}$), only the constant coefficient $c_1$ of $\mathbf{c}_{\mathbf{x}}$ is needed for the function value $f(\mathbf{x}) = \mathbf{b}(\mathbf{x})^T \mathbf{c}_{\mathbf{x}}$ [S1, unofficial].

**Step 4: Gradient Computation**
Compute the gradient $\nabla f(\mathbf{x})$ for shading and adaptive subdivision.
*   **Local Approximate Gradient (Recommended):**
    $$ \frac{\partial f(\mathbf{x})}{\partial x_k} \approx \frac{\partial \mathbf{b}(\mathbf{x})}{\partial x_k} \mathbf{c}_{\mathbf{x}} $$
    This is computationally efficient and produces indistinguishable results from the analytic gradient for most models [S1, unofficial].
*   **Analytic Gradient (High Frequency):**
    $$ \frac{\partial f(\mathbf{x})}{\partial x_k} = \frac{\partial \mathbf{b}(\mathbf{x})^T}{\partial x_k} \mathbf{c}_{\mathbf{x}} - \mathbf{b}(\mathbf{x})^T \mathbf{A}(\mathbf{x})^{-1} \left( \frac{\partial \mathbf{A}(\mathbf{x})}{\partial x_k} \mathbf{c}_{\mathbf{x}} + \mathbf{B}^T \frac{\partial \mathbf{W}(\mathbf{x})}{\partial x_k} \mathbf{f} \right) $$
    where $\mathbf{A}(\mathbf{x}) = \mathbf{B}^T \mathbf{W}(\mathbf{x}) \mathbf{B}$ [S1, unofficial].

**Step 5: Adaptive Preintegration**
Integrate along the ray segment $[d_i, d_{i+1}]$ with length $l$.
1.  **Initial Approximation:** Look up $\tilde{C}_1 = \tilde{C}_{i, i+1, l}$ and $\tilde{\alpha}_1 = \tilde{\alpha}_{i, i+1, l}$ from the preintegration table using $f(d_i)$, $f(d_{i+1})$, and $l$ [S1, unofficial].
2.  **Subdivision:** If termination criteria are not met, subdivide the interval into $[d_i, d_{i+1/2}]$ and $[d_{i+1/2}, d_{i+1}]$ with length $l/2$.
    *   Compute $\tilde{C}_{sub1} = \tilde{C}_{i, i+1/2, l/2}$ and $\tilde{\alpha}_{sub1} = \tilde{\alpha}_{i, i+1/2, l/2}$.
    *   Compute $\tilde{C}_{sub2} = \tilde{C}_{i+1/2, i+1, l/2}$ and $\tilde{\alpha}_{sub2} = \tilde{\alpha}_{i+1/2, i+1, l/2}$.
    *   Calculate combined color $\tilde{C}_2 = \tilde{C}_{sub1} + (1 - \tilde{\alpha}_{sub1})\tilde{C}_{sub2}$.
    *   Calculate combined opacity $\tilde{\alpha}_2 = \tilde{\alpha}_{sub1} + (1 - \tilde{\alpha}_{sub1})\tilde{\alpha}_{sub2}$.
    *   **Note:** Subdivide $[d_i, d_{i+1/2}]$ first to determine $\alpha_{i+1/2}$ for the termination check of the second subinterval [S1, unofficial].
3.  **Termination Check:**
    $$ (1 - \alpha_i) \frac{|C_1 - C_2|}{|C_1|} < \varepsilon $$
    where $|C| = \sqrt{C.r^2 + C.g^2 + C.b^2}$ and $\alpha_i$ is the accumulated opacity before the interval [S1, unofficial].
4.  **Update:** If converged, update accumulated color and opacity:
    $$ \alpha_{i+1} = \alpha_i + (1 - \alpha_i)\tilde{\alpha}_1 $$
    $$ C_{i+1} = C_i + (1 - \alpha_i)\tilde{C}_1 $$
    If not converged, recursively apply steps 2-4 to the subintervals [S1, unofficial].

**Pitfalls:**
*   **Boundary Divergence:** MLS reconstruction can diverge during extrapolation at boundaries, causing artifacts [S1, unofficial].
*   **Flat Spot Phenomenon:** Using a constant basis with interpolating MLS results in zero gradient at data points ($\nabla f(\mathbf{x}_i) = 0$). Use a higher-order basis to avoid this [S1, unofficial].
*   **Neighborhood Query Cost:** For irregular datasets, finding neighbors is the most expensive part of function evaluation [S1, unofficial].

## Subtopics

### Fast and Robust MLS Evaluation
A variant focusing on numerical stability and performance during the coefficient calculation step. It shifts the basis origin to the evaluation point $\mathbf{x}$ (i.e., $b(\mathbf{x}_i) = b(\mathbf{x}_i - \mathbf{x})$) to avoid large numbers and improve stability [S1, unofficial]. It exploits the fact that the basis is only evaluated at the origin to compute only the constant coefficient $c_1$ of $\mathbf{c}_X$, avoiding most calculations in the full solution [S1, unofficial]. It relies on a CPU kd-tree built recursively until node size is smaller than the average radius of the node's samples, referencing samples by all leaves intersecting their ellipsoids [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(m \cdot p \cdot (k + \log(n)))$ for irregular datasets, where $m$ is pixels, $p$ is evaluations per ray, $k$ is neighbors, and $n$ is dataset size [S1, unofficial]. A single function evaluation is $O(k + \log(n))$ [S1, unofficial].
*   **Memory Complexity:** Depends on the spatial index (kd-tree) and preintegration table size. The kd-tree stores references to samples in intersecting leaves [S1, unofficial].
*   **Performance:** For $k=8-20$, MLS evaluation flops are usually less than three times that of trilinear interpolation [S1, unofficial]. Theoretical complexity suggests adaptive schemes reduce evaluations significantly, but GPU architecture limitations make it challenging to fully exploit this gain [S1, unofficial].
*   **Strengths:** Handles irregular and regular grids uniformly; provides high-quality shading via accurate gradients; tunable smoothing/interpolation; no global system solve required [S1, unofficial].
*   **Limitations:** Boundary divergence artifacts; flat spot phenomenon with constant bases; neighborhood query cost for irregular data; limited GPU flexibility for adaptive schemes [S1, unofficial].
*   **When to Use:** High-quality visualization of irregular or regular volumetric data; when accurate gradients for shading are needed; when adaptive sampling is beneficial for complex scalar fields [S1, unofficial].
*   **When Not to Use:** Real-time rendering on hardware with limited flexibility for adaptive branching; when boundary artifacts are unacceptable without additional handling; when dataset is extremely large and neighborhood queries become prohibitive [S1, unofficial].

## Background
MLS Volume Reconstruction is based on Moving Least-Squares methods, with convergence theory and error bounds cited from Levin and others [S1, unofficial]. It was introduced to volume rendering by Jang et al. [S1, unofficial]. The method subsumes convolution (equivalent to MLS with a constant basis) and is more general than inverse distance weighting [S1, unofficial]. It has been applied to datasets like the Marschner-Lobb dataset, irregular 'bucky ball', 'torso model', and blunt fin data [S1, unofficial].

## Sources
[S1] Volume MLS Ray Casting, https://miriah.github.io/publications/volume_mls.pdf, fetched 2026-10-29 (unofficial)
