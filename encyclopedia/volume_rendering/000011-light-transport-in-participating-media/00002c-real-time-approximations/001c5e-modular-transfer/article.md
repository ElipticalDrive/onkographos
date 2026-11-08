# Modular Transfer

Also called: Modular Flux Transfer, Modular approach for volume rendering, Modular radiance transfer

## Overview
Modular Transfer (MFT) is a precomputation-based rendering technique for volumetric media with repeated structures, such as woven fabrics or wood [S1]. It approximates high-order multiple scattering by splitting light transport paths into components that are precomputed offline and computed at runtime, making path integrals separable via isotropy and diffuser events [S1]. Unlike its siblings, MFT uses flux instead of light fields to address the curse of dimensionality, allowing it to handle complex, optically thick media with strong heterogeneity and anisotropic scattering where diffusion approximations fail [S1]. It builds on [Real-time approximations](36-real-time-approximations.md) by exploiting the modularity of exemplar blocks to compress transport matrices [S1].

## How it works
**Inputs:** A volumetric medium modeled as a grid of blocks, where the full volume consists of millions of blocks but only a small number (tens or hundreds) of unique "exemplar blocks" exist [S1]. Light sources and camera position [S1].
**Outputs:** Multiple-scattered flux field $\Phi^m$ for the volume, which is used in the final gather phase to compute radiance [S1].

**Data Structures:**
*   **Block:** A 3D grid of voxels [S1].
*   **Voxel ($V_i$):** A cell in the block grid [S1].
*   **Non-empty voxel subset ($N_i$):** $\{x \in \tilde{V}_i \mid \sigma_t(x, \omega) > 0 \text{ for some } \omega\}$, where $\tilde{V}_i$ is the 3D space of voxel $V_i$ [S1]. Used to ensure sparsity and accuracy [S1].
*   **Interface:** The rectangular shared boundary between two neighboring blocks [S1].
*   **Patch ($P_i$):** An oriented face of a voxel on an interface [S1]. Each patch has a normal vector $\mathbf{n}(P_i)$ pointing into a neighboring block [S1].
*   **Transfer Matrices:** Precomputed for each exemplar block:
    *   $\tilde{T}^{vv}$: Voxel-to-voxel transfer. Block-diagonal global matrix [S1].
    *   $\tilde{T}^{vp}$: Voxel-to-patch transfer [S1].
    *   $\tilde{T}^{pp}$: Patch-to-patch transfer. Block-diagonal global matrix [S1].
    *   $\tilde{T}^{pv}$: Patch-to-voxel transfer (transpose of $\tilde{T}^{vp}$) [S1].
*   **Source Flux ($\Phi^s$):** Sparse vector representing single-scattered light arriving at voxels [S1].
*   **Flip Operator ($Q$):** Permutation matrix mapping patch index $i$ to $flip(i)$, the index of the overlapping patch with opposite normal on the neighboring block [S1].

**Formulas:**
*   Voxel-to-voxel transfer: $T^{vv}(i, j) = \int_{\Omega(N_i, N_j)} f(\bar{x}) \, d\mu(\bar{x})$ [S1].
*   Voxel-to-patch transfer: $T^{vp}(i, j) = \int_{\Omega(N_i, P_j)} f(\bar{x}) \, d\mu(\bar{x})$ [S1]. Includes cosine term on patch vertex [S1].
*   Patch-to-patch transfer: $T^{pp}(i, j) = \int_{\Omega(P_i, P_j)} f(\bar{x}) \, d\mu(\bar{x})$ [S1].
*   Source flux: $\Phi^s(i) = \int_{\Omega_1(N_i, C)} \int_{S^2} M(\mathbf{x}_0, \boldsymbol{\omega}, \overline{\mathbf{x}_0\mathbf{x}_1}) f(\bar{x}) \, d\boldsymbol{\omega} \, d\mu(\bar{x})$ [S1]. $\Omega_1$ is unscattered paths from voxel $N_i$ to emissive surfaces $C$ [S1]. $M$ includes material term [S1].
*   Ground-truth multiple-scattered flux: $\Phi^{gt}(i) = \int_{\Omega_{2+}(N_i, C)} f(\bar{x}) \, d\mu(\bar{x})$ [S1]. $\Omega_{2+}$ means paths with 2 or more segments [S1].
*   Approximate multiple-scattered flux: $\Phi^m = \tilde{T}^{vv}\Phi^s + \tilde{T}^{pv}U^{-1}Q\tilde{T}^{vp}\Phi^s$, where $U = I - Q\tilde{T}^{pp}$ [S1].

**Parameters:**
*   **Block size:** ~0.5 mm for fabric (yarn crossing) [S1].
*   **Exemplar resolution:** ~1000 non-empty voxels and 500 patches per block [S1].
*   **k:** Number of scattering events handled by explicit path tracing before MFT lookup [S1]. $k=6$ for structured materials (e.g., twill), $k=2-3$ for random structures (e.g., felt) [S1].
*   **Max scattering depth:** 6 scatterings [S1].
*   **Single-scattering albedo:** ~0.975 for brightest channel in fabric [S1].

**Steps:**

1.  **Precomputation (Offline):**
    For each unique exemplar block:
    1.  Divide block into $n$ voxels. Identify non-empty subsets $N_i$ [S1].
    2.  Identify patches $P_j$ on interfaces [S1].
    3.  Compute $\tilde{T}^{vv}$, $\tilde{T}^{vp}$, and $\tilde{T}^{pp}$ using Monte Carlo particle tracing [S1].
    4.  Store matrices (e.g., as OpenEXR images) [S1].

2.  **Runtime Phase 1: Source Flux:**
    1.  Trace particles from light sources to voxels [S1].
    2.  Compute $\Phi^s(i)$ for each voxel $i$ [S1].
    3.  Store $\Phi^s$ as a sparse vector [S1].

3.  **Runtime Phase 2: Modular Transfer:**
    1.  Compute short-range flux: $\Phi_{short} = \tilde{T}^{vv}\Phi^s$ [S1].
    2.  Compute patch flux $\Phi^p$ by solving $U\Phi^p = Q\tilde{T}^{vp}\Phi^s$ [S1].
        *   Use Monte Carlo Matrix Inversion (Algorithm 2) or Jacobi Iteration [S1].
    3.  Compute long-range flux: $\Phi_{long} = \tilde{T}^{pv}\Phi^p$ [S1].
    4.  Total multiple-scattered flux: $\Phi^m = \Phi_{short} + \Phi_{long}$ [S1].

4.  **Runtime Phase 3: Final Gathering:**
    1.  Run standard path tracing [S1].
    2.  For paths with $\ge k$ scatterings, look up precomputed $\Phi^m$ to approximate indirect illumination [S1].
    3.  Paths with $< k$ scatterings are computed explicitly [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import random

def sample_row(A, i):
    """Returns (index, value / probability) for a row sample."""
    row = A[i]
    total = sum(row)
    if total == 0:
        return 0, 0.0
    r = random.random() * total
    cumulative = 0.0
    for j, val in enumerate(row):
        cumulative += val
        if cumulative >= r:
            prob = val / total
            return j, val / prob if prob > 0 else 0.0
    return len(row) - 1, row[-1] / (total / len(row)) if total > 0 else 0.0

def compute_Tvv(block, num_particles):
    """Computes voxel-to-voxel transfer matrix Tvv for a single exemplar block."""
    num_voxels = block.num_voxels
    Tvv = [[0.0] * num_voxels for _ in range(num_voxels)]
    for i in range(num_voxels):
        Ni = block.get_non_empty_subset(i)
        if not Ni:
            continue
        for _ in range(num_particles):
            x = block.sample_point_in(Ni)
            omega = block.sample_uniform_sphere()
            weight = 4.0 * math.pi * block.volume(Ni)
            current_x, current_omega = x, omega
            while True:
                s = block.sample_distance(current_x, current_omega)
                x_prime = current_x + s * current_omega
                if not block.contains(x_prime):
                    break
                j = block.get_voxel_index(x_prime)
                sigma_t = block.get_extinction_coefficient(x_prime, current_omega)
                Tvv[i][j] += weight / sigma_t
                albedo = block.get_albedo(x_prime)
                weight *= albedo
                current_omega = block.sample_phase_function(x_prime, current_omega)
                current_x = x_prime
    for i in range(num_voxels):
        for j in range(num_voxels):
            Tvv[i][j] /= num_particles
    return Tvv

def compute_Tvp_Tpp(block, num_particles):
    """Computes voxel-to-patch and patch-to-patch transfer matrices."""
    num_voxels = block.num_voxels
    num_patches = block.num_patches
    Tvp = [[0.0] * num_patches for _ in range(num_voxels)]
    Tpp = [[0.0] * num_patches for _ in range(num_patches)]
    
    # Tvp: trace from voxels to patches
    for i in range(num_voxels):
        Ni = block.get_non_empty_subset(i)
        if not Ni:
            continue
        for _ in range(num_particles):
            x = block.sample_point_in(Ni)
            omega = block.sample_uniform_sphere()
            weight = 4.0 * math.pi * block.volume(Ni)
            current_x, current_omega = x, omega
            while True:
                s = block.sample_distance(current_x, current_omega)
                x_prime = current_x + s * current_omega
                if not block.contains(x_prime):
                    break
                patch_idx = block.get_patch_index(x_prime)
                if patch_idx >= 0:
                    Tvp[i][patch_idx] += weight / block.get_extinction_coefficient(x_prime, current_omega)
                    break
                j = block.get_voxel_index(x_prime)
                Tvp[i][patch_idx] # Not applicable for Tvv deposition
                albedo = block.get_albedo(x_prime)
                weight *= albedo
                current_omega = block.sample_phase_function(x_prime, current_omega)
                current_x = x_prime
                
    # Tpp: trace from patches to patches
    for j in range(num_patches):
        Pj = block.get_patch(j)
        x = block.sample_point_on_patch(Pj)
        omega = block.sample_cosine_hemisphere(Pj.normal)
        weight = block.area(Pj)
        current_x, current_omega = x, omega
        while True:
            s = block.sample_distance(current_x, current_omega)
            x_prime = current_x + s * current_omega
            if not block.contains(x_prime):
                break
            patch_idx = block.get_patch_index(x_prime)
            if patch_idx >= 0:
                Tpp[j][patch_idx] += weight / block.get_extinction_coefficient(x_prime, current_omega)
                break
            albedo = block.get_albedo(x_prime)
            weight *= albedo
            current_omega = block.sample_phase_function(x_prime, current_omega)
            current_x = x_prime
            
    for i in range(num_voxels):
        for j in range(num_patches):
            Tvp[i][j] /= num_particles
    for i in range(num_patches):
        for j in range(num_patches):
            Tpp[i][j] /= num_particles
    return Tvp, Tpp

def solve_patch_flux_monte_carlo(Tpv, Tpp, Q, Tvp, Phi_s, num_samples):
    """Solves for Phi^Is = Tpv * Phi^p using Monte Carlo Matrix Inversion."""
    num_voxels = len(Phi_s)
    num_patches = Tpp.shape[0]
    Phi_Is = [0.0] * num_voxels
    
    for i in range(num_voxels):
        # Start at voxel i, transition to patch j1 proportional to Tpv column (or Tvp row)
        # Algorithm 2 uses T_pv, but dimensionally samples from T_vp for voxel->patch
        (j, s) = sample_row(Tvp, i)
        
        while True:
            j = Q.flip(j)
            q = 0.5 # Stop probability [G]
            if random.random() < q:
                s = s / (1.0 - q)
                break
            (j_next, v) = sample_row(Tpp, j)
            s = s * v / q
            j = j_next
            
        (i0, v) = sample_row(Tvp, j)
        Phi_Is[i] = Phi_s[i0] * s * v
        
    return Phi_Is

def modular_transfer(Tvv, Tvp, Tpp, Q, Phi_s, num_mc_samples):
    """Full Modular Transfer pipeline."""
    Phi_short = [[sum(Tvv[i][k] * Phi_s[k] for k in range(len(Phi_s)))] for i in range(len(Tvv))]
    Phi_Is = solve_patch_flux_monte_carlo(Tvp.T, Tpp, Q, Tvp, Phi_s, num_mc_samples)
    Phi_m = [Phi_short[i][0] + Phi_Is[i] for i in range(len(Phi_s))]
    return Phi_m
```

**Edge Cases and Pitfalls:**
*   **Division by Zero:** In deposition, $\sigma_t(x', \omega)$ is in the denominator [S1]. Since $x'$ is chosen by importance-sampling $\sigma_t$, $\sigma_t(x', \omega) > 0$ [S1].
*   **Energy Loss:** Applying only $\tilde{T}^{vv}$ leads to darkening because paths crossing boundaries are missing [S1]. Patches address this [S1].
*   **Matrix Asymmetry:** $U = I - Q\tilde{T}^{pp}$ is asymmetric due to $Q$, so conjugate gradient cannot be used [S1]. Jacobi or Monte Carlo must be used [S1].
*   **Storage:** Storing full $\Phi^p$ vector is expensive for large models [S1]. Monte Carlo avoids this [S1].
*   **Re-precomputation:** Requires full re-precomputation if exemplar blocks change optical properties [S1].

## Operating characteristics
*   **Time Complexity:** Precomputation is expensive (~4 hours per block) [S1]. Runtime Monte Carlo matrix inversion is <15% of total rendering time [S1]. Source flux computation is <2 minutes [S1].
*   **Memory Complexity:** Storage per block is ~20MB [S1]. Sparse vectors for $\Phi^s$ [S1].
*   **Performance:** 12 minutes for MFT vs 1.3 hours for path tracing on complex volumes (10M particles) [S1]. Outperforms photon mapping (20-60 minutes) [S1].
*   **Strengths:** Handles complex, heterogeneous, anisotropic media [S1]. Scalable to millions of blocks [S1]. High quality for high-order scattering [S1].
*   **Limitations:** Requires repeated structures (exemplar blocks) [S1]. Precomputation is slow [S1]. Requires full re-precomputation if exemplar blocks change optical properties [S1].
*   **When to use:** Static volumetric media with repeated patterns (fabrics, wood) [S1].
*   **When not to use:** Dynamic media, non-repeating structures, or when precomputation time is prohibitive [S1].

## Background
Modular Transfer was introduced by [S1] to address the difficulty of rendering complex, optically thick media with anisotropic phase functions [S1]. It builds on the idea of modularity from Loos et al. [2011] but uses flux instead of light fields to avoid the curse of dimensionality [S1]. It was implemented in the Mitsuba renderer and tested on fabrics, wood, and synthetic volumes [S1].

## Sources
[S1] Modular flux transfer, https://doi.org/10.1145/2461912.2461938, fetched 2026-10-27
