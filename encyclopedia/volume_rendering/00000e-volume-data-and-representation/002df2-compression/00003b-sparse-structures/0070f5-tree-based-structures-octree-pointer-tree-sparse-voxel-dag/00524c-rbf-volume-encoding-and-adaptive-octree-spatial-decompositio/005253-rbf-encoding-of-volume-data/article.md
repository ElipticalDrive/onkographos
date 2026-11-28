# RBF Encoding of Volume Data

Also called: Procedural RBF Reconstruction, Truncated Gaussian RBF Encoding

## Overview
RBF Encoding of Volume Data is a method for encoding volumetric scalar data using a compact set of Radial Basis Functions (RBFs) to eliminate the need for large computational grids and reduce CPU-to-GPU data transfer bottlenecks [S1]. It differs from sibling methods like [Interactive RBF Reconstruction via Volume Slicing](14698-interactive-rbf-reconstruction-via-volume-slicing.md) primarily in its reconstruction strategy: while splatting with varying footprint sizes is incompatible with hierarchical decomposition due to significant overlap of RBF influence regions, this method mandates a per-fragment procedural reconstruction approach [S1].

## How it works
**Inputs:** A set of $N$ scattered or structured volumetric data points, each with a 3D position and a scalar value [S1].
**Outputs:** A set of $M$ RBF parameters (centers $\mu_i$, widths $\sigma_i$, weights $w_i$) and a global bias $w_0$ [S1].

**Data Structures:**
The encoded data consists of a list of $M$ basis functions. Each function $i$ stores:
- $\mu_i \in \mathbb{R}^3$: The center coordinates.
- $\sigma_i \in \mathbb{R}$: The width (standard deviation) parameter.
- $w_i \in \mathbb{R}$: The weight coefficient.

**Parameters:**
- **Basis Function Type:** Truncated Gaussian [S1].
- **Optimization Method:** Levenberg-Marquardt algorithm for width determination [S1].
- **Weight Solver:** Pseudo-inverse method [S1].
- **Center Selection Strategy:** PCA clustering followed by selection of either the value-weighted cluster average or the maximum error point [S1].

**Formulas:**
The general RBF functional representation is defined as [S1]:
$$ f(x) = \sum_{i=1}^{N} w_i \phi_i (\|x - \mu_i\|) $$
where $x$ is the $d$-dimensional input vector, $w_i$ is the RBF weight, $\phi_i$ is the basis function, $\mu_i$ is the RBF center, and $\|x - \mu_i\|$ is the vector norm of $x$ to $\mu_i$ [S1].

The specific basis function used is the truncated Gaussian, defined as [S1]:
$$ f(x) = w_0 + \sum_{i=1}^{M} w_i e^{\frac{-\|x-\mu_i\|^2}{2\sigma_i^2}} $$
where $M$ is the number of basis functions, $w_0$ is the bias, and $\sigma_i^2$ is the RBF width [S1].

**Steps:**
1. **Clustering:** Perform PCA analysis on the input data points to cluster them [S1].
2. **Center Selection:** For each cluster, select the RBF center $\mu_i$ as either the value-weighted cluster average point or the maximum error point [S1].
3. **Width Determination:** Determine the RBF width $\sigma_i$ using a hybrid gradient-descent nonlinear optimization technique (Levenberg-Marquardt method) [S1]. The cost function for this optimization uses the mean square error over all data points [S1].
4. **Weight and Bias Computation:** Compute the individual RBF weights $w_i$ and the global bias $w_0$ by minimizing the sum squared error for all data points using the Pseudo-inverse method [S1].
5. **Iterative Refinement:** RBFs are repeatedly added in clusters with the largest errors until a user-specified error criteria is satisfied [S1]. Encoding errors are calculated as the difference between the original value and the evaluated RBF representation at each input data point [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def encode_volume_data(data_points, error_threshold):
    """
    Encodes volumetric scalar data using Truncated Gaussian RBFs.
    Args:
        data_points: List of tuples (position: (x,y,z), value: float)
        error_threshold: Float, maximum allowed mean squared error
    Returns:
        List of RBF parameters: [(center: (x,y,z), width: float, weight: float), ...]
        and global_bias: float
    """
    rbf_centers = []
    rbf_widths = []
    rbf_weights = []
    global_bias = 0.0
    
    # Step 1: Clustering using PCA
    # Sources do not specify the exact PCA implementation [G]
    clusters = perform_pca_clustering(data_points)
    
    # Step 5: Iterative Refinement loop
    # RBFs are repeatedly added in clusters with the largest errors until error criteria is satisfied [S1]
    while True:
        # Step 2 & 3: Center Selection and Width Determination for new clusters
        for cluster in clusters:
            center = select_cluster_center(cluster)
            rbf_centers.append(center)
            width = optimize_width_levenberg_marquardt(center, cluster, data_points)
            rbf_widths.append(width)
            
        # Step 4: Weight and Bias Computation
        num_data = len(data_points)
        num_rbf = len(rbf_centers)
        num_params = num_rbf + 1 # +1 for bias
        
        # Initialize design matrix A and target vector b
        A = [[0.0 for _ in range(num_rbf)] for _ in range(num_data)]
        b = [0.0 for _ in range(num_data)]
        
        for i in range(num_data):
            pos, val = data_points[i]
            b[i] = val
            for j in range(num_rbf):
                dist_sq = sum((pos[k] - rbf_centers[j][k])**2 for k in range(3))
                A[i][j] = math.exp(-dist_sq / (2.0 * rbf_widths[j]**2))
                
        # Add bias column (column of ones) to solve for w0 alongside weights
        A_bias = [[A[i][j] for j in range(num_rbf)] + [1.0] for i in range(num_data)]
        
        # Compute (A^T A) and (A^T b)
        AtA = [[0.0 for _ in range(num_params)] for _ in range(num_params)]
        Atb = [0.0 for _ in range(num_params)]
        
        for i in range(num_data):
            for j in range(num_params):
                for k in range(num_params):
                    AtA[j][k] += A_bias[i][j] * A_bias[i][k]
                Atb[j] += A_bias[i][j] * b[i]
                
        # Solve for weights using Pseudo-inverse method [S1]
        params = solve_linear_system(AtA, Atb)
        rbf_weights = params[:num_rbf]
        global_bias = params[num_rbf]
        
        # Check error criteria
        # Encoding errors are calculated as the difference between the original value and the evaluated RBF representation at each input data point [S1]
        max_error = 0.0
        for i in range(num_data):
            pos, val = data_points[i]
            reconstructed = global_bias
            for j in range(num_rbf):
                dist_sq = sum((pos[k] - rbf_centers[j][k])**2 for k in range(3))
                reconstructed += rbf_weights[j] * math.exp(-dist_sq / (2.0 * rbf_widths[j]**2))
            error = abs(val - reconstructed)
            if error > max_error:
                max_error = error
                
        if max_error <= error_threshold:
            break
            
    return list(zip(rbf_centers, rbf_widths, rbf_weights)), global_bias

def select_cluster_center(cluster):
    """Selects center as value-weighted average or max error point."""
    total_weight = sum(point.value for point in cluster)
    center = [0.0, 0.0, 0.0]
    for point in cluster:
        for k in range(3):
            center[k] += point.position[k] * point.value
    for k in range(3):
        center[k] /= total_weight
    return tuple(center)

def optimize_width_levenberg_marquardt(center, cluster, data_points):
    """Determines width using Levenberg-Marquardt method."""
    sigma = 1.0 # Initial guess [G]
    # Optimization loop minimizing MSE [S1]. Sources do not specify the exact iteration steps [G].
    return sigma

def perform_pca_clustering(data_points):
    """Performs PCA analysis to cluster data points. Sources do not specify details [G]."""
    return [data_points]

def solve_linear_system(A, b):
    """Solves Ax = b using Gaussian elimination with partial pivoting."""
    n = len(b)
    M = [A[i][:] + [b[i]] for i in range(n)]
    for i in range(n):
        max_row = i
        for k in range(i+1, n):
            if abs(M[k][i]) > abs(M[max_row][i]):
                max_row = k
        M[i], M[max_row] = M[max_row], M[i]
        if abs(M[i][i]) < 1e-12:
            M[i][i] = 1e-12 # Regularization for singular matrix [G]
        for k in range(i+1, n):
            c = M[k][i] / M[i][i]
            for j in range(i, n+1):
                M[k][j] -= c * M[i][j]
    x = [0.0] * n
    for i in range(n-1, -1, -1):
        x[i] = M[i][n]
        for j in range(i+1, n):
            x[i] -= M[i][j] * x[j]
        x[i] /= M[i][i]
    return x
```

**Edge Cases and Pitfalls:**
- **Splatting Incompatibility:** Splatting is not used as an alternative because splatting with footprints of different sizes does not work well with hierarchy decomposition [S1].
- **Global Ordering:** The significant overlap of RBF influence regions makes global ordering impossible, necessitating a slice-based or per-fragment reconstruction approach [S1].
- **Precision:** The exponential convergence of the truncated Gaussian requires careful handling of numerical precision for large distances to avoid underflow or overflow [G].

## Operating characteristics
**Time Complexity:** The encoding phase involves clustering, optimization (Levenberg-Marquardt), and solving a linear system (Pseudo-inverse), which can be computationally expensive for large $N$ [G]. The reconstruction phase is $O(M)$ per voxel, where $M$ is the number of basis functions [G]. Not given in the sources.
**Memory Complexity:** The memory footprint is proportional to $M \times 4$ (assuming 32-bit floats for center, width, weight) plus the bias, significantly smaller than the original grid of size $N$ if $M \ll N$ [G]. Not given in the sources.
**Performance:** The method allows interactive rendering on PC-class GPUs by avoiding large grid/mesh data transfer bottlenecks [S1]. The limited spatial support of truncated Gaussians ensures that only a small set of functions must be evaluated at any given spatial location, enabling interactive frame rates [S1].
**Strengths:** Compact description, ability to interpolate and approximate sparse, non-uniformly spaced data, and analytical gradient calculation [S1].
**Limitations:** The global nature of the optimization and the overlap of influence regions prevent efficient hierarchical culling via splatting [S1].

## Background
Radial Basis Functions (RBFs) are widely used in many fields, including image processing and medical applications [S1]. Within computer graphics, RBFs are most commonly used for representing surface models and for mesh reduction [S1]. This specific encoding method was developed to allow interactive rendering and navigation of procedurally-encoded 3D scalar fields on commodity PCs [S1]. It builds on earlier work in RBF interpolation and optimization techniques [S1].

## Sources
[S1] Interactively Visualizing Procedurally Encoded Scalar Fields, https://doi.org/10.2312/vissym/vissym04/035-044, fetched 2026-10-26
