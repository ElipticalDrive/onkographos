# Multivariate Gradient Magnitude

Also called: Gradient measure for multivariate fields, L2 Norm of Structure Tensor for Local Change, L2 norm of G, Multi-field gradient, Multi-gradient magnitude estimation, Tensor G norm, ||G|| measure

## Overview
Multivariate Gradient Magnitude provides a scalar measure of local change in multidimensional volume data (e.g., RGB color, meteorological fields) by computing the L2 norm of the structure tensor $G$ [S2]. It solves the problem that gradient estimation for multivalued fields is not well-defined in the same way as for scalar datasets, where a single gradient vector exists [S3, unofficial]. Unlike scalar volume rendering which relies on a single intensity axis, this method generalizes gradient-based edge detection by incorporating a first-derivative measure of total local change across all data components as an additional axis in multidimensional transfer functions [S1, book]. It is distinguished from its siblings by its use of the rotationally invariant L2 norm of the tensor, rather than the principal eigenvalue or non-rotationally invariant measures [S1, book][S2].

## How it works
**Inputs and Outputs:**
*   **Input:** A multivariate 3D field $\mathbf{f}(x,y,z) = [f_1, f_2, \dots, f_m]^T$, where each sample is a vector in $\mathbb{R}^m$ [S3, unofficial].
*   **Output:** A scalar field of gradient magnitudes $\|G\|_2$ at each voxel, used as an axis in a transfer function [S1, book][S2].

**Data Structures:**
1.  **Weighted Jacobian Matrix ($Df$):** An $m \times 3$ matrix containing the partial derivatives of each data channel with respect to spatial coordinates $x, y, z$ [S1, book][S2].
    *   Layout: $Df_{ij} = w_i \frac{\partial f_i}{\partial x_j}$, where $i \in \{1, \dots, m\}$ and $j \in \{1, 2, 3\}$ corresponding to $x, y, z$ [S1, book][S3, unofficial].
2.  **Structure Tensor ($G$):** A $3 \times 3$ symmetric matrix [S1, book][S2].
    *   Layout: $G = (Df)^T Df$. The element $G_{jk} = \sum_{i=1}^m Df_{ij} Df_{ik}$ [S2].

**Parameters:**
*   **Channel Weights ($w_i$):** Heuristic weights to normalize dynamic ranges. Calculated as $w_i = \frac{1}{\max(f_i) - \min(f_i)}$ [S3, unofficial]. Alternatively, data values can be normalized to $[0, 1]$ before derivative calculation to ensure equal contribution [S1, book][S2].
*   **Derivative Kernel:** Not specified in sources [G]. Typically central differences or Sobel filters are used for discrete voxel data.

**Steps:**
1.  **Preprocessing:** Normalize each data channel $f_i$ to the range $[0, 1]$ or compute weights $w_i$ based on the dynamic range of each channel [S1, book][S2][S3, unofficial].
2.  **Jacobian Computation:** Compute the partial derivatives $\frac{\partial f_i}{\partial x}, \frac{\partial f_i}{\partial y}, \frac{\partial f_i}{\partial z}$ for each channel $i$ at the current voxel [S2]. Construct the $m \times 3$ Jacobian matrix $Df$, applying weights $w_i$ to the derivatives of channel $i$ [S3, unofficial].
3.  **Tensor Formation:** Compute the $3 \times 3$ structure tensor $G$ by matrix multiplication:
    $$ G = (Df)^T Df $$
    This captures the directional dependence of total change [S1, book][S2].
4.  **Magnitude Calculation:** Compute the L2 norm of $G$:
    $$ \|G\|_2 = \sqrt{\sum_{j=1}^3 \sum_{k=1}^3 G_{jk}^2} $$
    This is equivalent to the square root of the sum of the squares of the three eigenvalues of $G$ [S1, book][S2].
5.  **Application:** Use $\|G\|_2$ as an additional axis in a multidimensional transfer function to classify features based on local change intensity [S1, book][S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_multivariate_gradient_magnitude(field, weights=None):
    """
    Computes the L2 norm of the structure tensor for a multivariate volume.
    
    Args:
        field: 4D array (depth, height, width, channels) representing f(x,y,z).
        weights: 1D array of length m (channels). If None, assumes normalized data.
        
    Returns:
        3D array of gradient magnitudes.
    """
    depth, height, width, m = field.shape
    result = np.zeros((depth, height, width))
    
    # Precompute weights if not provided
    if weights is None:
        # Heuristic: reciprocal of dynamic range
        min_vals = np.min(field, axis=(0, 1, 2))
        max_vals = np.max(field, axis=(0, 1, 2))
        weights = 1.0 / (max_vals - min_vals)
        # Handle potential division by zero if a channel is constant
        weights = np.where(weights == np.inf, 0.0, weights)

    # Compute derivatives using central differences (G)
    # Note: Boundary conditions are not specified in sources [G]
    # Derivatives for each channel i
    df_dx = np.zeros((depth, height, width, m))
    df_dy = np.zeros((depth, height, width, m))
    df_dz = np.zeros((depth, height, width, m))

    for i in range(m):
        channel = field[:, :, :, i]
        # Central differences
        df_dx[:, :, :, i] = np.gradient(channel, axis=2) # x axis
        df_dy[:, :, :, i] = np.gradient(channel, axis=1) # y axis
        df_dz[:, :, :, i] = np.gradient(channel, axis=0) # z axis

    # Apply weights to derivatives
    for i in range(m):
        w = weights[i]
        df_dx[:, :, :, i] *= w
        df_dy[:, :, :, i] *= w
        df_dz[:, :, :, i] *= w

    # Compute Structure Tensor G for each voxel
    # G is 3x3. G_jk = sum_i (df_j_i * df_k_i)
    # We can compute this efficiently using einsum or matrix multiplication per voxel
    
    # Reshape for vectorized operation: (depth*height*width, m, 3)
    # df_dx, df_dy, df_dz are (D, H, W, m)
    # Stack them to get (D, H, W, m, 3)
    derivatives = np.stack([df_dx, df_dy, df_dz], axis=-1) # Shape: (D, H, W, m, 3)
    
    # Transpose to (D, H, W, 3, m) to compute (Df)^T Df
    # Df is (m, 3). (Df)^T is (3, m).
    # We want G = Df^T @ Df -> (3, m) @ (m, 3) -> (3, 3)
    # derivatives shape is (D, H, W, m, 3). 
    # Let's reshape to (N, m, 3) where N = D*H*W
    N = depth * height * width
    derivatives_flat = derivatives.reshape(N, m, 3)
    
    # Compute G for all voxels
    # G = derivatives_flat^T @ derivatives_flat
    # derivatives_flat^T shape: (N, 3, m)
    # Result shape: (N, 3, 3)
    G_flat = np.einsum('nij,nik->njk', derivatives_flat, derivatives_flat)
    
    # Compute L2 norm of G
    # ||G||_2 = sqrt(sum(G_jk^2))
    G_squared = G_flat ** 2
    norm_sq = np.sum(G_squared, axis=(1, 2))
    result = np.sqrt(norm_sq).reshape(depth, height, width)
    
    return result
```

## Subtopics

### [Multidimensional Transfer Function for Multivariate Data](9004-multidimensional-transfer-function-for-multivariate-data.md)
Multidimensional transfer functions (MTFs) visualize multivariate data by assigning different data components to distinct axes of a classification space, allowing features that are indistinguishable in single-variable spaces to be isolated. Unlike scalar volume rendering which relies on a single intensity axis, this method generalizes gradient-based edge detection by incorporating a first-derivative measure of total local change across all data components as an additional axis.

## Operating characteristics
*   **Time Complexity:** $O(m \cdot D \cdot H \cdot W)$, where $m$ is the number of channels and $D, H, W$ are volume dimensions. The tensor computation is constant time per voxel ($3 \times 3$ matrix operations).
*   **Memory Complexity:** $O(m \cdot D \cdot H \cdot W)$ to store the field and derivatives. The tensor $G$ is $3 \times 3$ per voxel.
*   **Performance:** Computation is efficient enough for interactive rendering if precomputed. For fields with three or more values per sample, the gradient portion of the transfer function may need to be applied separably due to graphics hardware limitations [S3, unofficial].
*   **Strengths:** Rotationally invariant; directionally independent; effective for identifying interfaces in multivariate data (e.g., air mass fronts, RGB boundaries) [S1, book][S2][S3, unofficial].
*   **Limitations:** Sensitive to dynamic range differences between channels if not weighted [S3, unofficial]. The heuristic weights assume equal influence and no outliers, which may not hold for all datasets [S3, unofficial]. Lighting models cannot use the gradient as a surface normal because the eigenvector direction can flip; atmospheric lighting is required [S3, unofficial].
*   **When to use:** When visualizing multivariate data (color, meteorological) where feature boundaries are defined by changes in multiple components simultaneously.
*   **When not to use:** For scalar data (use standard gradient magnitude) or when the direction of change is more important than the magnitude (use principal eigenvector of $G$) [S2].

## Background
The method originates from color image segmentation techniques [S1, book][S2][S3, unofficial]. It was adapted for volume rendering by Kniss, Hansen, and others to handle multivariate data such as the Visible Male RGB dataset and meteorological simulations [S1, book][S2]. The formulation is based on the structure tensor from computer vision, specifically the work of Cumani [7] and others cited in [S1, book][S2]. It addresses the limitation that standard gradient estimation is not well-defined for multivalued fields [S3, unofficial].

## Sources
[S1] Multi-Dimensional Transfer Functions for Volume Rendering, https://people.cs.uchicago.edu/~glk/pubs/pdf/Kniss-MultidimensionalTransferFunctions-VH-2005.pdf, fetched 2026-10-27 (book)
[S2] Multidimensional Transfer Functions for Interactive Volume ..., https://www.academia.edu/5403340/Multidimensional_Transfer_Functions_for_Interactive_Volume_Rendering, fetched 2026-11-01
[S3] Volume rendering multivariate data to visualize meteorological ..., https://www.researchgate.net/publication/262237652_Volume_rendering_multivariate_data_to_visualize_meteorological_simulations_a_case_study, fetched 2026-10-29 (unofficial)
