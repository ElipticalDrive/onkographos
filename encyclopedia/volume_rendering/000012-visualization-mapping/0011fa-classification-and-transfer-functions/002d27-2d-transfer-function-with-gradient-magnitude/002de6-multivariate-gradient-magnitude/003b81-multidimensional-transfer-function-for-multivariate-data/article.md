# Multidimensional Transfer Function for Multivariate Data

Also called: Color Volume Transfer Function, Tensor-based Transfer Function

## Overview
Multidimensional transfer functions (MTFs) visualize multivariate data by assigning different data components to distinct axes of a classification space, allowing features that are indistinguishable in single-variable spaces to be isolated [S1]. Unlike scalar volume rendering which relies on a single intensity axis, this method generalizes gradient-based edge detection by incorporating a first-derivative measure of total local change across all data components as an additional axis [S1].

## How it works
**Inputs and Outputs**
*   **Input:** A multivariate dataset represented as a function $\mathbf{f}(x,y,z)$ where each voxel contains $m$ scalar values (e.g., RGB color, or combined MRI/CT/PET values).
*   **Output:** A classification space (typically 2D or 3D) where axes correspond to data components and/or derivative measures, used to map opacity and color for volume rendering.

**Data Structures**
*   **Multivariate Function $\mathbf{f}$:** An array of shape $(N_x, N_y, N_z, m)$, where $N_x, N_y, N_z$ are grid dimensions and $m$ is the number of variables per sample.
*   **Jacobian Matrix $\text{Df}$:** A matrix of first partial derivatives. For a 3D grid, this is computed per voxel.
*   **Tensor $\mathbf{G}$:** A $3 \times 3$ symmetric matrix (tensor) per voxel, derived from the Jacobian.

**Steps**

1.  **Represent Data:** Define the dataset as a multivariate function:
    $$ \mathbf{f}(x, y, z) = (f_1(x, y, z), f_2(x, y, z), \dots, f_m(x, y, z)) $$
    where $f_i$ represents the $i$-th data component (e.g., Red, Green, Blue) [S1].

2.  **Compute Jacobian:** Calculate the matrix of first partial derivatives $\text{Df}$ for all $m$ components across the 3 spatial dimensions:
    $$ \text{Df} = \begin{bmatrix} \frac{\partial f_1}{\partial x} & \frac{\partial f_1}{\partial y} & \frac{\partial f_1}{\partial z} \\ \frac{\partial f_2}{\partial x} & \frac{\partial f_2}{\partial y} & \frac{\partial f_2}{\partial z} \\ \vdots & \vdots & \vdots \\ \frac{\partial f_m}{\partial x} & \frac{\partial f_m}{\partial y} & \frac{\partial f_m}{\partial z} \end{bmatrix} $$
    This matrix has dimensions $m \times 3$ [S1].

3.  **Form Tensor $\mathbf{G}$:** Compute the $3 \times 3$ tensor $\mathbf{G}$ by multiplying the transpose of the Jacobian with the Jacobian:
    $$ \mathbf{G} = (\text{Df})^{\text{T}} \text{Df} $$
    This tensor captures the directional dependence of total change [S1].

4.  **Compute Squared Local Contrast:** For a given direction vector $\mathbf{n}$, calculate the squared local contrast $S(\mathbf{n})$ using the quadratic form:
    $$ S(\mathbf{n}) = \mathbf{n}^T \mathbf{G} \mathbf{n} $$
    This value serves as the first-derivative measure representing the total amount of local change across all data components in direction $\mathbf{n}$ [S1].

5.  **Construct Transfer Function Axes:**
    *   **Data Axes:** Select $k$ axes from the original data components (e.g., R, G, B) or transformed color spaces (HSV, HLS, CIE) [S1].
    *   **Derivative Axis:** Include the computed first-derivative measure (total local change) as an additional axis. This generalizes the use of gradient magnitude in scalar data [S1].

6.  **Classification and Manipulation:**
    *   Due to the impracticality of manipulating the full $m$-dimensional space, classify regions by manipulating the transfer function using two axes at a time [S1].
    *   Represent classified regions as projections onto the selected two axes [S1].

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_multivariate_transfer_function(data, num_components):
    """
    Computes the Jacobian and Tensor G for multivariate volume data.
    
    Args:
        data: numpy array of shape (Nx, Ny, Nz, m) containing multivariate values.
        num_components: integer m, the number of variables per voxel.
        
    Returns:
        G: numpy array of shape (Nx, Ny, Nz, 3, 3) containing the tensor G.
    """
    Nx, Ny, Nz, m = data.shape
    
    # Initialize Jacobian Df: shape (Nx, Ny, Nz, m, 3)
    # Df[i,j,k, component, direction]
    Df = np.zeros((Nx, Ny, Nz, m, 3))
    
    # Compute partial derivatives for each component
    # Using central differences for interior points, forward/backward for boundaries
    for c in range(m):
        # Derivatives along x
        Df[:, :, 1:, c, 0] = (data[:, :, 2:, c] - data[:, :, :-2, c]) / 2.0
        Df[:, :, 0, c, 0] = data[:, :, 1, c] - data[:, :, 0, c]
        Df[:, :, -1, c, 0] = data[:, :, -1, c] - data[:, :, -2, c]
        
        # Derivatives along y
        Df[:, 1:, :, c, 1] = (data[:, 2:, :, c] - data[:, :-2, :, c]) / 2.0
        Df[:, 0, :, c, 1] = data[:, 1, :, c] - data[:, 0, :, c]
        Df[:, -1, :, c, 1] = data[:, -1, :, c] - data[:, -2, :, c]
        
        # Derivatives along z
        Df[1:, :, :, c, 2] = (data[2:, :, :, c] - data[:-2, :, :, c]) / 2.0
        Df[0, :, :, c, 2] = data[1, :, :, c] - data[0, :, :, c]
        Df[-1, :, :, c, 2] = data[-1, :, :, c] - data[-2, :, :, c]

    # Compute Tensor G = (Df)^T * Df
    # Df is (Nx, Ny, Nz, m, 3). Transpose last two dims for matrix mult logic.
    # We want G[i,j,k] = sum_c Df[i,j,k,c,:]^T * Df[i,j,k,c,:]
    # This results in a 3x3 matrix per voxel.
    
    G = np.zeros((Nx, Ny, Nz, 3, 3))
    
    # Vectorized computation of G
    # Df shape: (N, m, 3) where N = Nx*Ny*Nz
    Df_flat = Df.reshape(-1, m, 3)
    
    # (Df)^T is (3, m). Df is (m, 3). Result (3, 3).
    # However, standard matrix mult: (3, m) @ (m, 3) -> (3, 3)
    # But Df is (m, 3). So we need (Df)^T @ Df.
    # Df.T is (3, m).
    
    Df_T = Df_flat.transpose(0, 2, 1) # Shape (N, 3, m)
    
    # Matrix multiplication: (N, 3, m) @ (N, m, 3) -> (N, 3, 3)
    G_flat = np.matmul(Df_T, Df_flat)
    
    G = G_flat.reshape(Nx, Ny, Nz, 3, 3)
    
    return G

def compute_squared_local_contrast(G, direction_n):
    """
    Computes S(n) = n^T G n for a given direction n.
    
    Args:
        G: numpy array of shape (3, 3) for a single voxel.
        direction_n: numpy array of shape (3,) representing vector n.
        
    Returns:
        S_n: float, the squared local contrast.
    """
    # S(n) = n^T G n
    S_n = np.dot(direction_n, np.dot(G, direction_n))
    return S_n
```

**Parameters**
*   **Color Space:** Axes can be RGB, HSV, HLS, or CIE colorimetric space. RGB is the most direct, but HSV is better for human navigation [S1].
*   **Axis Selection:** Typically two axes are manipulated at a time due to the complexity of full 3D+ manipulation [S1].

**Edge Cases and Pitfalls**
*   **Color Space Limitations:** Materials indistinguishable in one color space (e.g., RGB) will remain indistinguishable in any other color space [S1].
*   **Manipulation Complexity:** Manipulating the transfer function in the full $m$-dimensional space is impractical. The method requires projecting classified regions onto two axes at a time [S1].
*   **Boundary Conditions:** Derivative computation at grid boundaries requires careful handling (e.g., forward/backward differences) to avoid artifacts [G].

## Operating characteristics
*   **Complexity:** Computing the Jacobian requires $O(N \cdot m)$ operations where $N$ is the number of voxels and $m$ is the number of components. Computing $\mathbf{G}$ is $O(N \cdot m)$ due to the fixed $3 \times 3$ matrix multiplication per voxel.
*   **Strengths:** Generalizes gradient-based edge detection to multivariate data, allowing isolation of features based on total local change across all components [S1].
*   **Limitations:** High dimensionality makes direct manipulation difficult; requires projection-based interaction [S1]. Performance depends on the number of data components $m$.

## Background
This method was introduced in the context of interactive volume rendering for multivariate data, such as MRI/CT/PET combinations or color cryosection data (e.g., Visible Male RGB data) [S1]. It builds on previous work in color image segmentation, specifically the concept of squared local contrast [S1].

## Sources
[S1] Multidimensional Transfer Functions for Interactive Volume ..., https://www.academia.edu/5403340/Multidimensional_Transfer_Functions_for_Interactive_Volume_Rendering, fetched 2026-11-01
