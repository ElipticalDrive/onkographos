# Multi-dimensional Transfer Functions for 3D Flow Visualization

Also called: MDTFs for flow, Multi-dimensional Transfer Function (MDTF), N-dimensional transfer function, Vector field MDTF

## Overview
Multi-dimensional Transfer Functions (MDTFs) for 3D flow visualization are a feature-based classification method that maps multiple scalar properties derived from vector calculus (velocity magnitude, gradient determinant, curl magnitude, helicity, and divergence) to RGBA optical properties to extract and track subtle flow features [S2]. Unlike standard 1D transfer functions that rely solely on velocity magnitude, MDTFs alleviate the occlusion problems inherent in texture-based flow visualization by allowing distinct visual encoding of complex flow characteristics [S2].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D discrete vector flow field $\mathbf{F}$ defined on a grid, where $\mathbf{F}(i, j, k) = (F_1, F_2, F_3)$ represents the velocity components at grid point $(i, j, k)$ [S2].
*   **Output:** A 5-dimensional feature vector $\mathbf{x} = (x_1, x_2, x_3, x_4, x_5)$ for each grid point, and a resulting RGBA color and opacity value $(R, G, B, \alpha)$ assigned via the transfer function $T: \mathbb{R}^5 \rightarrow \mathbb{R}^4$ [S2].

**Data Structures:**
The velocity gradient tensor is computed as a 3×3 matrix at each grid point [S2]. The five scalar properties are evaluated per voxel to form the input vector $\mathbf{x}$ [S2].

**Numbered Steps:**

1.  **Compute Velocity Magnitude:**
    Calculate the Euclidean norm of the velocity vector at each grid point $(i, j, k)$ [S2]:
    $$ x_1 = |\mathbf{F}(i, j, k)| = \sqrt{F_1(i, j, k)^2 + F_2(i, j, k)^2 + F_3(i, j, k)^2} $$

2.  **Compute Velocity Gradient Tensor:**
    Approximate the nine partial derivatives of the velocity components with respect to spatial coordinates using central differencing [S2]. For example, the partial derivative of the first component $F_1$ with respect to $x$ is [S2]:
    $$ \frac{\partial F_1}{\partial x}(i, j, k) = \frac{1}{2} (F_1(i+1, j, k) - F_1(i-1, j, k)) $$
    This is repeated for all combinations of components ($F_1, F_2, F_3$) and coordinates ($x, y, z$) to form the 3×3 matrix $\nabla \mathbf{F}$ [S2].

3.  **Compute Gradient Magnitude (Determinant):**
    Calculate the determinant of the velocity gradient tensor matrix [S2]. This serves as a proxy for the product of the eigenvalues, avoiding the computational cost of full eigenanalysis [S2]:
    $$ x_2 = |\nabla \mathbf{F}(x, y, z)| = \det \begin{pmatrix} \frac{\partial F_1}{\partial x} & \frac{\partial F_2}{\partial x} & \frac{\partial F_3}{\partial x} \\ \frac{\partial F_1}{\partial y} & \frac{\partial F_2}{\partial y} & \frac{\partial F_3}{\partial y} \\ \frac{\partial F_1}{\partial z} & \frac{\partial F_2}{\partial z} & \frac{\partial F_3}{\partial z} \end{pmatrix} $$

4.  **Compute Divergence:**
    Calculate the trace of the velocity gradient tensor, which represents the rate of expansion or compression [S2]:
    $$ x_3 = \text{div } \mathbf{F}(x, y, z) = \frac{\partial F_1}{\partial x} + \frac{\partial F_2}{\partial y} + \frac{\partial F_3}{\partial z} $$

5.  **Compute Curl Magnitude:**
    Calculate the curl vector $\text{curl } \mathbf{F}$, which measures vorticity, and then its magnitude [S2]. The curl vector is defined as [S2]:
    $$ \text{curl } \mathbf{F}(x, y, z) = \begin{pmatrix} \frac{\partial F_3}{\partial y} - \frac{\partial F_2}{\partial z} \\ \frac{\partial F_1}{\partial z} - \frac{\partial F_3}{\partial x} \\ \frac{\partial F_2}{\partial x} - \frac{\partial F_1}{\partial y} \end{pmatrix} $$
    The magnitude $x_4$ is the Euclidean norm of this vector [S2].

6.  **Compute Helicity:**
    Calculate the dot product of the curl vector and the velocity vector [S2]:
    $$ x_5 = \text{heli } \mathbf{F}(x, y, z) = \text{curl } \mathbf{F}(x, y, z) \cdot \mathbf{F}(x, y, z) $$

7.  **Define 1D Transfer Functions:**
    Define $N=5$ one-dimensional transfer functions $T_i: \mathbb{R} \rightarrow \mathbb{R}^4$ for $i = 1, \dots, 5$ [S2]. Each $T_i$ maps a single scalar property $x_i$ to a 4D vector containing color $(R, G, B)$ and opacity $\alpha$ [S2].

8.  **Combine into MDTF:**
    Combine the 1D functions into the 5D transfer function $T: \mathbb{R}^5 \rightarrow \mathbb{R}^4$ [S1]. The final opacity $\alpha$ is the average of the individual opacities [S1]:
    $$ \alpha = T_\alpha(\mathbf{x}) = \frac{1}{N} \sum_{i=1}^{N} T_{i,\alpha}(x_i) $$
    The final color $(R, G, B)$ is the average of the individual colors weighted by their respective opacities [S1]:
    $$ \mathbf{C} = T_{RGB}(\mathbf{x}) = \frac{1}{N} \sum_{i=1}^{N} T_{i,RGB}(x_i) \cdot T_{i,\alpha}(x_i) $$

9.  **Adjust Opacity (Optional):**
    If the resulting features appear too dark due to low combined opacity, multiply the final opacity $\alpha$ by a constant factor $k > 1$ [S2].

**Parameters:**
*   $N=5$: The number of scalar properties (velocity magnitude, gradient determinant, curl magnitude, helicity, divergence) [S2].
*   $M=4$: The dimensionality of the output (RGBA) [S2].

**Edge Cases and Pitfalls:**
*   **Computational Cost:** Full eigenanalysis of the velocity gradient tensor is computationally impractical for real-time processing of unsteady fields [S2]. The determinant is used as a cheaper proxy for the product of eigenvalues [S2].
*   **Boundary Conditions:** Central differencing requires neighbors [G]. At grid boundaries, appropriate handling (e.g., clamping, reflection, or one-sided differencing) is required, though not explicitly detailed in the source [G].
*   **Dark Features:** If extracted features are too dark, opacity adjustment is necessary [S2].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def compute_velocity_gradient_tensor(F, i, j, k, grid_shape):
    """
    Computes the 3x3 velocity gradient tensor at grid point (i, j, k)
    using central differencing.
    F is a 3D array of shape (Dx, Dy, Dz, 3) containing velocity components.
    """
    def get_val(comp, x, y, z):
        x = max(0, min(x, grid_shape[0] - 1))
        y = max(0, min(y, grid_shape[1] - 1))
        z = max(0, min(z, grid_shape[2] - 1))
        return F[x, y, z, comp]

    df1_dx = 0.5 * (get_val(0, i+1, j, k) - get_val(0, i-1, j, k))
    df1_dy = 0.5 * (get_val(0, i, j+1, k) - get_val(0, i, j-1, k))
    df1_dz = 0.5 * (get_val(0, i, j, k+1) - get_val(0, i, j, k-1))

    df2_dx = 0.5 * (get_val(1, i+1, j, k) - get_val(1, i-1, j, k))
    df2_dy = 0.5 * (get_val(1, i, j+1, k) - get_val(1, i, j-1, k))
    df2_dz = 0.5 * (get_val(1, i, j, k+1) - get_val(1, i, j, k-1))

    df3_dx = 0.5 * (get_val(2, i+1, j, k) - get_val(2, i-1, j, k))
    df3_dy = 0.5 * (get_val(2, i, j+1, k) - get_val(2, i, j-1, k))
    df3_dz = 0.5 * (get_val(2, i, j, k+1) - get_val(2, i, j, k-1))

    return [
        [df1_dx, df2_dx, df3_dx],
        [df1_dy, df2_dy, df3_dy],
        [df1_dz, df2_dz, df3_dz]
    ]

def determinant_3x3(M):
    """Computes the determinant of a 3x3 matrix."""
    return (M[0][0] * (M[1][1] * M[2][2] - M[1][2] * M[2][1]) -
            M[0][1] * (M[1][0] * M[2][2] - M[1][2] * M[2][0]) +
            M[0][2] * (M[1][0] * M[2][1] - M[1][1] * M[2][0]))

def compute_features(F, i, j, k, grid_shape):
    """
    Computes the 5 scalar features for MDTF at grid point (i, j, k).
    Returns tuple (x1, x2, x3, x4, x5).
    """
    v1 = F[i, j, k, 0]
    v2 = F[i, j, k, 1]
    v3 = F[i, j, k, 2]
    x1 = math.sqrt(v1**2 + v2**2 + v3**2)

    grad_tensor = compute_velocity_gradient_tensor(F, i, j, k, grid_shape)
    x2 = determinant_3x3(grad_tensor)

    x3 = grad_tensor[0][0] + grad_tensor[1][1] + grad_tensor[2][2]

    c1 = grad_tensor[1][2] - grad_tensor[2][1]
    c2 = grad_tensor[2][0] - grad_tensor[0][2]
    c3 = grad_tensor[0][1] - grad_tensor[1][0]
    x4 = math.sqrt(c1**2 + c2**2 + c3**2)

    x5 = c1 * v1 + c2 * v2 + c3 * v3

    return (x1, x2, x3, x4, x5)

def apply_mdtf(features, transfer_functions):
    """
    Applies the MDTF combination rules.
    features: tuple (x1, x2, x3, x4, x5)
    transfer_functions: list of 5 functions, each taking a scalar and returning (R, G, B, alpha)
    """
    N = 5
    sum_alpha = 0.0
    weighted_color = [0.0, 0.0, 0.0]

    for i in range(N):
        xi = features[i]
        r, g, b, alpha = transfer_functions[i](xi)
        
        sum_alpha += alpha
        weighted_color[0] += r * alpha
        weighted_color[1] += g * alpha
        weighted_color[2] += b * alpha

    final_alpha = sum_alpha / N
    final_color = [c / N for c in weighted_color]

    return final_color, final_alpha
```

## Operating characteristics
*   **Time Complexity:** Not given in the sources. [G]
*   **Memory Complexity:** Requires storing the original vector field and the computed feature volumes (5 scalar fields) if precomputed, or computing them on the fly [G].
*   **Measured Performance:** The method was implemented on an NVIDIA GeForce FX GPU to achieve interactive frame rates for steady and unsteady flow visualization [S2].
*   **Strengths:** Alleviates occlusion problems inherent in texture-based flow visualization by allowing distinct visual encoding of flow features [S2]. Can extract subtle features that 1D transfer functions miss [S2].
*   **Limitations:** Computationally expensive if full eigenanalysis were used (mitigated by using determinant) [S2]. Requires careful tuning of 5 transfer functions [G].
*   **When to use:** When visualizing complex 3D flow fields where occlusion is a problem and multiple flow characteristics (vorticity, divergence, etc.) need to be distinguished simultaneously [S2].
*   **When not to use:** When computational resources are extremely limited and real-time performance is critical without GPU acceleration, or when only simple velocity magnitude visualization is sufficient [G].

## Background
The method was introduced by Park, Budge, Linsen, Hamann, and Joy in 2004 [S2]. It was developed to address the limitations of early flow visualization techniques like hedgehog plots, particle tracing, and streamlines, which present sparse, directed information [S2]. While texture-based approaches (like LIC and IBFV) gained popularity due to graphics hardware progress, they suffer from occlusion [S2]. This method combines feature-based extraction with texture-based rendering to provide global information while highlighting specific flow properties [S2]. It was demonstrated on the NVIDIA GeForce FX GPU [S2].

## Sources
[S1] Dense Geometric Flow Visualization, https://doi.org/10.2312/vissym/eurovis05/021-028, fetched 2026-10-26
[S2] Multi-dimensional Transfer Functions for Interactive 3D Flow Visualization, https://web.cs.ucdavis.edu/~hamann/ParkBudgeLinsenHamannJoy2004.pdf, fetched 2026-10-27
