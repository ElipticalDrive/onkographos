# Automatic covariance and rotation recalculation

Also called: Automatic parameter update for transformed Gaussians, Covariance update under mesh transformation

## Overview
This method updates the parameters of 3D Gaussians when the underlying mesh vertices are transformed, ensuring that splats correctly reflect the new geometry without requiring manual recalculation of scaling factors [S1]. It sets itself apart from vanilla Gaussian Splatting modifications, which often require explicit matrix updates that can introduce holes or blurriness, by moving vertices directly in $\mathbb{R}^3$ and automatically deriving the new covariance and rotation parameters [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Original Gaussian parameters (rotation matrix $R$, scaling matrix $S$), and a vertex transformation matrix $B \in SO(3)$ representing the movement or rotation of the mesh vertex in $\mathbb{R}^3$ [S1].
*   **Output:** Updated rotation matrix $R'$ and updated covariance matrix $\Sigma'$ [S1].

**Data Structures:**
*   $R$: $3 \times 3$ rotation matrix.
*   $S$: $3 \times 3$ diagonal scaling matrix.
*   $B$: $3 \times 3$ transformation matrix in $SO(3)$.
*   $\Sigma$: $3 \times 3$ symmetric positive-definite covariance matrix.

**Steps:**
1.  Calculate the new rotation matrix $R'$ by substituting the original rotation $R$ with the transformation $B$:
    $$R' = B R$$
    [S1]
2.  Calculate the new covariance matrix $\Sigma'$ using the new rotation $R'$ and the original scaling $S$:
    $$\Sigma' = R' S S^T R'^T$$
    [S1]
3.  Substitute $R'$ into the covariance equation to express $\Sigma'$ in terms of the original parameters and the transformation $B$:
    $$\Sigma' = (B R) S S^T (B R)^T = B R S S^T R^T B^T$$
    [S1]

**Formulas:**
*   Original Covariance: $\Sigma = R S S^T R^T$ [S1]
*   Updated Covariance: $\Sigma' = B R S S^T R^T B^T$ [S1]

**Parameters:**
*   $B$: The transformation matrix applied to the mesh vertex. Must be in $SO(3)$ (rotation) or include translation if applied to means separately. [S1]

**Edge Cases and Pitfalls:**
*   **Manual Scaling Errors:** Manually updating scaling factors via $S' = AS$ (where $A$ is a diagonal scaling matrix) is error-prone because exact scaling factors are not always apparent for intended modifications, especially when scales differ per Gaussian [S1].
*   **Shear:** If the transformation $B$ introduces non-rigid warping or shear, the resulting covariance matrix may not correspond to a valid rotation and scaling decomposition directly compatible with standard rendering pipelines without further processing (see *Covariance Re-fitting*). [S2, unofficial]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def update_covariance(R, S, B):
    """
    Updates Gaussian covariance and rotation after vertex transformation.
    
    Args:
        R (np.ndarray): Original 3x3 rotation matrix.
        S (np.ndarray): Original 3x3 diagonal scaling matrix.
        B (np.ndarray): 3x3 transformation matrix in SO(3).
        
    Returns:
        R_prime (np.ndarray): New rotation matrix.
        Sigma_prime (np.ndarray): New covariance matrix.
    """
    # Step 1: Calculate new rotation
    R_prime = B @ R
    
    # Step 2: Calculate new covariance
    # Sigma = R * S * S^T * R^T
    # Sigma_prime = R_prime * S * S^T * R_prime^T
    # Equivalent to: B * R * S * S^T * R^T * B^T
    
    # Compute S * S^T (since S is diagonal, this is just element-wise square)
    S_S_T = S @ S.T
    
    # Compute original covariance part: R * S * S^T * R^T
    # This is equivalent to the original Sigma if needed, but we compute directly
    Sigma_prime = B @ (R @ S_S_T @ R.T) @ B.T
    
    return R_prime, Sigma_prime
```

## Subtopics

### Covariance Re-fitting
A post-processing step applied to the transformed covariance matrix to recover rotation and scaling parameters compatible with 3DGS rendering, handling potential shear introduced by deformation [S2, unofficial]. It is chosen when the transformation $B$ is non-rigid (e.g., cage-based deformation) and introduces shear, which standard 3DGS rendering cannot handle directly [S2, unofficial].
1.  Perform eigendecomposition on the deformed covariance matrix $\Sigma_d$ to obtain the orthogonal matrix of eigenvectors $R_d$ and the diagonal matrix of eigenvalues $\Lambda_d$, such that $\Sigma_d = R_d \Lambda_d R_d^T$ [S2, unofficial].
2.  Set the new rotation parameter to the orthogonal matrix of eigenvectors $R_d$ [S2, unofficial].
3.  Compute the new scaling parameters from the square roots of the eigenvalues in $\Lambda_d$ [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per Gaussian, as it involves fixed-size matrix multiplications ($3 \times 3$).
*   **Memory Complexity:** $O(1)$ per Gaussian for storing the updated parameters.
*   **Strengths:** Eliminates the need for manual scaling factor calculation, reducing user error and visual artifacts like holes or blurriness associated with naive updates [S1]. Allows direct manipulation of mesh vertices in $\mathbb{R}^3$ [S1].
*   **Limitations:** If the transformation is non-rigid, the resulting covariance may require re-fitting (see *Covariance Re-fitting*) to remain compatible with the rendering pipeline [S2, unofficial].
*   **When to use:** When modifying the geometry of a mesh represented by Gaussian splats and automatic parameter updates are desired [S1].
*   **When not to use:** When the transformation is complex and introduces significant shear, requiring additional re-fitting steps [S2, unofficial].

## Background
This method originates from the GaMeS (Mesh-based adapting and modification of Gaussian Splatting) paper, which proposes moving vertices directly in $\mathbb{R}^3$ to update Gaussian parameters automatically [S1]. It addresses limitations in vanilla Gaussian Splatting where explicit matrix updates can lead to visual artifacts [S1]. The related *Covariance Re-fitting* technique is described in GSDeformer, which handles non-rigid cage-based deformations [S2, unofficial].

## Sources
[S1] GaMeS: Mesh-based adapting and modification of Gaussian Splatting, https://doi.org/10.1016/j.patcog.2026.114561, fetched 2026-10-29
[S2] GSDeformer: Direct Cage-based Deformation for 3D Gaussian Splatting, https://doi.org/10.48550/arXiv.2405.15491, fetched 2026-10-30 (unofficial)
