# Apparent Diffusion Coefficient (ADC)


## Overview
The Apparent Diffusion Coefficient (ADC) is a scalar metric that quantifies the magnitude of water diffusion along a specific direction vector $\mathbf{v}$ within a voxel, derived directly from the underlying diffusion tensor $\mathbf{D}$ [S1, unofficial]. Unlike rotationally invariant metrics such as Mean Diffusivity (MD) or Fractional Anisotropy (FA), which summarize the entire tensor, ADC is direction-dependent, allowing for the visualization of anisotropic diffusion pathways [S1, unofficial]. It serves as a fundamental building block for directional visualization techniques, such as Line Integral Convolution (LIC) on tensor fields, by providing a scalar weight for diffusion intensity along streamlines [S1, unofficial].

## How it works
**Inputs:**
1.  $\mathbf{D}$: A $3 \times 3$ symmetric positive-definite diffusion tensor at a specific voxel location.
2.  $\mathbf{v}$: A normalized direction vector ($\|\mathbf{v}\| = 1$) along which the diffusion is to be measured.

**Outputs:**
1.  $a_{\mathbf{v}}$: A scalar value representing the Apparent Diffusion Coefficient along $\mathbf{v}$.

**Data Structures:**
The diffusion tensor $\mathbf{D}$ is typically stored as a $3 \times 3$ matrix. Due to symmetry, only 6 unique values are required, but the formula operates on the full matrix representation:
$$
\mathbf{D} = \begin{bmatrix}
D_{xx} & D_{xy} & D_{xz} \\
D_{yx} & D_{yy} & D_{yz} \\
D_{zx} & D_{zy} & D_{zz}
\end{bmatrix}
$$
where $D_{ij} = D_{ji}$. The direction vector $\mathbf{v}$ is a 3-element vector:
$$
\mathbf{v} = \begin{bmatrix}
v_x \\
v_y \\
v_z
\end{bmatrix}
$$

**Steps:**
1.  Ensure the direction vector $\mathbf{v}$ is normalized. If not, normalize it: $\mathbf{v} \leftarrow \frac{\mathbf{v}}{\|\mathbf{v}\|}$.
2.  Compute the quadratic form $a_{\mathbf{v}} = \mathbf{v}^T \mathbf{D} \mathbf{v}$.

**Formula:**
The ADC along direction $\mathbf{v}$ is defined as:
$$
a_{\mathbf{v}} = \mathbf{v}^T \mathbf{D} \mathbf{v}
$$
Expanding this into component form:
$$
a_{\mathbf{v}} = D_{xx}v_x^2 + D_{yy}v_y^2 + D_{zz}v_z^2 + 2D_{xy}v_xv_y + 2D_{xz}v_xv_z + 2D_{yz}v_yv_z
$$

**Parameters:**
*   $\mathbf{v}$: A normalized direction vector.
*   $\mathbf{D}$: The diffusion tensor.

**Edge Cases and Pitfalls:**
*   **Critical Points:** If the diffusion tensor $\mathbf{D}$ has two equal eigenvalues (a critical point), the eigenvectors are not uniquely defined [S1, unofficial]. While the ADC calculation itself remains numerically stable, the choice of $\mathbf{v}$ becomes sensitive to noise if $\mathbf{v}$ is derived from eigenvectors at these points.
*   **Normalization:** The formula assumes $\mathbf{v}$ is a unit vector. If $\mathbf{v}$ is not normalized, the result scales with the square of the vector's magnitude, leading to incorrect physical interpretations.
*   **Tensor Validity:** The tensor $\mathbf{D}$ must be positive semi-definite. In practice, noise in DTI acquisition can lead to non-positive-definite tensors, which may result in negative ADC values for certain directions. Clamping or tensor regularization may be required before computation [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_adc(tensor_D, direction_v):
    """
    Computes the Apparent Diffusion Coefficient (ADC) along a given direction.
    
    Args:
        tensor_D: 3x3 symmetric matrix representing the diffusion tensor.
                  Stored as a list of lists or 2D array: [[Dxx, Dxy, Dxz], [Dyx, Dyy, Dyz], [Dzx, Dzy, Dzz]]
        direction_v: 3-element list or array representing the direction vector [vx, vy, vz].
        
    Returns:
        float: The ADC value along the direction vector.
    """
    # Extract tensor components
    Dxx, Dxy, Dxz = tensor_D[0]
    Dyx, Dyy, Dyz = tensor_D[1]
    Dzx, Dzy, Dzz = tensor_D[2]
    
    # Ensure symmetry (Dxy == Dyx, etc.)
    # In practice, Dxy should equal Dyx, etc.
    
    # Extract direction components
    vx, vy, vz = direction_v
    
    # Normalize direction vector
    norm_v = (vx**2 + vy**2 + vz**2)**0.5
    if norm_v == 0:
        return 0.0 # Or raise error
    vx, vy, vz = vx/norm_v, vy/norm_v, vz/norm_v
    
    # Compute quadratic form: v^T D v
    # Expanded: Dxx*vx^2 + Dyy*vy^2 + Dzz*vz^2 + 2*Dxy*vx*vy + 2*Dxz*vx*vz + 2*Dyz*vy*vz
    adc = (Dxx * vx**2 + 
           Dyy * vy**2 + 
           Dzz * vz**2 + 
           2 * Dxy * vx * vy + 
           2 * Dxz * vx * vz + 
           2 * Dyz * vy * vz)
           
    return adc
```

## Subtopics

### [Fractional Anisotropy (FA)](12299-fractional-anisotropy-fa.md)
Fractional Anisotropy (FA) is a scalar metric that quantifies the degree of anisotropy in a voxel based solely on the eigenvalues of the diffusion tensor. It solves the problem of distinguishing directional diffusion structures (such as white matter tracts) from isotropic environments (such as cerebrospinal fluid) where Mean Diffusivity (MD) alone is insufficient. Unlike higher-order anisotropy measures, FA is rotationally invariant but ignores higher-order tensor components, making it sensitive to noise when only a single tensor is available.

## Operating characteristics
*   **Time Complexity:** $O(1)$ per voxel per direction, as it involves a fixed number of arithmetic operations (6 multiplications and 5 additions for the expanded form).
*   **Memory Complexity:** $O(1)$ auxiliary memory, requiring only storage for the direction vector and the result.
*   **Strengths:** Computationally inexpensive; provides direction-specific diffusion information; essential for directional visualization techniques like LIC.
*   **Limitations:** Sensitive to the choice of direction vector $\mathbf{v}$; does not provide a single scalar summary of the entire tensor (unlike MD or FA); requires a valid direction vector, which may be ambiguous at critical points (equal eigenvalues).
*   **When to use:** When visualizing diffusion along specific streamlines or when computing directional weights for rendering techniques.
*   **When not to use:** When a single scalar summary of diffusion magnitude regardless of direction is needed (use MD); when quantifying the degree of anisotropy (use FA).

## Background
The Apparent Diffusion Coefficient (ADC) is a fundamental concept in Diffusion Tensor Imaging (DTI). The term "Apparent" distinguishes it from the intrinsic diffusion coefficient, as it accounts for restrictions and barriers to diffusion in biological tissues [S1, unofficial]. The quadratic form $\mathbf{v}^T \mathbf{D} \mathbf{v}$ is a direct consequence of interpreting the diffusion tensor as an ellipsoid, where the ADC represents the squared length of the ellipsoid's radius in the direction $\mathbf{v}$ [S1, unofficial]. Basser et al. [BP96] introduced several rotationally invariant anisotropy metrics, and the ADC serves as the foundational directional quantity from which these invariants are derived [S1, unofficial].

## Sources
[S1] Image-Space Tensor Field Visualization Using a LIC-like Method, https://doi.org/10.1007/978-3-642-21608-4_11, fetched 2026-10-27 (unofficial)
