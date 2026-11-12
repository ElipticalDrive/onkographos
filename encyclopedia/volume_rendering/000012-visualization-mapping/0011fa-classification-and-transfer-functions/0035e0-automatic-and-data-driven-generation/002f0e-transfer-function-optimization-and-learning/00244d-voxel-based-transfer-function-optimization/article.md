# Voxel-based Transfer Function Optimization

Also called: Least Squares Transfer Function Optimization, Linear Transfer Function Optimization

## Overview
Voxel-based Transfer Function Optimization automatically computes an optimized transfer function $T_o$ for a scalar field $F_o$ such that its pre-shaded voxel colors and opacities match those of a reference field $F_r$ rendered with a reference transfer function $T_r$ [S1, unofficial]. It differs from image-based approaches by minimizing the difference between pre-shaded voxel values rather than final rendered pixels, treating all voxels equally in the linear system rather than prioritizing those with larger visual impact [S1, unofficial]. Unlike methods that only optimize color, this approach minimizes differences in both color and opacity [S1, unofficial].

## How it works
**Inputs:**
*   Reference scalar field $F_r$ defined on a rectangular grid.
*   Optimization scalar field $F_o$ defined on the same grid structure.
*   Reference transfer function $T_r$ (discretized).
*   Number of transfer function bins $n_T$.

**Outputs:**
*   Optimized transfer function $T_o$ (discretized into $n_T$ bins).

**Data Structures:**
*   **Transfer Function Bins:** $n_T$ regularly spaced bins. The transfer function is defined by linear interpolation between these bins. Each bin has 4 components (R, G, B, $\alpha$).
*   **System Matrix $A$:** Dimensions $m \times n$, where $m = 4m_V$ (rows, one per RGB$\alpha$ entry per voxel) and $n = 4n_T$ (columns, one per RGB$\alpha$ entry per bin) [S1, unofficial].
*   **Vector $V$:** Dimensions $m \times 1$, containing linearized RGB$\alpha$ values for all voxels [S1, unofficial].
*   **Vector $T$:** Dimensions $n \times 1$, containing the linearized RGB$\alpha$ values of the transfer function bins [S1, unofficial].

**Steps:**

1.  **Discretize Transfer Functions:** Discretize $T_r$ and $T_o$ into $n_T$ regularly spaced bins with linear interpolation between them [S1, unofficial].
2.  **Compute Voxel Colors:** For each voxel $i$ in $F_r$ and $F_o$, calculate the density $D_i$ at the voxel vertex. Apply the respective transfer function to $D_i$ to obtain the color $V_i = (C_i, \alpha_i)$ [S1, unofficial].
    *   Let $j$ be the bin index quantizing $D_i$.
    *   Let $w(i)$ be the interpolation weight between bin $j$ and $j+1$.
    *   The color is computed as:
        $$ V_i = (1 - w(i))T(D_i)_j + w(i)T(D_i)_{j+1} $$
        where $T(D_i)_j$ represents the 4-component vector (RGB$\alpha$) of bin $j$ evaluated at density $D_i$ [S1, unofficial].
3.  **Construct System Matrices:** Formulate the linear system $AT = V$ for all voxels.
    *   For the reference field, construct $A_r$ and $V_r$.
    *   For the optimization field, construct $A_o$ and $V_o$.
    *   The matrix $A$ is sparse. For linearized RGB$\alpha$ values, there are only two non-zero entries per row. There is an additional stride of 4 between the two non-zero row entries because different color channels (R, G, B, $\alpha$) do not interact [S1, unofficial].
4.  **Formulate Least Squares Problem:** The goal is to find $T_o$ such that $A_o T_o$ closely matches $V_r$. This is an over-determined system ($m \gg n$) [S1, unofficial].
    *   Define the energy term to minimize:
        $$ E(T_o) = ||V_r - A_o T_o||_2^2 = (V_r - A_o T_o)^T (V_r - A_o T_o) $$
    *   Let $A = A_o$, $x = T_o$, and $b = V_r$ [S1, unofficial].
5.  **Solve Normal Equations:** Solve the least squares problem using the normal equations:
    $$ A^T A x = A^T b $$
    *   The solution is:
        $$ x = (A^T A)^{-1} A^T b $$
        provided $A$ has full rank [S1, unofficial].
    *   Note: The condition number of $A^T A$ is the square of the condition number of $A$ ($\text{cond}(A^T A) = \text{cond}(A)^2$), which can lead to numerical instability if the matrix is ill-conditioned [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def voxel_based_tf_optimization(F_r, F_o, T_r_bins, n_T):
    """
    F_r: Reference field density values at vertices (m_V x 1)
    F_o: Optimization field density values at vertices (m_V x 1)
    T_r_bins: Reference transfer function bin values (n_T x 4)
    n_T: Number of transfer function bins
    
    Returns:
    T_o: Optimized transfer function bin values (n_T x 4)
    """
    m_V = len(F_r)
    m = 4 * m_V
    n = 4 * n_T
    
    # Step 1 & 2: Compute V_r (Reference Voxel Colors)
    # V_r is a vector of size m (4 components per voxel)
    V_r = compute_voxel_colors(F_r, T_r_bins, n_T)
    
    # Step 3: Construct System Matrix A_o for Optimization Field
    # A_o is sparse matrix of size m x n
    A_o = construct_system_matrix(F_o, n_T)
    
    # Step 4 & 5: Solve Least Squares Problem A_o * T_o = V_r
    # Using Normal Equations: (A_o^T A_o) T_o = A_o^T V_r
    
    # Compute A_o^T A_o
    ATA = A_o.T @ A_o
    
    # Compute A_o^T V_r
    ATb = A_o.T @ V_r
    
    # Solve linear system ATA * T_o = ATb
    # Check for singularity/ill-conditioning [G]
    try:
        T_o_flat = solve_linear_system(ATA, ATb)
    except SingularMatrixError:
        # Fallback or regularization if A is not full rank [G]
        T_o_flat = solve_linear_system(ATA + lambda * I, ATb)
        
    # Reshape T_o_flat from (n,) to (n_T, 4)
    T_o = T_o_flat.reshape(n_T, 4)
    
    return T_o

def compute_voxel_colors(densities, T_bins, n_T):
    """
    Computes linearized RGBa colors for all voxels given densities and TF bins.
    Returns vector of size 4 * len(densities)
    """
    m_V = len(densities)
    colors = np.zeros(4 * m_V)
    
    for i in range(m_V):
        d = densities[i]
        # Quantize density to bin index j
        j = quantize_density(d, n_T)
        
        # Calculate interpolation weight w(i)
        w = interpolation_weight(d, j, n_T)
        
        # Get TF entries for bin j and j+1
        # T_bins is (n_T, 4)
        T_j = T_bins[j]
        T_j1 = T_bins[min(j+1, n_T-1)] # Handle boundary
        
        # Linear interpolation: V_i = (1-w)*T_j + w*T_j1
        V_i = (1 - w) * T_j + w * T_j1
        
        # Store in vector (R, G, B, Alpha)
        colors[4*i : 4*i+4] = V_i
        
    return colors

def construct_system_matrix(densities, n_T):
    """
    Constructs the sparse system matrix A for the optimization field.
    A is size (4*m_V) x (4*n_T).
    """
    m_V = len(densities)
    m = 4 * m_V
    n = 4 * n_T
    A = sparse.lil_matrix((m, n))
    
    for i in range(m_V):
        d = densities[i]
        j = quantize_density(d, n_T)
        w = interpolation_weight(d, j, n_T)
        
        # The color V_i is a linear combination of T_j and T_j1
        # V_i = (1-w)*T_j + w*T_j1
        # This affects 4 rows in A (one for each RGBa component)
        # and 8 columns (4 for T_j, 4 for T_j1)
        
        # Row indices for voxel i
        row_start = 4 * i
        
        # Column indices for bin j
        col_start_j = 4 * j
        # Column indices for bin j+1
        col_start_j1 = 4 * min(j+1, n_T-1)
        
        # Fill matrix entries
        # For each channel c in 0..3
        for c in range(4):
            # Entry for T_j
            A[row_start + c, col_start_j + c] = 1 - w
            # Entry for T_j1
            A[row_start + c, col_start_j1 + c] = w
            
    return A.tocsr() # Convert to CSR for efficient multiplication

def quantize_density(d, n_T):
    """
    Maps density value d to bin index j.
    Assumes density is normalized or scaled to [0, n_T-1] range for binning.
    Implementation detail depends on specific density normalization [G]
    """
    # Placeholder for specific binning logic based on source [G]
    # Typically: j = floor(d * (n_T - 1)) if d in [0,1]
    return int(d * (n_T - 1))

def interpolation_weight(d, j, n_T):
    """
    Calculates weight w for linear interpolation between bin j and j+1.
    """
    # w = d - j (if d is scaled to [0, n_T-1])
    return d - j
```

## Operating characteristics
*   **Time Complexity:** Solving the normal equations involves computing $A^T A$ ($O(mn^2)$ or less due to sparsity) and solving the linear system ($O(n^3)$ or less with sparse solvers). Since $m \gg n$, the dominant cost is often the matrix construction and multiplication.
*   **Memory Complexity:** Storing $A$ requires $O(m)$ space due to sparsity (2 non-zeros per row). Storing $A^T A$ requires $O(n^2)$ space.
*   **Strengths:** Fast computation as it reduces to a linear least squares problem. Considers both color and opacity.
*   **Limitations:** Numerical instability due to squaring of the condition number in normal equations. Treats all voxels equally, ignoring their contribution to the final image (unlike image-based methods). Assumes linear relationship between TF entries and voxel colors, which holds for pre-shading but not for the full volume rendering integral [S1, unofficial].
*   **When to use:** When a fast, automatic approximation of a transfer function is needed, and the linear assumption is valid. When computational resources are limited.
*   **When not to use:** When high accuracy in the final rendered image is critical and the non-linear nature of volume rendering (opacity accumulation) significantly affects the result. When the system matrix is ill-conditioned.

## Background
This method was proposed in the context of comparative volume rendering to automatically optimize transfer functions [S1, unofficial]. It addresses the problem of matching a reference volume's appearance with an optimization volume's transfer function. The approach formulates the problem as a least squares optimization on voxel colors, contrasting with image-based differentiable volume rendering approaches [S1, unofficial]. It builds on earlier work in transfer function optimization using least squares, such as that by Berger et al. [BLL19] and Günther et al. [GRT13], but specifically applies it to voxel-level color residuals for comparative rendering [S1, unofficial].

## Sources
[S1] Transfer Function Optimization for Comparative Volume Rendering, https://doi.org/10.48550/arxiv.2306.05885, fetched 2026-10-26 (unofficial)
