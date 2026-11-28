# Eigenvalue-Driven Texture Parameterization for LIC

Also called: LIC with eigenvalue-dependent parameters, Metric-based LIC parameterization, Metric-based Texture Parameter Mapping for Tensor Field Visualization, Physical tensor visualization via metric distortion, Tensor field LIC with eigenvalue scaling

## Overview
This method visualizes symmetric second-order tensor fields (e.g., stress, strain) by interpreting the tensor as a distortion of a flat metric, mapping eigenvalues to Line Integral Convolution (LIC) texture parameters [S1, S2]. Unlike standard LIC which uses constant parameters, this approach varies density, spot size, and convolution length based on eigenvalues to encode physical compression and expansion, providing an intuitive representation similar to stress/strain fields [S2]. It builds on [Tensor-to-Metric Transformation](10784-tensor-to-metric-transformation.md) by defining specific parameter assignments for 2D and 3D eigenvector fields [S2].

## How it works
**Inputs:** A symmetric second-order tensor field $T(\mathbf{x})$ defined on a grid [S2].
**Outputs:** A textured image or volume where texture properties (density, spot size, convolution length, color) vary according to the local eigenvalues of $T$ [S2].

**Data Structures:**
The tensor field is stored as a grid of symmetric $3 \times 3$ matrices (3D) or $2 \times 2$ matrices (2D) [S2]. For each point, the eigenvalues $\lambda_1, \lambda_2$ (and $\lambda_3$ for 3D) and corresponding eigenvectors are computed [S2]. The eigenvectors define the integral line directions for LIC [S2].

**Steps:**
1.  **Eigen Decomposition:** At each grid point, calculate the eigenvalues $\lambda_i$ and eigenvectors $\mathbf{v}_i$ of the tensor field [S2].
2.  **Base LIC Generation:** Generate an initial LIC image using constant density, spot size, color intensity, and convolution length [S2].
3.  **Parameter Mapping:** Adjust the texture parameters for each eigenvector field $i$ based on the eigenvalues using the assignments in Table 1 (2D) or Table 2 (3D) [S2].
    *   **Spot Radius:** The radius $r_{i,j}$ changes proportionally to eigenvalues [S2]. For 2D, circular spots are defined by radius $r_i$ [S2]. The formula is:
        $$r_{i,j} = r_0 \frac{d_{i,j}}{d_i}$$
        where $r_0$ is a base radius, $d_{i,j}$ is a distance parameter, and $d_i$ is a normalization factor [S2].
    *   **Convolution Length:** The filter length $l_i$ is proportional to a function of the eigenvalue, representing stretching/compressing along integral lines [S2]. The formula is:
        $$l_i = l_0 \times F(\bar{\lambda}_i)$$
        where $l_0$ is a base length and $F$ is a mapping function [S2].
    *   **Color Intensity:** To enforce the impression of stretching/compressing along fibers, color intensity $I_i$ is modified [S2]. The formula is:
        $$I_i = I_0 \cdot \frac{1}{\lambda_i}$$
        where $I_0$ is a base intensity [S2].
    *   **Color Mapping:** A continuous color code is applied: Red for smallest negative eigenvalues (compression), White for zero eigenvalues (no change), and Green for positive eigenvalues (expansion/tension) [S1, S2].
4.  **Animation (Optional):** To intensify the impression of expansion or compression, scale a factor $\sigma$ or use a time variable $t$ to animate the parameter changes [S2].

**Parameter Assignments:**
The following tables define the mapping of eigenvalues to texture parameters for 2D and 3D fields [S2].

**Table 1: 2D Parameter Assignments**

| Eigenvector Field | Density $d_{i,j}$ | Color Intensity $I_i$ | Convolution Length $l_i$ | Spot Size $r_{i,j}$ |
| :--- | :--- | :--- | :--- | :--- |
| $i=1$ | $1/\lambda_2$ | $1/\lambda_1$ | $1/\lambda_1$ | $1/\lambda_2$ |
| $i=2$ | $1/\lambda_1$ | $1/\lambda_2$ | $1/\lambda_2$ | $1/\lambda_1$ |

**Table 2: 3D Parameter Assignments**

| Eigenvector Field | Density $d_{i,j}$ | Density $d_{i,k}$ | Color Intensity $I_i$ | Convolution Length $l_i$ | Spot Size $r_{i,j}$ | Spot Size $r_{i,k}$ |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| $i=1$ | $1/\lambda_2$ | $1/\lambda_2$ | $1/\lambda_1$ | $\lambda_1/\lambda_2$ | $1/\lambda_1$ | $1/\lambda_1$ |
| $i=2$ | $\lambda_1/\lambda_2$ | $\lambda_3/\lambda_1$ | $1/\lambda_2$ | $\lambda_2/\lambda_3$ | $1/\lambda_2$ | $1/\lambda_3$ |
| $i=3$ | $1/\lambda_1$ | $1/\lambda_1$ | $1/\lambda_3$ | $\lambda_3/\lambda_1$ | $1/\lambda_3$ | $1/\lambda_1$ |

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def eigenvalue_driven_lic(tensor_field, base_radius, base_length, base_intensity):
    """
    Generates an LIC visualization where parameters are driven by tensor eigenvalues.
    
    Args:
        tensor_field: 3D array of symmetric tensors (or 2D for 2D case).
        base_radius: Initial spot radius r0.
        base_length: Initial convolution length l0.
        base_intensity: Initial color intensity I0.
        
    Returns:
        result_image: The final textured image.
    """
    dim = tensor_field.shape[-1]
    eigenvalues, eigenvectors = compute_symmetric_eigen_decomposition(tensor_field)
    # Standard LIC generation with constant parameters [G]
    base_lic = standard_lic(eigenvectors, density=1.0, spot_radius=base_radius,
                            convolution_length=base_length, color_intensity=base_intensity)
    result_image = base_lic.copy()
    for i in range(dim):
        lam = eigenvalues[..., i]
        if dim == 2:
            if i == 0:
                density_factor = 1.0 / eigenvalues[..., 1]
                spot_factor = 1.0 / eigenvalues[..., 1]
                conv_factor = 1.0 / eigenvalues[..., 0]
                intensity_factor = 1.0 / eigenvalues[..., 0]
            else:
                density_factor = 1.0 / eigenvalues[..., 0]
                spot_factor = 1.0 / eigenvalues[..., 0]
                conv_factor = 1.0 / eigenvalues[..., 1]
                intensity_factor = 1.0 / eigenvalues[..., 1]
        else:
            if i == 0:
                density_factor_j = 1.0 / eigenvalues[..., 1]
                density_factor_k = 1.0 / eigenvalues[..., 1]
                spot_factor_j = 1.0 / eigenvalues[..., 0]
                spot_factor_k = 1.0 / eigenvalues[..., 0]
                conv_factor = eigenvalues[..., 0] / eigenvalues[..., 1]
                intensity_factor = 1.0 / eigenvalues[..., 0]
            elif i == 1:
                density_factor_j = eigenvalues[..., 0] / eigenvalues[..., 1]
                density_factor_k = eigenvalues[..., 2] / eigenvalues[..., 0]
                spot_factor_j = 1.0 / eigenvalues[..., 1]
                spot_factor_k = 1.0 / eigenvalues[..., 2]
                conv_factor = eigenvalues[..., 1] / eigenvalues[..., 2]
                intensity_factor = 1.0 / eigenvalues[..., 1]
            else:
                density_factor_j = 1.0 / eigenvalues[..., 0]
                density_factor_k = 1.0 / eigenvalues[..., 0]
                spot_factor_j = 1.0 / eigenvalues[..., 2]
                spot_factor_k = 1.0 / eigenvalues[..., 0]
                conv_factor = eigenvalues[..., 2] / eigenvalues[..., 0]
                intensity_factor = 1.0 / eigenvalues[..., 2]
        current_radius = base_radius * spot_factor
        current_length = base_length * conv_factor
        current_intensity = base_intensity * intensity_factor
        current_density = density_factor
        # Standard LIC generation for single eigenvector field with modified parameters [G]
        component_lic = standard_lic(eigenvectors[..., i], current_density,
                                     current_radius, current_length, current_intensity)
        result_image = result_image + component_lic
    # Map eigenvalues to colors: red for negative, white for zero, green for positive [G]
    result_image = apply_color_mapping(result_image, eigenvalues)
    return result_image

def compute_symmetric_eigen_decomposition(tensor_field):
    # Closed-form eigen decomposition for symmetric 2x2 or 3x3 matrices [G]
    pass

def standard_lic(field, density, spot_radius, convolution_length, color_intensity):
    # Standard Line Integral Convolution: sample noise, integrate along field lines, apply filter [G]
    pass

def apply_color_mapping(image, eigenvalues):
    # Post-processing: assign RGB based on eigenvalue sign/magnitude [G]
    pass
```

**Edge Cases and Pitfalls:**
*   **Boundary Discontinuities:** The tensor field resulting from finite element simulations is continuous inside cells but not on cell boundaries [S2].
*   **Zero Eigenvalues:** The formulas involve division by eigenvalues ($1/\lambda_i$) [S2]. Zero eigenvalues must be handled to avoid division by zero, typically by clamping or adding a small epsilon [G].
*   **Resolution:** Low resolution of the simulation data can cause fluctuation in color [S2].

## Operating characteristics
**Time and Memory Complexity:**
Not given in the sources.

**Measured Performance:**
The method was evaluated on synthetic datasets and real data from finite element simulations of stress fields [S2]. The simulation dataset used a $10 \times 10 \times 10$ grid, with each cell consisting of $3^3 = 27$ Gauss-points [S2].

**Strengths and Limitations:**
*   **Strengths:** Intuitive visualization of compression and expansion; flexible representation based on physical metric distortion; easy to understand [S1, S2].
*   **Limitations:** Discontinuities at cell boundaries in FEM data [S2].

**When to Use:**
Use this method when the goal is to visualize the physical effect of stress/strain in symmetric tensor fields, particularly when an intuitive understanding of compression and expansion is required [S1, S2].

**When Not to Use:**
Not suitable for non-symmetric tensor fields or when the primary goal is to visualize vector flow without tensor distortion information [G].

## Background
**Origin:**
The method was proposed by Hotz, Feng, Hagen, Hamann, Joy, and Jeremic in 2004 [S1, S2].

**History and Development:**
The work builds on the interpretation of tensor fields as distortions of a flat metric [S1, S2]. It extends standard LIC by incorporating eigenvalue-dependent parameters [S2]. The authors evaluated the method on 2D datasets and 2D slices of 3D tensor fields, laying the foundation for full 3D volume rendering extensions [S1, S2].

**Usage:**
Used in scientific visualization for stress and strain analysis in engineering and physics simulations [S2].

## Sources
[S1] Physically Based Methods for Tensor Field Visualization, https://sokocalo.engr.ucdavis.edu/~jeremic/VisResearch/tensorVis04.pdf, fetched 2026-10-28
[S2] Physically Based Methods for Tensor Field Visualization, https://web.cs.ucdavis.edu/~hamann/HotzFengHagenHamannJoyJeremic2004.pdf, fetched 2026-10-28
