# Multivariate Gradient Magnitude

Also called: L2 norm of G, Tensor G norm

## Overview
Multivariate Gradient Magnitude is a first-derivative measure for multivariate data (e.g., RGB color) that captures the total amount of local change across all data components, used as an axis in multidimensional transfer functions [S1, book]. It sets itself apart from scalar gradient magnitude by operating on a tensor derived from the Jacobian of the multivariate field, providing a rotationally invariant measure of local contrast [S1, book]. Unlike non-rotationally invariant measures of the gradient tensor used in other work [S1, book], this method uses the L2 norm of the resulting tensor $G$ to ensure directional independence [S1, book].

## How it works
**Inputs:**
- Multivariate volume data $f: \mathbb{R}^3 \to \mathbb{R}^m$, where $m$ is the number of components (e.g., $m=3$ for RGB).
- Spatial coordinates $(x, y, z)$.

**Outputs:**
- Scalar field representing the multivariate gradient magnitude $\|G\|_2$ at each voxel.

**Data Structures:**
1. **Input Data:** Array of shape $(N_x, N_y, N_z, m)$ containing normalized values $f_i \in [0, 1]$.
2. **Jacobian Matrix $Df$:** An $m \times 3$ matrix at each voxel.
   - Rows correspond to data components $f_1 \dots f_m$.
   - Columns correspond to spatial derivatives $\partial/\partial x, \partial/\partial y, \partial/\partial z$.
   - Layout: $Df_{ij} = \frac{\partial f_i}{\partial x_j}$, where $x_1=x, x_2=y, x_3=z$.
3. **Tensor $G$:** A $3 \times 3$ symmetric matrix computed as $G = (Df)^T Df$.
4. **Output Scalar:** $\|G\|_2$.

**Parameters:**
- **Normalization:** Individual channel data values $f_i$ are normalized to the range $[0, 1]$ before derivative calculation to ensure equal contribution from each channel [S1, book].

**Steps:**
1. **Normalize Input:** Scale each component $f_i$ of the multivariate data to $[0, 1]$.
2. **Compute Jacobian $Df$:** For each voxel, calculate the first partial derivatives for each component with respect to $x, y, z$.
   - $Df_{i,1} = \frac{\partial f_i}{\partial x}$
   - $Df_{i,2} = \frac{\partial f_i}{\partial y}$
   - $Df_{i,3} = \frac{\partial f_i}{\partial z}$
3. **Compute Tensor $G$:** Calculate the $3 \times 3$ tensor $G$ using the formula:
   $$ G = (Df)^T Df $$
   This results in a matrix where $G_{jk} = \sum_{i=1}^m \frac{\partial f_i}{\partial x_j} \frac{\partial f_i}{\partial x_k}$.
4. **Compute Magnitude:** Calculate the L2 norm of $G$, denoted $\|G\|_2$. This is defined as the square root of the sum of the squares of the individual matrix components of $G$:
   $$ \|G\|_2 = \sqrt{\sum_{j=1}^3 \sum_{k=1}^3 G_{jk}^2} $$
   Alternatively, this is equivalent to the L2 norm of the three eigenvalues of $G$ [S1, book].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def compute_multivariate_gradient_magnitude(volume_data):
    """
    Computes the multivariate gradient magnitude for a volume.
    
    Args:
        volume_data: numpy array of shape (Nx, Ny, Nz, m)
                     containing multivariate data (e.g., RGB).
                     
    Returns:
        numpy array of shape (Nx, Ny, Nz) containing the gradient magnitude.
    """
    # Step 1: Normalize input data to [0, 1]
    # [G] Assuming standard min-max normalization if not already normalized.
    # If data is already normalized, this step is skipped.
    min_val = np.min(volume_data, axis=(0, 1, 2), keepdims=True)
    max_val = np.max(volume_data, axis=(0, 1, 2), keepdims=True)
    range_val = max_val - min_val
    # Avoid division by zero
    range_val[range_val == 0] = 1.0
    normalized_data = (volume_data - min_val) / range_val
    
    # Step 2: Compute Jacobian Df
    # Df shape: (Nx, Ny, Nz, m, 3)
    # We compute partial derivatives for each component f_i
    # Using central differences for interior points, forward/backward for boundaries
    # [G] Implementation of numerical differentiation is assumed standard.
    # Here we use numpy.gradient which handles boundaries with one-sided differences.
    
    # Compute gradients for each component
    # gradients[i] will be shape (Nx, Ny, Nz, 3) for component i
    gradients = np.zeros((normalized_data.shape[0], normalized_data.shape[1], 
                          normalized_data.shape[2], normalized_data.shape[3], 3))
    
    for i in range(normalized_data.shape[3]):
        # normalized_data[..., i] is shape (Nx, Ny, Nz)
        # np.gradient returns list of gradients along each axis
        gx, gy, gz = np.gradient(normalized_data[..., i])
        gradients[..., i, 0] = gx
        gradients[..., i, 1] = gy
        gradients[..., i, 2] = gz
        
    # Df is now gradients, shape (Nx, Ny, Nz, m, 3)
    # We need to compute G = (Df)^T Df for each voxel
    # G is 3x3. G[j,k] = sum_i (Df[i,j] * Df[i,k])
    
    # Reshape for efficient computation
    # Let's compute G directly using einsum or manual summation
    # G[j,k] = sum over i of Df[i,j] * Df[i,k]
    
    # Initialize output magnitude array
    magnitude = np.zeros((normalized_data.shape[0], normalized_data.shape[1], 
                          normalized_data.shape[2]))
    
    # Compute G and its norm for each voxel
    # Using vectorized operations
    # Df shape: (Nx, Ny, Nz, m, 3)
    # We want G shape: (Nx, Ny, Nz, 3, 3)
    
    # G[j,k] = sum_i (Df[..., i, j] * Df[..., i, k])
    # This is equivalent to matrix multiplication of Df^T and Df for each voxel
    # Df^T is (3, m), Df is (m, 3) -> result (3, 3)
    
    # Transpose Df to (Nx, Ny, Nz, 3, m)
    Df_T = np.transpose(gradients, (0, 1, 2, 4, 3))
    
    # Matrix multiply: (Nx, Ny, Nz, 3, m) @ (Nx, Ny, Nz, m, 3) -> (Nx, Ny, Nz, 3, 3)
    # Using einsum for clarity and efficiency
    # 'ijkm,ikln->ijlm' -> sum over k (component index)
    G = np.einsum('ijkm,ikln->ijlm', Df_T, gradients)
    
    # Compute L2 norm of G
    # Sum of squares of all components of G
    sum_sq = np.sum(G**2, axis=(3, 4)) # Sum over the last two dimensions (3,3)
    magnitude = np.sqrt(sum_sq)
    
    return magnitude
```

**Edge Cases and Pitfalls:**
- **Boundary Conditions:** Numerical differentiation at volume boundaries requires one-sided differences, which may affect accuracy [S1, book].
- **Dynamic Range:** If channel dynamic ranges differ significantly, normalization is critical to prevent one channel from dominating the derivative calculation [S1, book].
- **Rotation Invariance:** The L2 norm of $G$ is rotationally invariant, unlike some other measures of $G$ [S1, book].

## Subtopics

### [Rectangular Classification Widget](5634-rectangular-classification-widget.md)
A tool for defining classification regions in the transfer function space. It complements Multivariate Gradient Magnitude by providing a UI for selecting regions based on the computed magnitude and other dimensions.

## Operating characteristics
- **Time Complexity:** $O(N \cdot m)$, where $N$ is the number of voxels and $m$ is the number of components. Computing the Jacobian requires $O(N \cdot m)$ derivatives, and computing $G$ and its norm is $O(N \cdot m \cdot 3^2) = O(N \cdot m)$.
- **Memory Complexity:** $O(N \cdot m \cdot 3)$ for storing the Jacobian, plus $O(N \cdot 3^2)$ for $G$, and $O(N)$ for the output.
- **Strengths:** Provides a rotationally invariant measure of local change in multivariate data, allowing for effective isolation of features based on the total amount of change across all components [S1, book].
- **Limitations:** Computationally more expensive than scalar gradient magnitude due to the need to compute and process the tensor $G$. Requires normalization of input channels.
- **When to Use:** When visualizing multivariate data (e.g., RGB color) and needing to highlight boundaries or regions of high local change across all components, especially when rotational invariance is desired [S1, book].
- **When Not to Use:** When computational resources are extremely limited, or when the data is scalar (use scalar gradient magnitude instead).

## Background
The method originates from work on multidimensional transfer functions for volume rendering, specifically in the context of color image segmentation and visualization [S1, book]. It builds on previous work in color image segmentation [S1, book] where the gradient tensor $G$ was used to analyze directional dependence of change. The specific use of the L2 norm of $G$ as a rotationally invariant measure of local change was motivated by the observation that it is equivalent to the L2 norm of the eigenvalues of $G$ [S1, book]. This approach was introduced in the Visualization Handbook chapter on Multi-Dimensional Transfer Functions for Volume Rendering [S1, book].

## Sources
[S1] Multi-Dimensional Transfer Functions for Volume Rendering, https://people.cs.uchicago.edu/~glk/pubs/pdf/Kniss-MultidimensionalTransferFunctions-VH-2005.pdf, fetched 2026-10-27 (book)
