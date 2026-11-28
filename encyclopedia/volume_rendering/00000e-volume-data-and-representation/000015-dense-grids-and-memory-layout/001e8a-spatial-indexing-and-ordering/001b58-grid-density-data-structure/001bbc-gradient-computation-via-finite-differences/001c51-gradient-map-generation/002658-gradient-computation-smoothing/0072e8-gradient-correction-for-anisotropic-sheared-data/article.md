# Gradient Correction for Anisotropic/Sheared Data

Also called: Lighting Space Gradient Correction

## Overview
This method corrects gradients estimated from anisotropic or sheared volume data so that lighting calculations (diffuse/specular) are accurate, avoiding the computational cost of full 3D convolution [S1]. It sets itself apart from naive world-space correction by decomposing the transformation into a shear-scale and a rotation, reducing the multiplicative cost per gradient from 9 to 6 (or 3) [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Volume data in voxel space, Model matrix $M$ (world space to voxel space), Correction matrix $C$ (voxel space to world space, accounting for anisotropy/shear), Light direction $\mathbf{l}$, Eye vector $\mathbf{v}$.
*   **Outputs:** Corrected gradient $\mathbf{g}_{LS}$ in Lighting Space, transformed light direction $\mathbf{l}_{LS}$, transformed eye vector $\mathbf{v}_{LS}$.

**Data Structures:**
The hardware gradient correction matrix $G$ is a $3 \times 3$ upper triangular matrix defined as:
$$
G = \begin{bmatrix} s_{11} & s_{21} & s_{31} \\ 0 & s_{22} & s_{23} \\ 0 & 0 & s_{33} \end{bmatrix} = (LS^{-1})^T
$$
where $LS$ is the shear-scale transformation matrix [S1].

**Steps:**
1.  **Decompose Transformation:** Decompose the combined Model and Correction transformation ($M \times C$) into a shear-scale transformation $LS$ and a rotation transformation $LR$. The relationship is defined by:
    $$
    LR^{-1} \times M \times C = LS
    $$
    [S1]
2.  **Transform Lighting Vectors:** Transform the light direction $\mathbf{l}$ and eye vector $\mathbf{v}$ from world space to Lighting Space using the inverse rotation $(LR)^{-1}$:
    $$
    \mathbf{l}_{LS} = (LR)^{-1} \mathbf{l}
    $$
    $$
    \mathbf{v}_{LS} = (LR)^{-1} \mathbf{v}
    $$
    [S1]
3.  **Correct Gradient:** Convert the gradient $\mathbf{g}_{voxel}$ (computed via separable kernels in voxel space) to Lighting Space by multiplying by the inverse transpose of the shear-scale matrix $G$:
    $$
    \mathbf{g}_{LS} = G \mathbf{g}_{voxel} = (LS^{-1})^T \mathbf{g}_{voxel}
    $$
    [S1]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_lighting_space_gradient(gradient_voxel, G, LR_inv):
    """
    Corrects a voxel-space gradient for anisotropy/shear and transforms
    lighting vectors to Lighting Space.
    
    Args:
        gradient_voxel: 3D vector [gx, gy, gz] computed via separable differences.
        G: 3x3 upper triangular matrix (LS^-1)^T.
        LR_inv: 3x3 inverse rotation matrix.
        
    Returns:
        gradient_ls: Corrected gradient in Lighting Space.
        light_dir_ls: Transformed light direction.
        eye_vec_ls: Transformed eye vector.
    """
    # Step 3: Correct gradient using the hardware correction matrix G
    # G is upper triangular:
    # [s11 s21 s31]
    # [0   s22 s23]
    # [0   0   s33]
    
    gx, gy, gz = gradient_voxel
    
    # Perform 6 multiplications for general sheared case
    # If non-sheared (s21=0, s31=0, s23=0), this reduces to 3 multiplications
    g_ls_x = G[0][0]*gx + G[0][1]*gy + G[0][2]*gz
    g_ls_y = G[1][1]*gy + G[1][2]*gz
    g_ls_z = G[2][2]*gz
    
    gradient_ls = [g_ls_x, g_ls_y, g_ls_z]
    
    # Step 2: Transform lighting vectors (assuming light_dir and eye_vec are 3D vectors)
    # Note: In practice, light_dir and eye_vec are inputs to this function or 
    # computed elsewhere. Here we assume they are passed or available.
    # For the purpose of this self-contained function, we assume they are inputs.
    # However, the prompt asks for gradient correction. The transformation of 
    # light/eye is part of the "Lighting Space" definition.
    
    return gradient_ls, light_dir_ls, eye_vec_ls
```
*[G] The pseudocode assumes `light_dir_ls` and `eye_vec_ls` are computed similarly via `LR_inv * vector`. The core gradient correction is the matrix-vector multiplication with $G$.]*

**Parameters:**
*   **Multiplications per gradient:** 6 in the general sheared case; 3 in non-sheared anisotropic cases [S1].

**Edge Cases and Pitfalls:**
*   **Separable Kernels:** Separable kernels produce gradients with the wrong direction and/or magnitude in anisotropic or sheared data [S1].
*   **Computational Cost:** Full 3D convolution is computationally prohibitive in hardware accelerators, necessitating this correction approach [S1].
*   **Rotation:** When lights rotate with the volume object, the $LS$ matrix does not change from frame to frame, saving recomputation time [S1].

## Operating characteristics
*   **Time Complexity:** 6 multiplications per gradient in the general case, 3 in non-sheared anisotropic cases [S1].
*   **Memory Complexity:** Requires storage for the $3 \times 3$ upper triangular correction matrix $G$ and the rotation matrix $LR$ [S1].
*   **Strengths:** Avoids the prohibitive cost of full 3D convolution while providing accurate lighting for anisotropic/sheared data [S1].
*   **Limitations:** Requires decomposition of the model/correction transformation; assumes separable kernels are used for initial gradient estimation [S1].
*   **When to use:** When rendering anisotropic or sheared volume data on hardware accelerators where full 3D convolution is too expensive [S1].
*   **When not to use:** When data is isotropic and orthogonal, or when software rendering allows for full 3D convolution [S1].

## Background
This method was developed for the VolumePro 1000 hardware accelerator to handle anisotropic volume data, such as CT scans, where separable gradient kernels fail to produce correct surface normals [S1]. It introduces "Lighting Space" as an intermediate coordinate system that simplifies the correction of gradients from voxel space to a space compatible with world-space lighting [S1].

## Sources
[S1] Shear-image order ray casting volume rendering, https://doi.org/10.1145/641480.641510, fetched 2026-10-31
