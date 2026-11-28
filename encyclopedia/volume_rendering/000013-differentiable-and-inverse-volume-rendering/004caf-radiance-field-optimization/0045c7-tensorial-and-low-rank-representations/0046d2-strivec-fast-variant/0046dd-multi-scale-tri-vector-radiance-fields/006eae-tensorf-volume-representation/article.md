# TensoRF Volume Representation

Also called: Static TensoRF, Tensorial Radiance Fields

## Overview
TensoRF represents a 3D radiance field by factorizing the volume into a sum of outer products between functions of one or more spatial dimensions, significantly reducing memory usage compared to dense voxel grids [S1]. It differs from its parent method, Multi-scale Tri-Vector Radiance Fields, by using a specific low-rank tensor decomposition structure for both density and appearance, enabling efficient optimization of static scenes [S1].

## How it works
**Inputs:**
*   Spatial coordinates $x_k = (x_k, y_k, z_k)$ for sample points along a ray.
*   Ray direction $\vec{\omega}$ for color evaluation.
*   Hyperparameters:
    *   $M$: Output dimension of the vector-valued functions (rank) [S1].
    *   $N_x, N_y, N_z$: Spatial resolution of the discretized arrays [S1].

**Outputs:**
*   Density $\sigma(x_k)$ at each sample point.
*   Spherical harmonic coefficients $A(x_k)$ at each sample point.
*   Final color $L_e(x_k, \vec{\omega})$ computed from coefficients and direction.

**Data Structures:**
The representation consists of six sets of vector-valued functions discretized into arrays. For a single component of the vector output (dimension $M$):
*   $f_1$: Stored as a 2D array of size $N_x \times N_y$.
*   $g_1$: Stored as a 1D array of size $N_z$.
*   $f_2$: Stored as a 2D array of size $N_x \times N_z$.
*   $g_2$: Stored as a 1D array of size $N_y$.
*   $f_3$: Stored as a 2D array of size $N_y \times N_z$.
*   $g_3$: Stored as a 1D array of size $N_x$.
*   $h_1, k_1, h_2, k_2, h_3, k_3$: Same layout as $f/g$ pairs, used for density [S1].
*   $B_1, B_2, B_3$: Linear transforms mapping the element-wise products to the spherical harmonic coefficient space [S1].

**Algorithm:**

1.  **Evaluate Appearance Coefficients:**
    Compute the spherical harmonic coefficients $A(x_k)$ for point $x_k$ using the following factorization [S1]:
    $$
    A(x_k) = B_1(f_1(x_k, y_k) \odot g_1(z_k)) + B_2(f_2(x_k, z_k) \odot g_2(y_k)) + B_3(f_3(y_k, z_k) \odot g_3(x_k))
    $$
    Here, $\odot$ denotes element-wise multiplication. $f_j$ and $g_j$ are vector-valued functions with output dimension $M$. $B_j$ are linear transforms [S1].

2.  **Evaluate Density:**
    Compute the density $\sigma(x_k)$ using a similar factorization [S1]:
    $$
    \sigma(x_k) = \mathbf{1}^\top(h_1(x_k, y_k) \odot k_1(z_k)) + \mathbf{1}^\top(h_2(x_k, z_k) \odot k_2(y_k)) + \mathbf{1}^\top(h_3(y_k, z_k) \odot k_3(x_k))
    $$
    Here, $\mathbf{1}$ is a vector of ones, and $h_j, k_j$ are vector-valued functions with output dimension $M$. The term $\mathbf{1}^\top(\cdot)$ sums the elements of the resulting vector [S1].

3.  **Compute Color:**
    Compute the color $L_e(x_k, \vec{\omega})$ by taking the dot product of the coefficients $A(x_k)$ and the spherical harmonic basis functions evaluated at the ray direction $\vec{\omega}$ [S1].

4.  **Volume Rendering:**
    Use the computed color $L_e(x_k, \vec{\omega})$ and density $\sigma(x_k)$ for all sample points $\{x_k\}$ along a ray to render the final pixel color, typically via standard volume rendering integration (e.g., Eq. 2 in [S1]) [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def evaluate_tensorrf(x, y, z, B1, B2, B3, f1, g1, f2, g2, f3, g3, 
                      h1, k1, h2, k2, h3, k3, sh_basis_func):
    """
    Evaluates TensoRF representation for a batch of points.
    
    Args:
        x, y, z: 1D arrays of coordinates, shape (N,)
        B1, B2, B3: Linear transform matrices, shape (M_sh, M)
        f1: Array, shape (Nx, Ny)
        g1: Array, shape (Nz,)
        ... (other arrays follow similar layout)
        sh_basis_func: Function evaluating spherical harmonics at direction omega.
                       Note: This implementation assumes omega is handled externally
                       or passed if needed. For pure spatial evaluation, we return A.
    
    Returns:
        A: Spherical harmonic coefficients, shape (N, M_sh)
        sigma: Density, shape (N,)
    """
    # Interpolate functions at coordinates
    # f1(x, y) -> shape (N, M) assuming M channels per array element or 
    # M separate arrays. The source says "M different 2D and 1D arrays".
    # We assume f1 is a list of M arrays or a single array with last dim M.
    # For clarity, we assume f1 has shape (Nx, Ny, M).
    
    # Interpolation helper [G]
    def interp_2d(arr_2d, x_coords, y_coords):
        # Bilinear interpolation
        # Returns shape (N, M) if arr_2d is (Nx, Ny, M)
        pass 

    def interp_1d(arr_1d, z_coords):
        # Linear interpolation
        # Returns shape (N, M) if arr_1d is (Nz, M)
        pass

    # Evaluate terms for Appearance
    # Term 1: f1(x,y) * g1(z)
    f1_val = interp_2d(f1, x, y) # (N, M)
    g1_val = interp_1d(g1, z)    # (N, M)
    term1 = f1_val * g1_val      # Element-wise, (N, M)
    
    # Term 2: f2(x,z) * g2(y)
    f2_val = interp_2d(f2, x, z)
    g2_val = interp_1d(g2, y)
    term2 = f2_val * g2_val
    
    # Term 3: f3(y,z) * g3(x)
    f3_val = interp_2d(f3, y, z)
    g3_val = interp_1d(g3, x)
    term3 = f3_val * g3_val
    
    # Sum and apply linear transforms
    # B1, B2, B3 are (M_sh, M)
    A = np.zeros((len(x), B1.shape[0]))
    A += (term1 @ B1.T)
    A += (term2 @ B2.T)
    A += (term3 @ B3.T)
    
    # Evaluate terms for Density
    # Term 1: h1(x,y) * k1(z)
    h1_val = interp_2d(h1, x, y)
    k1_val = interp_1d(k1, z)
    d_term1 = h1_val * k1_val
    sigma1 = np.sum(d_term1, axis=1) # Sum over M
    
    # Term 2: h2(x,z) * k2(y)
    h2_val = interp_2d(h2, x, z)
    k2_val = interp_1d(k2, y)
    d_term2 = h2_val * k2_val
    sigma2 = np.sum(d_term2, axis=1)
    
    # Term 3: h3(y,z) * k3(x)
    h3_val = interp_2d(h3, y, z)
    k3_val = interp_1d(k3, x)
    d_term3 = h3_val * k3_val
    sigma3 = np.sum(d_term3, axis=1)
    
    sigma = sigma1 + sigma2 + sigma3
    
    return A, sigma

def compute_color(A, omega):
    """
    Computes color from SH coefficients and direction.
    """
    # Evaluate SH basis at omega
    sh_basis = sh_basis_func(omega) # Shape (M_sh,)
    # Dot product for each point
    color = A @ sh_basis
    return color
```

## Subtopics

### Keyframe-Based Dynamic Volume Representation
Extends TensoRF to parameterize volumetric snapshots at discrete time steps (keyframes) by making the $g_j$ and $k_j$ functions dependent on time $\tau_i$, in addition to one spatial dimension [S1]. The spatial functions $f_j$ remain static [S1].
*   **Formula:**
    $$
    A(x_k, \tau_i) = B_1(f_1(x_k, y_k) \odot g_1(z_k, \tau_i)) + B_2(f_2(x_k, z_k) \odot g_2(y_k, \tau_i)) + B_3(f_3(y_k, z_k) \odot g_3(x_k, \tau_i))
    $$
    $$
    \sigma(x_k, \tau_i) = \mathbf{1}^\top(h_1(x_k, y_k) \odot k_1(z_k, \tau_i)) + \mathbf{1}^\top(h_2(x_k, z_k) \odot k_2(y_k, \tau_i)) + \mathbf{1}^\top(h_3(y_k, z_k) \odot k_3(x_k, \tau_i))
    $$
*   **Layout:** Memory layout uses spatial resolution $(N_x, N_y, N_z)$ and number of keyframes $N_t$. $f_1$ is stored as $N_x \times N_y$. $g_1$ is stored as $N_z \times N_t$ [S1]. Since $N_t \ll N_x/y/z$, the time-dependent arrays do not significantly increase model size [S1].
*   **Parameters:** For dynamic datasets, every 4th frame is used as a keyframe. Input videos are split into 50-frame chunks for training [S1].

## Operating characteristics
*   **Memory:** The memory footprint is significantly lower than dense voxel grids because it stores low-rank factors. For dynamic scenes, the memory footprint is similar to a static TensoRF for a single frame, assuming the number of keyframes is small relative to spatial resolution [S1].
*   **Performance:** Specific FPS and PSNR numbers are provided in comparisons against other methods (e.g., DoNeRF dataset) in the source, but exact hardware specs for the TensoRF baseline are not detailed in the provided text [S1].
*   **Strengths:** Memory efficiency for static and dynamic scenes; ability to handle high-resolution volumes via factorization [S1].
*   **Limitations:** Requires discretization of functions into arrays; dynamic scenes require keyframe selection and chunking [S1].

## Background
TensoRF (Tensorial Radiance Fields) was introduced as a memory-efficient approach to represent 3D volumes by factorizing them into outer products of functions [S1]. It builds on the concept of low-rank tensor decomposition applied to radiance fields. The dynamic extension, Keyframe-Based Dynamic Volume Representation, was developed to handle 6-DoF video by adapting TensoRF to parameterize volumetric snapshots at discrete time steps [S1].

## Sources
[S1] HyperReel: High-Fidelity 6-DoF Video with Ray-Conditioned Sampling, https://doi.org/10.1109/cvpr52729.2023.01594, fetched 2026-10-26
