# Eigenvalue-Driven Texture Parameterization for LIC

Also called: LIC with eigenvalue-dependent parameters, Metric-based LIC parameterization, Metric-based Texture Parameter Mapping for Tensor Field Visualization, Physical tensor visualization via metric distortion, Tensor field LIC with eigenvalue scaling

## Overview
This method visualizes symmetric second-order tensor fields (e.g., stress, strain) by interpreting the tensor as a distortion of a flat metric, mapping eigenvalues to Line Integral Convolution (LIC) texture parameters [S1, S2]. Unlike standard LIC which uses constant parameters, this approach varies density, spot size, and convolution length based on eigenvalues to encode physical compression and expansion, providing an intuitive representation similar to stress/strain fields [S2]. It builds on [Tensor-to-Metric Transformation](10784-tensor-to-metric-transformation.md) by defining specific parameter assignments for 2D and 3D eigenvector fields [S2].

## How it works
**Inputs:** A symmetric second-order tensor field $T(\mathbf{x})$ defined on a grid.
**Outputs:** A textured image or volume where texture properties (density, spot size, convolution length, color) vary according to the local eigenvalues of $T$.

**Data Structures:**
The tensor field is stored as a grid of symmetric $3 \times 3$ matrices (3D) or $2 \times 2$ matrices (2D). For each point, the eigenvalues $\lambda_1, \lambda_2$ (and $\lambda_3$ for 3D) and corresponding eigenvectors are computed. The eigenvectors define the integral line directions for LIC.

**Steps:**
1.  **Eigen Decomposition:** At each grid point, calculate the eigenvalues $\lambda_i$ and eigenvectors $\mathbf{v}_i$ of the tensor field [S2].
2.  **Base LIC Generation:** Generate an initial LIC image using constant density, spot size, color intensity, and convolution length. This serves as the baseline "pure LIC" image where only eigendirections are illustrated [S2].
3.  **Parameter Mapping:** Adjust the texture parameters for each eigenvector field $i$ based on the eigenvalues using the assignments in Table 1 (2D) or Table 2 (3D) [S2].
    *   **Spot Radius:** The radius $r_{i,j}$ changes proportionally to eigenvalues. For 2D, circular spots are defined by radius $r_i$. The formula is:
        $$r_{i,j} = r_0 \frac{d_{i,j}}{d_i}$$
        where $r_0$ is a base radius, $d_{i,j}$ is a distance parameter, and $d_i$ is a normalization factor [S2].
    *   **Convolution Length:** The filter length $l_i$ is proportional to a function of the eigenvalue, representing stretching/compressing along integral lines:
        $$l_i = l_0 \times F(\bar{\lambda}_i)$$
        where $l_0$ is a base length and $F$ is a mapping function [S2].
    *   **Color Intensity:** To enforce the impression of stretching/compressing along fibers, color intensity $I_i$ is modified:
        $$I_i = I_0 \cdot \frac{1}{\lambda_i}$$
        where $I_0$ is a base intensity [S2].
    *   **Color Mapping:** A continuous color code is applied: Red for smallest negative eigenvalues (compression), White for zero eigenvalues (no change), and Green for positive eigenvalues (expansion/tension) [S1, S2].
4.  **Animation (Optional):** To intensify the impression of expansion or compression, scale a factor $\sigma$ (defined in Equation 19 or 24 of the source) or use a time variable $t$ to animate the parameter changes [S2].

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
    # 1. Compute eigenvalues and eigenvectors for all points
    # For each point x, get eigenvalues [lambda1, lambda2, (lambda3)] and eigenvectors [v1, v2, (v3)]
    eigenvalues, eigenvectors = compute_eigen_decomposition(tensor_field)
    
    # 2. Initialize parameters based on dimensionality
    dim = tensor_field.shape[-1] # 2 or 3
    if dim == 2:
        param_map = get_2d_param_map()
    else:
        param_map = get_3d_param_map()
        
    # 3. Generate base LIC image with constant parameters
    # This creates the "pure LIC" baseline
    base_lic = generate_lic(
        eigenvectors=eigenvectors,
        density=1.0, # Constant
        spot_radius=base_radius,
        convolution_length=base_length,
        color_intensity=base_intensity
    )
    
    # 4. Apply eigenvalue-driven modifications
    result_image = base_lic.copy()
    
    for i in range(dim):
        # Get eigenvalue for current field
        lam = eigenvalues[..., i]
        
        # Get parameter mappings for this eigenvector field i
        # Note: The source tables define specific mappings for i=1, i=2, etc.
        # We assume lambda1, lambda2, lambda3 are ordered.
        
        # Calculate modified parameters
        # Density
        # The source uses d_{i,j} and d_{i,k}. In implementation, these correspond 
        # to density components along orthogonal directions.
        # For simplicity in 2D, we map density to 1/lambda_orthogonal.
        if dim == 2:
            if i == 0: # i=1 in 1-based indexing
                density_factor = 1.0 / eigenvalues[..., 1] # 1/lambda2
                spot_factor = 1.0 / eigenvalues[..., 1] # 1/lambda2
                conv_factor = 1.0 / eigenvalues[..., 0] # 1/lambda1
                intensity_factor = 1.0 / eigenvalues[..., 0] # 1/lambda1
            else: # i=2
                density_factor = 1.0 / eigenvalues[..., 0] # 1/lambda1
                spot_factor = 1.0 / eigenvalues[..., 0] # 1/lambda1
                conv_factor = 1.0 / eigenvalues[..., 1] # 1/lambda2
                intensity_factor = 1.0 / eigenvalues[..., 1] # 1/lambda2
        else: # 3D
            # Mapping based on Table 2
            # i=0 (field 1)
            if i == 0:
                density_factor_j = 1.0 / eigenvalues[..., 1] # 1/lambda2
                density_factor_k = 1.0 / eigenvalues[..., 1] # 1/lambda2
                spot_factor_j = 1.0 / eigenvalues[..., 0] # 1/lambda1
                spot_factor_k = 1.0 / eigenvalues[..., 0] # 1/lambda1
                conv_factor = eigenvalues[..., 0] / eigenvalues[..., 1] # lambda1/lambda2
                intensity_factor = 1.0 / eigenvalues[..., 0] # 1/lambda1
            # i=1 (field 2)
            elif i == 1:
                density_factor_j = eigenvalues[..., 0] / eigenvalues[..., 1] # lambda1/lambda2
                density_factor_k = eigenvalues[..., 2] / eigenvalues[..., 0] # lambda3/lambda1
                spot_factor_j = 1.0 / eigenvalues[..., 1] # 1/lambda2
                spot_factor_k = 1.0 / eigenvalues[..., 2] # 1/lambda3
                conv_factor = eigenvalues[..., 1] / eigenvalues[..., 2] # lambda2/lambda3
                intensity_factor = 1.0 / eigenvalues[..., 1] # 1/lambda2
            # i=2 (field 3)
            else:
                density_factor_j = 1.0 / eigenvalues[..., 0] # 1/lambda1
                density_factor_k = 1.0 / eigenvalues[..., 0] # 1/lambda1
                spot_factor_j = 1.0 / eigenvalues[..., 2] # 1/lambda3
                spot_factor_k = 1.0 / eigenvalues[..., 0] # 1/lambda1
                conv_factor = eigenvalues[..., 2] / eigenvalues[..., 0] # lambda3/lambda1
                intensity_factor = 1.0 / eigenvalues[..., 2] # 1/lambda3

        # Apply spot size modification
        # r_{i,j} = r0 * (d_{i,j} / d_i)
        # Assuming d_i is a normalization constant, we scale the base radius
        current_radius = base_radius * spot_factor
        # Note: The source states "radius of the spots changes proportionally to the eigenvalues"
        # and gives formula r_{i,j} = r0 * d_{i,j} / d_i. 
        # The tables map "spot size" to eigenvalue functions. 
        # We apply the spot factor to the radius.
        
        # Apply convolution length modification
        # l_i = l0 * F(lambda_i)
        # The tables map "convolution length" to eigenvalue functions.
        current_length = base_length * conv_factor
        
        # Apply color intensity modification
        # I_i = I0 * (1 / lambda_i)
        current_intensity = base_intensity * intensity_factor
        
        # Apply density modification
        # The source states density is mapped to 1/lambda.
        current_density = 1.0 * density_factor # Assuming base density 1.0
        
        # Regenerate LIC for this component with new parameters
        # In a full implementation, this would involve integrating along the specific eigenvector field i
        # and combining the results. The source suggests overlaying or combining fields.
        component_lic = generate_lic(
            eigenvectors=eigenvectors[..., i:i+1],
            density=current_density,
            spot_radius=current_radius,
            convolution_length=current_length,
            color_intensity=current_intensity
        )
        
        # Combine component into result
        # The source shows images of single fields and overlaid fields.
        # A simple combination is additive or weighted blending.
        result_image = combine_lic_components(result_image, component_lic, i)

    # 5. Apply Color Mapping for Compression/Expansion
    # Red: negative eigenvalues (compression), White: zero, Green: positive (expansion)
    # This is applied to the final result or per component before combination.
    # The source says "apply a continuous color mapping".
    result_image = apply_color_mapping(result_image, eigenvalues)
    
    return result_image

def get_2d_param_map():
    return {
        0: {'density': '1/lambda2', 'intensity': '1/lambda1', 'conv': '1/lambda1', 'spot': '1/lambda2'},
        1: {'density': '1/lambda1', 'intensity': '1/lambda2', 'conv': '1/lambda2', 'spot': '1/lambda1'}
    }

def get_3d_param_map():
    return {
        0: {'density_j': '1/lambda2', 'density_k': '1/lambda2', 'intensity': '1/lambda1', 'conv': 'lambda1/lambda2', 'spot_j': '1/lambda1', 'spot_k': '1/lambda1'},
        1: {'density_j': 'lambda1/lambda2', 'density_k': 'lambda3/lambda1', 'intensity': '1/lambda2', 'conv': 'lambda2/lambda3', 'spot_j': '1/lambda2', 'spot_k': '1/lambda3'},
        2: {'density_j': '1/lambda1', 'density_k': '1/lambda1', 'intensity': '1/lambda3', 'conv': 'lambda3/lambda1', 'spot_j': '1/lambda3', 'spot_k': '1/lambda1'}
    }

def combine_lic_components(base, component, index):
    # Implementation detail: How to combine multiple LIC fields.
    # The source does not specify the exact blending function.
    # A common approach is additive blending or max intensity.
    # [G] Assuming additive blending for visualization of multiple fields.
    return base + component

def apply_color_mapping(image, eigenvalues):
    # Map eigenvalues to colors: Red (neg), White (0), Green (pos)
    # [G] Standard RGB mapping based on sign and magnitude of eigenvalues.
    # This is a post-processing step on the final image.
    pass
```

**Edge Cases and Pitfalls:**
*   **Boundary Discontinuities:** The tensor field resulting from finite element simulations is continuous inside cells but not on cell boundaries. This discontinuity can be observed in the resulting images [S2].
*   **Zero Eigenvalues:** The formulas involve division by eigenvalues ($1/\lambda_i$). Zero eigenvalues must be handled to avoid division by zero, likely by clamping or using a small epsilon [G].
*   **Resolution:** Low resolution of the simulation data can cause fluctuation in color [S2].

## Operating characteristics
**Time and Memory Complexity:**
The method requires eigen decomposition at each point, which is $O(1)$ per point for $2 \times 2$ or $3 \times 3$ matrices. LIC generation is typically $O(N \cdot L)$ where $N$ is the number of pixels/voxels and $L$ is the integration length. The parameter mapping is $O(N)$. Memory usage is dominated by storing the tensor field and the intermediate LIC images.

**Measured Performance:**
The method was evaluated on synthetic datasets and real data from finite element simulations of stress fields [S2]. The simulation dataset used a $10 \times 10 \times 10$ grid, with each cell consisting of $3^3 = 27$ Gauss-points [S2].

**Strengths and Limitations:**
*   **Strengths:** Intuitive visualization of compression and expansion; flexible representation based on physical metric distortion; easy to understand [S1, S2].
*   **Limitations:** Discontinuities at cell boundaries in FEM data; potential for visual clutter when overlaying multiple eigenvector fields; requires careful parameter tuning [S2].

**When to Use:**
Use this method when the goal is to visualize the physical effect of stress/strain in symmetric tensor fields, particularly when an intuitive understanding of compression and expansion is required [S1, S2].

**When Not to Use:**
Not suitable for non-symmetric tensor fields or when the primary goal is to visualize vector flow without tensor distortion information.

## Background
**Origin:**
The method was proposed by Hotz, Feng, Hagen, Hamann, Joy, and Jeremic in 2004 [S1, S2]. It is part of the "Physically Based Methods for Tensor Field Visualization" research.

**History and Development:**
The work builds on the interpretation of tensor fields as distortions of a flat metric. It extends standard LIC by incorporating eigenvalue-dependent parameters. The authors evaluated the method on 2D datasets and 2D slices of 3D tensor fields, laying the foundation for full 3D volume rendering extensions [S1, S2].

**Usage:**
Used in scientific visualization for stress and strain analysis in engineering and physics simulations [S2].

## Sources
[S1] Physically Based Methods for Tensor Field Visualization, https://sokocalo.engr.ucdavis.edu/~jeremic/VisResearch/tensorVis04.pdf, fetched 2026-10-28
[S2] Physically Based Methods for Tensor Field Visualization, https://web.cs.ucdavis.edu/~hamann/HotzFengHagenHamannJoyJeremic2004.pdf, fetched 2026-10-28
