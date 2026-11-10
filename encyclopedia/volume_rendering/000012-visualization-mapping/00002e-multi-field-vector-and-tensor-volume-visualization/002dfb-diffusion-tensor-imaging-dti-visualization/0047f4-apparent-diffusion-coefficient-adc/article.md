# Apparent Diffusion Coefficient (ADC)


## Overview
The Apparent Diffusion Coefficient (ADC) is a scalar quantity that describes the magnitude of water diffusion along a specific direction vector $\mathbf{v}$, derived directly from the underlying diffusion tensor $\mathbf{D}$ [S1, unofficial]. It serves as a directional measure of diffusivity, distinct from rotationally invariant metrics like Mean Diffusivity, allowing visualization of diffusion properties aligned with specific anatomical structures [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** A diffusion tensor $\mathbf{D}$ (a $3 \times 3$ symmetric positive-definite matrix) and a direction vector $\mathbf{v}$.
*   **Output:** A scalar value $a_{\mathbf{v}}$ representing the apparent diffusion coefficient along $\mathbf{v}$.

**Data Structures**
The diffusion tensor $\mathbf{D}$ is typically stored as a $3 \times 3$ matrix. Due to symmetry, only 6 unique values are required, but the formula operates on the full matrix representation. The vector $\mathbf{v}$ is a 3-element vector.

**Formula**
The ADC along a normalized direction vector $\mathbf{v}$ is defined as the quadratic form:
$$ a_{\mathbf{v}} = \mathbf{v}^T \mathbf{D} \mathbf{v} $$

**Parameters**
*   $\mathbf{v}$: A normalized direction vector ($\|\mathbf{v}\| = 1$) [S1, unofficial].
*   $\mathbf{D}$: The diffusion tensor at the voxel of interest [S1, unofficial].

**Steps**
1.  Ensure the direction vector $\mathbf{v}$ is normalized. If not, normalize it: $\mathbf{v} \leftarrow \mathbf{v} / \|\mathbf{v}\|$.
2.  Compute the matrix-vector product $\mathbf{u} = \mathbf{D} \mathbf{v}$.
3.  Compute the dot product $a_{\mathbf{v}} = \mathbf{v} \cdot \mathbf{u}$.
4.  Return $a_{\mathbf{v}}$.

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_adc(D, v):
    """
    Compute the Apparent Diffusion Coefficient along direction v.
    
    Args:
        D: 3x3 list of lists or array representing the symmetric diffusion tensor.
        v: 3-element list or array representing the direction vector.
        
    Returns:
        float: The scalar ADC value.
    """
    # Step 1: Normalize v
    norm_v = (v[0]**2 + v[1]**2 + v[2]**2)**0.5
    if norm_v == 0:
        return 0.0 # Pitfall: undefined for zero vector
    v_norm = [v[0]/norm_v, v[1]/norm_v, v[2]/norm_v]
    
    # Step 2: Matrix-vector multiplication u = D * v
    u = [0.0, 0.0, 0.0]
    for i in range(3):
        for j in range(3):
            u[i] += D[i][j] * v_norm[j]
            
    # Step 3: Dot product v . u
    adc = v_norm[0]*u[0] + v_norm[1]*u[1] + v_norm[2]*u[2]
    
    return adc
```

**Edge Cases and Pitfalls**
*   **Zero Vector:** If $\mathbf{v}$ is the zero vector, normalization fails. The result is undefined or treated as 0 [G].
*   **Precision:** Since $\mathbf{D}$ is symmetric, numerical errors in storage or computation should be minimized by enforcing symmetry if necessary, though the formula assumes a valid tensor [G].

## Subtopics

### Fractional Anisotropy (FA)
Fractional Anisotropy (FA) is a scalar measure of the degree of anisotropy in a voxel, derived from the eigenvalues of the diffusion tensor [S1, unofficial]. It is rotationally invariant and ranges from 0 to 1 [S1, unofficial].
*   **Formula:** $FA(\mathbf{D}) = \sqrt{\frac{3}{2}} \sqrt{\frac{\sum_i(\lambda_i - MD(\mathbf{D}))^2}{\sum_i \lambda_i}}$, where $\lambda_i$ are the three eigenvalues of $\mathbf{D}$ and $MD(\mathbf{D}) = \text{trace}(\mathbf{D})/3$ is the Mean Diffusivity [S1, unofficial].
*   **Interpretation:** $FA=0$ represents perfect isotropic diffusion; $FA=1$ represents the hypothetical case of an infinite cylinder [S1, unofficial].
*   **Usage:** FA is used to distinguish tissue types where Mean Diffusivity is insufficient (e.g., white matter vs. gray matter), as high FA often indicates white matter tissue [S1, unofficial].
*   **Pitfall:** FA is calculated using only one tensor, making it prone to noise conditions [S1, unofficial].

## Operating characteristics
*   **Complexity:** The computation is $O(1)$ per voxel per direction, involving a constant number of arithmetic operations (matrix-vector multiplication and dot product).
*   **Memory:** Requires storage of the diffusion tensor $\mathbf{D}$ (6-9 floats) and the direction vector $\mathbf{v}$ (3 floats).
*   **Strengths:** Provides directional information about diffusion, which is crucial for visualizing fiber tracts in DTI.
*   **Limitations:** It is direction-dependent, unlike FA or MD. It does not provide a single scalar summary of the tensor's overall anisotropy but rather its property along a specific axis.

## Background
The Apparent Diffusion Coefficient is a fundamental concept in Diffusion Tensor Imaging (DTI). The term "Apparent" distinguishes it from the intrinsic diffusion coefficient, as it accounts for restrictions and barriers to diffusion in biological tissue [S1, unofficial]. The quadratic form $\mathbf{v}^T \mathbf{D} \mathbf{v}$ is a direct consequence of the tensor nature of diffusion, where $\mathbf{D}$ describes the ellipsoidal shape of the diffusion process [S1, unofficial]. Basser et al. introduced related rotationally invariant metrics like FA and MD, establishing the framework for scalar derived quantities from tensors [S1, unofficial].

## Sources
[S1] Image-Space Tensor Field Visualization Using a LIC-like Method, https://doi.org/10.1007/978-3-642-21608-4_11, fetched 2026-10-27 (unofficial)
