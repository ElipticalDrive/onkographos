# SGGX Parameter Fitting from Directional Transparencies

Also called: Algorithm Aniso, Algorithm Iso, Computing temporary parameters ρ0 and A0, Computing temporary parameters ρ0 and A0 from directional transparencies

## Overview
This method computes the initial microflake parameters—density $\rho_0$ and self-shadowing $A_0$—for a low-resolution voxel such that its directional transparencies match those of the input high-resolution voxel block along the principal axes [S1]. It solves the problem of preserving the average extinction and directional attenuation characteristics when aggregating fine-scale volume data into a coarser representation [S1]. It differs from simple averaging by explicitly inverting the microflake transparency model to derive parameters that satisfy target transparency constraints, rather than approximating them [S1].

## How it works
**Inputs:**
- $L$: The side length (size) of the low-resolution voxel [S1].
- $T(\omega_x), T(\omega_y), T(\omega_z)$: The directional transparencies of the input high-resolution voxel block along the principal axes $X, Y, Z$ [S1]. These are typically computed by ray-marching or integrating the high-resolution density and scattering properties [S1].
- $D(m)$: The microflake normal distribution function (NDF) [S1].

**Outputs:**
- $\rho_0$: The initial density parameter for the low-resolution voxel [S1].
- $A_0(\omega)$: The self-shadowing function parameters (either scalar $A_0$ for Iso or matrix diagonal $s_X, s_Y, s_Z$ for Aniso) [S1].

**Data Structures:**
- The self-shadowing function $A_0(\omega)$ is represented either as a scalar constant $A_0$ (Algorithm Iso) or as a diagonal matrix $\mathbf{S} = \text{diag}(s_X, s_Y, s_Z)$ (Algorithm Aniso) [S1].

**Formulas:**
The transparency of a low-resolution voxel in direction $\omega_i$ is modeled as [S1]:
$$
\exp(-L\sigma_i(\omega_i)) = \exp\left(-L\rho_0 A_0(\omega_i) \int D(m) \langle \omega_i \cdot m \rangle dm\right)
$$
Let $I(\omega_i) = \int D(m) \langle \omega_i \cdot m \rangle dm$ be the integral of the NDF weighted by the dot product with direction $\omega_i$ [S1].

**Algorithm Iso (Simplified Model):**
1. Set $A_0 = 1$ for all directions $\omega$ [S1].
2. Compute $\rho_0$ by averaging the required density across the three principal axes to preserve average transparency [S1]:
   $$
   \rho_0 = \frac{1}{3} \sum_{i \in \{X, Y, Z\}} \frac{-\log(T(\omega_i))}{L \cdot I(\omega_i)} \quad (7.6)
   $$
   [S1]

**Algorithm Aniso (Anisotropic Model):**
1. Compute $\rho_0$ as the maximum density required among the three principal axes to ensure no axis exceeds its target transparency [S1]:
   $$
   \rho_0 = \max_{i \in \{X, Y, Z\}} \left( \frac{-\log(T(\omega_i))}{L \cdot I(\omega_i)} \right) \quad (7.7)
   $$
   [S1]
2. Compute the anisotropic self-shadowing parameters $s_X, s_Y, s_Z$ for each axis $i \in \{X, Y, Z\}$ using [S1]:
   $$
   s_i = \frac{-\log(T(\boldsymbol{\omega}_i))}{\rho_0 L \cdot I(\boldsymbol{\omega}_i)} \quad (7.9)
   $$
   [S1]
3. Define the self-shadowing function as a quadratic form [S1]:
   $$
   A_0(\omega) = \omega^T \begin{bmatrix} s_X & 0 & 0 \\ 0 & s_Y & 0 \\ 0 & 0 & s_Z \end{bmatrix} \omega \quad (7.8)
   $$
   [S1]

**Edge Cases and Pitfalls:**
- The logarithm $\log(T(\omega_i))$ requires $T(\omega_i) > 0$. If a direction is completely opaque ($T=0$), the density becomes infinite. In practice, a small epsilon should be added to $T$ or the transparency clamped [G].
- The integral $I(\omega_i)$ depends on the specific NDF $D(m)$. For the SGGX model, this integral has a closed-form solution or efficient approximation [S1, S2]. If $D(m)$ is not normalized or defined correctly, the derived parameters will be incorrect.
- Algorithm Aniso ensures that the directional transparencies match *exactly* along the principal axes [S1]. However, it does not guarantee exact matching in arbitrary directions, as $A_0(\omega)$ is constrained to be a quadratic form aligned with the axes [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_sggx_parameters_iso(L, T_x, T_y, T_z, NDF_integral_x, NDF_integral_y, NDF_integral_z):
    """
    Computes SGGX parameters using the Iso algorithm.
    
    Args:
        L: Size of the low-res voxel.
        T_x, T_y, T_z: Directional transparencies along X, Y, Z axes.
        NDF_integral_x, NDF_integral_y, NDF_integral_z: Integral of D(m)<omega.m> for each axis.
        
    Returns:
        rho0: Density parameter.
        A0: Self-shadowing parameter (scalar, always 1).
    """
    # Compute required density for each axis
    rho_x = -log(T_x) / (L * NDF_integral_x)
    rho_y = -log(T_y) / (L * NDF_integral_y)
    rho_z = -log(T_z) / (L * NDF_integral_z)
    
    # Average density
    rho0 = (rho_x + rho_y + rho_z) / 3.0
    
    # Iso model sets A0 = 1
    A0 = 1.0
    
    return rho0, A0

def compute_sggx_parameters_aniso(L, T_x, T_y, T_z, NDF_integral_x, NDF_integral_y, NDF_integral_z):
    """
    Computes SGGX parameters using the Aniso algorithm.
    
    Args:
        L: Size of the low-res voxel.
        T_x, T_y, T_z: Directional transparencies along X, Y, Z axes.
        NDF_integral_x, NDF_integral_y, NDF_integral_z: Integral of D(m)<omega.m> for each axis.
        
    Returns:
        rho0: Density parameter.
        s: Diagonal self-shadowing parameters [s_x, s_y, s_z].
    """
    # Compute required density for each axis
    rho_x = -log(T_x) / (L * NDF_integral_x)
    rho_y = -log(T_y) / (L * NDF_integral_y)
    rho_z = -log(T_z) / (L * NDF_integral_z)
    
    # Max density to ensure no axis exceeds target transparency
    rho0 = max(rho_x, rho_y, rho_z)
    
    # Compute self-shadowing parameters
    s_x = -log(T_x) / (rho0 * L * NDF_integral_x)
    s_y = -log(T_y) / (rho0 * L * NDF_integral_y)
    s_z = -log(T_z) / (rho0 * L * NDF_integral_z)
    
    s = [s_x, s_y, s_z]
    
    return rho0, s
```

## Subtopics

### Phase Function Clustering (Local)
This variant extends the parameter fitting by clustering the microflake orientations (SGGX matrices) within the input voxel block to preserve anisotropic scattering behavior, rather than just extinction [S2]. It is chosen when the phase function (scattering directionality) of the high-resolution data is anisotropic and cannot be represented by a single SGGX lobe [S2]. The method identifies the set of input voxel indices $V(i)$ contained in the downsampled voxel $i$, treats each input SGGX matrix $S_{i'}^{orig}$ as a 9D vector, and applies K-means clustering to partition $V(i)$ into $m$ clusters $V(i,1) \dots V(i,m)$ [S2]. For each cluster $j$, the average matrix $S_{i,j}$ is computed, and the corresponding lobe $f_{i,j}$ is set to $f_{SGGX}(S_{i,j})$ [S2]. The final phase function is a weighted sum of these lobes, where weights are optimized globally [S2]. The average matrix for cluster $j$ is calculated as [S2]:
$$
S_{i,j} = \frac{1}{|V(i,j)|} \sum_{i' \in V(i,j)} S_{i'}^{orig}
$$
[S2]

## Operating characteristics
- **Time Complexity:** The Iso and Aniso algorithms are $O(1)$ per low-resolution voxel, as they involve a constant number of arithmetic operations and logarithms [S1]. The Phase Function Clustering variant adds the cost of K-means clustering on the input voxels within the block, which is $O(k \cdot n \cdot d)$ where $k$ is the number of clusters, $n$ is the number of input voxels in the block, and $d=9$ is the dimensionality of the SGGX matrix vector [S2].
- **Memory Complexity:** $O(1)$ for Iso/Aniso. $O(n)$ for Clustering to store the input matrices and cluster assignments [S2].
- **Measured Performance:** Not given in the sources.
- **Strengths:** Exact matching of directional transparencies along principal axes (Aniso) or average transparency (Iso) [S1]. Simple and fast [S1]. Clustering preserves complex scattering anisotropy [S2].
- **Limitations:** Iso ignores anisotropy in self-shadowing [S1]. Aniso only matches transparencies along principal axes, not arbitrary directions [S1]. Clustering requires global optimization of weights for best results, adding complexity [S2].
- **When to use:** Use Iso for isotropic or weakly anisotropic media where speed is critical [S1]. Use Aniso for media with strong directional extinction anisotropy [S1]. Use Clustering when the scattering phase function is highly anisotropic and cannot be approximated by a single lobe [S2].

## Background
The method originates from the work on microflake models for volume downsampling [S1]. The specific algorithms for computing $\rho_0$ and $A_0$ from directional transparencies were introduced in "A new microflake model with microscopic self-shadowing for accurate volume downsampling" [S1]. The phase function clustering approach is described in "Downsampling scattering parameters for rendering anisotropic media" [S2]. These methods are used in computer graphics for rendering large-scale participating media efficiently by reducing the resolution of volume data while preserving visual fidelity [S1, S2].

## Sources
[S1] A new microflake model with microscopic self-shadowing for accurate volume downsampling, https://doi.org/10.1111/cgf.13346, fetched 2026-10-26
[S2] Downsampling scattering parameters for rendering anisotropic media, https://doi.org/10.1145/2980179.2980228, fetched 2026-10-28
