# Diffusion Tensor Visualization using Random Field Correlation

Also called: Adaptive filtering of noise fields for DTI, Random field correlation volume rendering

## Overview
This method visualizes diffusion tensor fields by generating a scalar volume from an anisotropically correlated random noise field, preserving high-frequency components in transversal directions while smearing noise along the direction of greatest diffusivity [S1, unofficial]. Unlike glyphs, streamlines, or line integral convolution, it is not sensitive to occlusion, does not depend on starting points, and provides a continuous scalar field representation that handles non-linear diffusion better [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** A 3D diffusion tensor field $D$. Each voxel contains a $3 \times 3$ symmetric positive-definite tensor.
*   **Output:** A scalar volume $N_{out}$ and a color volume, suitable for standard volume rendering.

**Data Structures**
*   **Diffusion Tensor Field:** Stored as a 3D grid. Each voxel holds a $3 \times 3$ matrix.
*   **Noise Field:** A 3D grid of floating-point values.
*   **Spectral Decomposition:** For each voxel, the tensor $D$ is decomposed into eigenvalues $\lambda_1 \ge \lambda_2 \ge \lambda_3 \ge 0$ and corresponding orthonormal eigenvectors $\hat{\mathbf{e}}_1, \hat{\mathbf{e}}_2, \hat{\mathbf{e}}_3$. These form the matrix $\mathbf{E} = [\hat{\mathbf{e}}_1, \hat{\mathbf{e}}_2, \hat{\mathbf{e}}_3]$ and diagonal matrix $\mathbf{\Sigma} = \text{diag}(\lambda_1, \lambda_2, \lambda_3)$ such that $D = \mathbf{E}\mathbf{\Sigma}\mathbf{E}^T$ [S1, unofficial].

**Parameters**
*   **Amplification factor ($\alpha$):** Not given in the sources.
*   **Number of iterations ($J$):** 20 [S1, unofficial].
*   **Filter Directions:** The six high-pass filter directions correspond to the vertices of an icosahedron [S1, unofficial].

**Algorithm Steps**

1.  **Spectral Decomposition:** For every voxel in the input tensor field, compute the eigenvalues $\lambda_i$ and eigenvectors $\hat{\mathbf{e}}_i$ of $D$ [S1, unofficial].

2.  **Generate Initial Noise:** Create three independent zero-mean Gaussian noise processes. Let $\mathbf{x}$ be a vector containing realizations of these three processes at each voxel location [S1, unofficial].

3.  **Compute Correlated Noise Magnitude:** Calculate the initial noise field magnitude $N$ using the formula:
    $$N = \|\mathbf{y}\| = \|\mathbf{E}\mathbf{\Sigma}\mathbf{x}\|$$
    where $\mathbf{E}$ is the matrix of eigenvectors, $\mathbf{\Sigma}$ is the diagonal matrix of eigenvalues, and $\mathbf{x}$ is the vector of independent Gaussian noise [S1, unofficial]. This creates a spatially independent but not identically distributed noise field with greater power in areas of stronger diffusivity [S1, unofficial].

4.  **Decompose Noise Field:** Decompose the noise field $N$ into a low-pass version $N_{lp}$ and six high-pass components $N_k$ using spherical separable filters. The directions of these six high-pass filters are defined by the vertices of an icosahedron [S1, unofficial].

5.  **Iterative Adaptive Filtering:** Perform $J$ iterations of filtering. In each iteration $j$ (from 1 to $J$):
    a.  **Compute Control Tensor:** For each voxel, compute the control tensor $\mathbf{C}$:
        $$\mathbf{C} = \sum_{n=2}^3 \frac{\lambda_3}{\lambda_n} \hat{\mathbf{e}}_n \hat{\mathbf{e}}_n^T$$
        This tensor is intentionally degenerated to rank 2 to allow full smearing in the major diffusion direction given by $\lambda_1$ [S1, unofficial].
    b.  **Compute Linearity Coefficient:** Calculate the linearity coefficient $c_l$:
        $$c_l = \frac{\lambda_1 - \lambda_2}{\lambda_1}$$
        This accounts for the degree of linearity of the diffusion tensor [S1, unofficial].
    c.  **Compute Mapping Function:** Calculate the weighting factor $h$:
        $$h(|\mathbf{D}|) = h(c_l) = (\alpha c_l + 1)^{\frac{1}{2}}$$
        where $\alpha$ is an amplification factor [S1, unofficial].
    d.  **Filter Output:** Compute the filtered output $F$ for the current iteration using the input noise field from the previous iteration (or $N$ for the first iteration):
        $$F = N_{lp} + h(|\mathbf{D}|) \sum_k \langle \mathbf{C}, \mathbf{M}_k \rangle N_k$$
        Here, $\langle \mathbf{C}, \mathbf{M}_k \rangle$ is the inner product of the control tensor $\mathbf{C}$ and the dual basis $\mathbf{M}_k$ associated with filter direction $k$, and $N_k$ are the high-pass components of the input noise field [S1, unofficial].
    e.  **Update Input:** Set the input noise field for the next iteration to $F$. The goal is to make the noise structure more continuous and sharper [S1, unofficial].

6.  **Final Visualization Data:** After $J$ iterations, the output is $N_{out} = F$ [S1, unofficial].

7.  **Opacity and Color Computation:**
    a.  **Opacity:** Set opacity to $N_{out} \cdot c_l$. This shows the structure only in areas containing fibers [S1, unofficial].
    b.  **Color:** Compute the color vector $(r, g, b)^t$ as:
        $$(r \ g \ b)^t = N_{out} \frac{(1 \ 1 \ 1)^t + \hat{\mathbf{e}}_1}{2}$$
        where $\hat{\mathbf{e}}_1$ is the eigenvector corresponding to the largest eigenvalue $\lambda_1$. This multiplies the computed color by $N_{out}$ to darken the edges of the structures [S1, unofficial].

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def compute_eigen_decomposition(tensor_field):
    """
    Computes eigenvalues and eigenvectors for a field of 3x3 symmetric tensors.
    Returns:
        eigenvalues: (H, W, D, 3) array of lambda_1, lambda_2, lambda_3
        eigenvectors: (H, W, D, 3, 3) array where eigenvectors[:, :, :, :, i] is e_i
    """
    # Implementation depends on the specific tensor field storage.
    # Assumes tensor_field is (H, W, D, 3, 3)
    eigenvalues = np.zeros((H, W, D, 3))
    eigenvectors = np.zeros((H, W, D, 3, 3))
    
    for h in range(H):
        for w in range(W):
            for d in range(D):
                T = tensor_field[h, w, d]
                # T is 3x3 symmetric
                vals, vecs = np.linalg.eigh(T)
                # eigh returns sorted eigenvalues, but we need lambda_1 >= lambda_2 >= lambda_3
                # Sort in descending order
                idx = np.argsort(vals)[::-1]
                eigenvalues[h, w, d] = vals[idx]
                eigenvectors[h, w, d] = vecs[:, idx]
    return eigenvalues, eigenvectors

def generate_gaussian_noise(shape):
    """Generates three independent zero-mean Gaussian noise processes."""
    return np.random.normal(0, 1, (shape[0], shape[1], shape[2], 3))

def decompose_noise_field(noise_field, filter_directions):
    """
    Decomposes noise field into low-pass and high-pass components.
    noise_field: (H, W, D)
    filter_directions: (6, 3) array of icosahedron vertices
    Returns:
        N_lp: (H, W, D)
        N_k: (H, W, D, 6)
    """
    # Implementation of spherical separable filters is not detailed in sources.
    # This is a placeholder for the filtering operation.
    # N_lp is the low-pass version of noise_field.
    # N_k are the high-pass components along the 6 filter directions.
    N_lp = apply_low_pass_filter(noise_field)
    N_k = np.zeros((H, W, D, 6))
    for k in range(6):
        N_k[:, :, :, k] = apply_high_pass_filter(noise_field, filter_directions[k])
    return N_lp, N_k

def compute_control_tensor(eigenvalues, eigenvectors):
    """
    Computes the control tensor C for each voxel.
    eigenvalues: (H, W, D, 3)
    eigenvectors: (H, W, D, 3, 3)
    Returns:
        C: (H, W, D, 3, 3)
    """
    lambda_1 = eigenvalues[:, :, :, 0]
    lambda_2 = eigenvalues[:, :, :, 1]
    lambda_3 = eigenvalues[:, :, :, 2]
    
    e_2 = eigenvectors[:, :, :, :, 1] # Shape (H, W, D, 3)
    e_3 = eigenvectors[:, :, :, :, 2] # Shape (H, W, D, 3)
    
    # C = sum_{n=2}^3 (lambda_3 / lambda_n) * e_n * e_n^T
    # Note: lambda_n here refers to lambda_2 and lambda_3 for n=2,3
    # So for n=2: term is (lambda_3 / lambda_2) * e_2 * e_2^T
    # For n=3: term is (lambda_3 / lambda_3) * e_3 * e_3^T = e_3 * e_3^T
    
    # Compute outer products
    # e_2_outer: (H, W, D, 3, 3)
    e_2_outer = np.einsum('...i,...j->...ij', e_2, e_2)
    e_3_outer = np.einsum('...i,...j->...ij', e_3, e_3)
    
    # Weights
    w_2 = lambda_3 / lambda_2
    w_3 = lambda_3 / lambda_3 # = 1
    
    C = w_2[:, :, :, np.newaxis, np.newaxis] * e_2_outer + w_3[:, :, :, np.newaxis, np.newaxis] * e_3_outer
    return C

def compute_dual_basis_inner_product(C, M_k):
    """
    Computes <C, M_k> for each voxel and each k.
    C: (H, W, D, 3, 3)
    M_k: (6, 3, 3) dual basis tensors
    Returns:
        weights: (H, W, D, 6)
    """
    # Inner product of two matrices A, B is trace(A^T B) or sum(A*B) for symmetric
    # M_k is precomputed for the 6 directions
    weights = np.zeros((H, W, D, 6))
    for k in range(6):
        Mk = M_k[k] # (3, 3)
        # Element-wise multiplication and sum over last two dimensions
        weights[:, :, :, k] = np.sum(C * Mk[np.newaxis, np.newaxis, np.newaxis, :, :], axis=(3, 4))
    return weights

def adaptive_filter_iteration(N_lp, N_k, eigenvalues, eigenvectors, alpha, M_k):
    """
    Performs one iteration of adaptive filtering.
    N_lp: (H, W, D)
    N_k: (H, W, D, 6)
    eigenvalues: (H, W, D, 3)
    eigenvectors: (H, W, D, 3, 3)
    alpha: scalar
    M_k: (6, 3, 3)
    Returns:
        F: (H, W, D)
    """
    C = compute_control_tensor(eigenvalues, eigenvectors)
    
    lambda_1 = eigenvalues[:, :, :, 0]
    lambda_2 = eigenvalues[:, :, :, 1]
    
    c_l = (lambda_1 - lambda_2) / lambda_1
    
    h = np.sqrt(alpha * c_l + 1)
    
    weights = compute_dual_basis_inner_product(C, M_k)
    
    # F = N_lp + h * sum_k (weights_k * N_k_k)
    # weights: (H, W, D, 6), N_k: (H, W, D, 6)
    weighted_N_k = weights * N_k
    sum_k = np.sum(weighted_N_k, axis=3)
    
    F = N_lp + h * sum_k
    return F

def visualize(N_out, eigenvalues, eigenvectors):
    """
    Computes opacity and color for volume rendering.
    N_out: (H, W, D)
    eigenvalues: (H, W, D, 3)
    eigenvectors: (H, W, D, 3, 3)
    Returns:
        opacity: (H, W, D)
        color: (H, W, D, 3)
    """
    lambda_1 = eigenvalues[:, :, :, 0]
    lambda_2 = eigenvalues[:, :, :, 1]
    e_1 = eigenvectors[:, :, :, :, 0] # (H, W, D, 3)
    
    c_l = (lambda_1 - lambda_2) / lambda_1
    opacity = N_out * c_l
    
    # Color = N_out * ( (1,1,1) + e_1 ) / 2
    ones = np.ones((H, W, D, 3))
    color = N_out[:, :, :, np.newaxis] * (ones + e_1) / 2.0
    
    return opacity, color

# Main execution
# 1. Load tensor field
tensor_field = load_tensor_field() # Shape (H, W, D, 3, 3)

# 2. Precompute eigen decomposition
eigenvalues, eigenvectors = compute_eigen_decomposition(tensor_field)

# 3. Generate initial noise
x = generate_gaussian_noise((H, W, D)) # (H, W, D, 3)

# 4. Compute initial noise magnitude N
# N = || E Sigma x ||
# y = E Sigma x
# Sigma x: (H, W, D, 3) * (H, W, D, 3) -> element-wise multiplication of eigenvalues and x components
Sigma_x = eigenvalues * x # (H, W, D, 3)
# y = E @ Sigma_x
# E: (H, W, D, 3, 3), Sigma_x: (H, W, D, 3)
y = np.einsum('...ij,...j->...i', eigenvectors, Sigma_x)
N = np.linalg.norm(y, axis=3) # (H, W, D)

# 5. Decompose noise field
# Define icosahedron vertices
icosahedron_vertices = get_icosahedron_vertices() # (6, 3)
N_lp, N_k = decompose_noise_field(N, icosahedron_vertices)

# 6. Precompute dual basis M_k for icosahedron directions
# This is a complex calculation not detailed in sources, assumed precomputed
M_k = compute_dual_basis_for_icosahedron(icosahedron_vertices) # (6, 3, 3)

# 7. Iterative filtering
alpha = 1.0 # Not given in sources, placeholder
J = 20
current_N_lp = N_lp
current_N_k = N_k

for j in range(J):
    F = adaptive_filter_iteration(current_N_lp, current_N_k, eigenvalues, eigenvectors, alpha, M_k)
    
    # Decompose F for next iteration
    # Note: The source says "using the output noise field from the previous iteration as input"
    # It implies we re-decompose F into low-pass and high-pass components
    current_N_lp, current_N_k = decompose_noise_field(F, icosahedron_vertices)

N_out = current_N_lp + np.sum(current_N_k, axis=3) # Final output is the filtered field, which is effectively the sum of components if we consider the full reconstruction, but the source defines F as the output. 
# Actually, the source defines F as the output of the filtering step. 
# The final N_out is the result of the last iteration's F.
# However, F is a scalar field. The decomposition N_lp + sum(N_k) reconstructs the original field if filters are perfect.
# The source says "The output from the filtering process, Nout, is visualized".
# So N_out = F from the last iteration.
N_out = F

# 8. Visualize
opacity, color = visualize(N_out, eigenvalues, eigenvectors)
```

**Edge Cases and Pitfalls**
*   **Degenerate Tensors:** If $\lambda_1 = 0$, the linearity coefficient $c_l$ is undefined (division by zero). The sources do not specify a guard for this case [G].
*   **Zero Eigenvalues:** If $\lambda_2 = 0$, the control tensor computation involves division by zero. The sources do not specify a guard for this case [G].
*   **Precision:** The iterative filtering process may accumulate numerical errors. The sources do not specify precision requirements [G].
*   **Boundary Conditions:** The spherical separable filters require handling of volume boundaries. The sources do not specify boundary conditions [G].

## Subtopics

### Tensor-to-Metric Transformation
Transforms tensors into scalar metrics (e.g., FA, MD) for visualization. Chosen when a single scalar value per voxel is sufficient to represent diffusion characteristics, rather than a full field correlation. [Tensor-to-Metric Transformation](10784-tensor-to-metric-transformation.md)

### Apparent Diffusion Coefficient (ADC)
Computes a scalar measure of diffusion magnitude. Chosen for quantifying overall diffusion intensity, ignoring directionality. [Apparent Diffusion Coefficient (ADC)](12151-apparent-diffusion-coefficient-adc.md)

### Diffusion Tensor Reconstruction via Spherical Harmonics
Reconstructs the diffusion orientation distribution function (ODF) using spherical harmonics. Chosen when high angular resolution is needed to resolve crossing fibers, rather than a simple anisotropic noise correlation. [Diffusion Tensor Reconstruction via Spherical Harmonics](14002-diffusion-tensor-reconstruction-via-spherical-harmonics.md)

## Operating characteristics
*   **Time Complexity:** The method involves spectral decomposition per voxel ($O(1)$ per voxel), noise generation ($O(N)$), and $J$ iterations of filtering. Each filtering iteration involves decomposing the noise field into 7 components (1 low-pass, 6 high-pass) and computing inner products. The complexity is dominated by the filtering operations, which are convolution-like. With $J$ iterations, the time complexity is $O(J \cdot N \cdot K)$, where $N$ is the number of voxels and $K$ is the filter kernel size.
*   **Memory Complexity:** Requires storing the tensor field, eigenvalues, eigenvectors, noise fields ($N_{lp}$, $N_k$), and intermediate results. Memory usage is $O(N)$ for the tensor field and $O(N)$ for the noise fields.
*   **Measured Performance:** The method was applied to a $128^3$ voxel volume resampled from 3 Tesla DT-MRI data with original resolution $0.86 \times 5 \times 0.86$ mm. The resulting volume was rendered in real time [S1, unofficial].
*   **Strengths:** Not sensitive to occlusion, does not depend on starting points, provides a continuous scalar field representation, handles non-linear diffusion better than streamlines or LIC.
*   **Limitations:** Requires iterative filtering which can be computationally expensive. The quality of the visualization depends on the choice of parameters ($\alpha$, $J$). The method assumes the diffusion tensor accurately represents the underlying tissue structure.
*   **When to Use:** When a continuous, occlusion-resistant visualization of fiber tracts is needed, and computational resources allow for iterative filtering.
*   **When Not to Use:** When real-time performance on very large datasets is critical and iterative filtering is too slow, or when directional information beyond the primary eigenvector is needed (e.g., crossing fibers).

## Background
The method was proposed by researchers from Valladolid, Spain, and Brigham and Women's Hospital, Harvard Medical School, Boston [S1, unofficial]. It builds on the concept of adaptive filtering [1, 3] and was previously used to visualize strain-rate tensors in cardiac studies [2]. The specific application to diffusion tensor visualization aims to overcome limitations of glyphs, streamlines, and line integral convolution, such as occlusion sensitivity and dependence on starting points [S1, unofficial]. The work was funded by the Swedish Research Council, the Swedish Heart and Lung Foundation, NIH grant P41-RR13218, and CIMIT [S1, unofficial].

## Sources
[S1] Diffusion tensor visualization using random field correlation ..., https://www.diva-portal.org/smash/get/diva2:360926/FULLTEXT01.pdf, fetched 2026-10-30 (unofficial)
