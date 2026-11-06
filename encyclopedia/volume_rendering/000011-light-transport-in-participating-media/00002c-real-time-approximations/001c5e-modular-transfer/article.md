# Modular Transfer

Also called: Modular Flux Transfer, Modular approach for volume rendering, Modular radiance transfer

## Overview
Modular Transfer (MFT) is a precomputation-based rendering technique for volumetric media with repeated structures, such as woven fabrics or wood. It approximates high-order multiple scattering by splitting light transport paths into components that are precomputed offline and computed at runtime, making path integrals separable via isotropy and diffuser events [S1]. Unlike its siblings, MFT uses flux instead of light fields to address the curse of dimensionality, allowing it to handle complex, optically thick media with strong heterogeneity and anisotropic scattering where diffusion approximations fail [S1]. It builds on [Real-time approximations](36-real-time-approximations.md) by exploiting the modularity of exemplar blocks to compress transport matrices.

## How it works
**Inputs:** A volumetric medium modeled as a grid of blocks, where the full volume consists of millions of blocks but only a small number (tens or hundreds) of unique "exemplar blocks" exist. Each exemplar block is voxelized. Light sources and camera position.
**Outputs:** Multiple-scattered flux field $\Phi^m$ for the volume, which is used in the final gather phase to compute radiance.

**Data Structures:**
*   **Block:** A 3D grid of voxels.
*   **Voxel ($V_i$):** A cell in the block grid.
*   **Non-empty voxel subset ($N_i$):** $\{x \in \tilde{V}_i \mid \sigma_t(x, \omega) > 0 \text{ for some } \omega\}$, where $\tilde{V}_i$ is the 3D space of voxel $V_i$. Used to ensure sparsity.
*   **Interface:** The rectangular shared boundary between two neighboring blocks.
*   **Patch ($P_i$):** An oriented face of a voxel on an interface. Each patch has a normal vector $\mathbf{n}(P_i)$ pointing into a neighboring block.
*   **Transfer Matrices:** Precomputed for each exemplar block:
    *   $\tilde{T}^{vv}$: Voxel-to-voxel transfer. Block-diagonal global matrix.
    *   $\tilde{T}^{vp}$: Voxel-to-patch transfer.
    *   $\tilde{T}^{pp}$: Patch-to-patch transfer. Block-diagonal global matrix.
    *   $\tilde{T}^{pv}$: Patch-to-voxel transfer (transpose of $\tilde{T}^{vp}$).
*   **Source Flux ($\Phi^s$):** Sparse vector representing single-scattered light arriving at voxels.
*   **Flip Operator ($Q$):** Permutation matrix mapping patch index $i$ to $flip(i)$, the index of the overlapping patch with opposite normal on the neighboring block.

**Formulas:**
*   Voxel-to-voxel transfer: $T^{vv}(i, j) = \int_{\Omega(N_i, N_j)} f(\bar{x}) \, d\mu(\bar{x})$ [S1].
*   Voxel-to-patch transfer: $T^{vp}(i, j) = \int_{\Omega(N_i, P_j)} f(\bar{x}) \, d\mu(\bar{x})$ [S1]. Includes cosine term on patch vertex.
*   Patch-to-patch transfer: $T^{pp}(i, j) = \int_{\Omega(P_i, P_j)} f(\bar{x}) \, d\mu(\bar{x})$ [S1].
*   Source flux: $\Phi^s(i) = \int_{\Omega_1(N_i, C)} \int_{S^2} M(\mathbf{x}_0, \boldsymbol{\omega}, \overline{\mathbf{x}_0\mathbf{x}_1}) f(\bar{x}) \, d\boldsymbol{\omega} \, d\mu(\bar{x})$ [S1]. $\Omega_1$ is unscattered paths from voxel $N_i$ to emissive surfaces $C$. $M$ includes material term.
*   Ground-truth multiple-scattered flux: $\Phi^{gt}(i) = \int_{\Omega_{2+}(N_i, C)} f(\bar{x}) \, d\mu(\bar{x})$ [S1]. $\Omega_{2+}$ means paths with 2 or more segments.
*   Approximate multiple-scattered flux: $\Phi^m = \tilde{T}^{vv}\Phi^s + \tilde{T}^{pv}U^{-1}Q\tilde{T}^{vp}\Phi^s$, where $U = I - Q\tilde{T}^{pp}$ [S1].

**Parameters:**
*   **Block size:** ~0.5 mm for fabric (yarn crossing). [S1]
*   **Exemplar resolution:** ~1000 non-empty voxels and 500 patches per block. [S1]
*   **k:** Number of scattering events handled by explicit path tracing before MFT lookup. $k=6$ for structured materials (e.g., twill), $k=2-3$ for random structures (e.g., felt). [S1]
*   **Max scattering depth:** 6 scatterings. [S1]
*   **Single-scattering albedo:** ~0.975 for brightest channel in fabric. [S1]

**Steps:**

1.  **Precomputation (Offline):**
    For each unique exemplar block:
    1.  Divide block into $n$ voxels. Identify non-empty subsets $N_i$.
    2.  Identify patches $P_j$ on interfaces.
    3.  Compute $\tilde{T}^{vv}$, $\tilde{T}^{vp}$, and $\tilde{T}^{pp}$ using Monte Carlo particle tracing.
    4.  Store matrices (e.g., as OpenEXR images).

2.  **Runtime Phase 1: Source Flux:**
    1.  Trace particles from light sources to voxels.
    2.  Compute $\Phi^s(i)$ for each voxel $i$.
    3.  Store $\Phi^s$ as a sparse vector.

3.  **Runtime Phase 2: Modular Transfer:**
    1.  Compute short-range flux: $\Phi_{short} = \tilde{T}^{vv}\Phi^s$.
    2.  Compute patch flux $\Phi^p$ by solving $U\Phi^p = Q\tilde{T}^{vp}\Phi^s$.
        *   Use Monte Carlo Matrix Inversion (Algorithm 2) or Jacobi Iteration.
    3.  Compute long-range flux: $\Phi_{long} = \tilde{T}^{pv}\Phi^p$.
    4.  Total multiple-scattered flux: $\Phi^m = \Phi_{short} + \Phi_{long}$.

4.  **Runtime Phase 3: Final Gathering:**
    1.  Run standard path tracing.
    2.  For paths with $\ge k$ scatterings, look up precomputed $\Phi^m$ to approximate indirect illumination.
    3.  Paths with $< k$ scatterings are computed explicitly.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_Tvv(block, num_particles):
    """
    Computes voxel-to-voxel transfer matrix Tvv for a single exemplar block.
    """
    num_voxels = block.num_voxels
    Tvv = zeros((num_voxels, num_voxels))
    
    for i in range(num_voxels):
        Ni = block.get_non_empty_subset(i)
        if Ni is empty:
            continue
            
        for _ in range(num_particles):
            # Sample initial ray
            x = sample_point_in(Ni)
            omega = sample_uniform_sphere()
            weight = 4 * pi * volume(Ni)
            
            current_x = x
            current_omega = omega
            
            while True:
                # Woodcock tracking
                s = sample_distance(current_x, current_omega, block)
                x_prime = current_x + s * current_omega
                
                # Check if scattering event is within block
                if not block.contains(x_prime):
                    break
                
                # Find voxel j containing x_prime
                j = block.get_voxel_index(x_prime)
                
                # Deposit energy
                # sigma_t is guaranteed > 0 because of importance sampling
                sigma_t = block.get_extinction_coefficient(x_prime, current_omega)
                Tvv[i, j] += weight / sigma_t
                
                # Update weight
                albedo = block.get_albedo(x_prime)
                weight *= albedo
                
                # Sample new direction
                current_omega = sample_phase_function(x_prime, current_omega, block)
                current_x = x_prime
                
    # Normalize
    Tvv /= num_particles
    return Tvv

def compute_Tvp(block):
    """
    Computes voxel-to-patch transfer matrix Tvp.
    """
    num_voxels = block.num_voxels
    num_patches = block.num_patches
    Tvp = zeros((num_voxels, num_patches))
    
    # Similar to Tvv, but deposition is on patches
    # Origin is chosen on patch for Tpv, but Tvp is transpose
    # So we trace from patches to voxels for Tpv, then transpose
    # Or trace from voxels to patches directly as defined in text
    # Text says: "Computing Tpv is handled similarly to Tvv... origin on patch"
    # So Tvp(i,j) is integral from voxel i to patch j.
    # We can compute Tpv(j,i) by tracing from patch j to voxel i, then transpose.
    
    Tpv = zeros((num_patches, num_voxels))
    
    for j in range(num_patches):
        Pj = block.get_patch(j)
        # Sample origin on patch, direction importance-sampled by cosine
        x = sample_point_on_patch(Pj)
        omega = sample_cosine_hemisphere(Pj.normal)
        weight = volume(Pj) # Or area? Text says |Ni| for voxel, likely area for patch
        
        current_x = x
        current_omega = omega
        
        while True:
            s = sample_distance(current_x, current_omega, block)
            x_prime = current_x + s * current_omega
            
            if not block.contains(x_prime):
                break
                
            # Find voxel i containing x_prime
            i = block.get_voxel_index(x_prime)
            
            # Deposit
            sigma_t = block.get_extinction_coefficient(x_prime, current_omega)
            # Include cosine term for patch vertex? 
            # Text: "contribution f(x) includes a cosine term on the path vertex lying on the patch"
            # This applies to Tvp. Since we are computing Tpv (patch to voxel),
            # the deposition is on a voxel, so no cosine on deposition.
            # The cosine is on the *origin* patch for Tpv? 
            # Text: "Tpv is the transpose of Tvp... contribution f(x) includes a cosine term on the path vertex lying on the patch"
            # This implies Tvp(i,j) has cosine on Pj. So Tpv(j,i) should have cosine on Pj.
            # But we are tracing FROM patch. The cosine is part of the sampling density.
            # The weight initialization handles the solid angle/area conversion.
            
            Tpv[j, i] += weight / sigma_t
            
            albedo = block.get_albedo(x_prime)
            weight *= albedo
            
            current_omega = sample_phase_function(x_prime, current_omega, block)
            current_x = x_prime
            
    Tpv /= num_particles
    return Tpv.T # Return Tvp

def solve_patch_flux_monte_carlo(Tpv, Tpp, Q, Tvp, Phi_s, num_samples):
    """
    Solves for Phi^p using Monte Carlo Matrix Inversion.
    Returns Phi^m = Tvv * Phi_s + Tpv * Phi^p
    Note: Tvv is not passed here, computed separately.
    """
    # We need to compute Phi^Is = Tpv * Phi^p
    # Phi^p = U^{-1} Q Tvp Phi_s
    # Phi^Is = Tpv U^{-1} Q Tvp Phi_s
    
    # Algorithm 2 estimates Phi^Is(i)
    # But we need the full vector Phi^p or Phi^Is?
    # Text: "If elements of Phi^Is can be efficiently estimated, we can then compute Equation 9 with Phi^m = Tvv Phi_s + Phi^Is"
    # So we estimate Phi^Is directly.
    
    num_patches = Tpp.shape[0]
    num_voxels = Tpv.shape[1] # Tpv is patches x voxels
    Phi_Is = zeros(num_voxels) # Wait, Tpv is patches x voxels? 
    # Tpv shape: (num_patches, num_voxels)
    # Tvp shape: (num_voxels, num_patches)
    # Phi_s is voxels.
    # Q Tvp Phi_s is patches.
    # Phi^p is patches.
    # Tpv Phi^p is voxels.
    # So Phi^Is is voxels.
    
    # Algorithm 2 estimates Phi^Is(i) for a specific i.
    # We need to sum over all i? No, we estimate each element.
    
    for i in range(num_voxels):
        # Sample initial jump from voxel i to patch
        # sampleRow(Tpv, i) -> Tpv is patches x voxels. Row i? No, column i.
        # Text: "transitions to patch j1 with probability proportional to the i-th row of Tpv"
        # This implies Tpv is indexed (voxel, patch)? 
        # Text: "Tpv is the transpose of Tvp". Tvp is (voxel, patch). So Tpv is (patch, voxel).
        # "i-th row of Tpv" -> i is patch index?
        # But Phi^Is is voxels.
        # Let's re-read carefully.
        # "Phi^Is := Tpv Phi^p". Tpv is (patch, voxel). Phi^p is (patch). Result is (voxel).
        # "random walk... starts at voxel Vi".
        # "transitions to patch j1 with probability proportional to the i-th row of Tpv".
        # If i is voxel, and Tpv is (patch, voxel), then i-th column.
        # Maybe Tpv in the text refers to the matrix used for sampling, which might be Tvp?
        # "sampleRow(Tpv, i)" in Algorithm 2.
        # If i is voxel, and we sample a patch, we need Tpv^T or Tvp.
        # Algorithm 2 line 1: (j, s) <- sampleRow(Tpv, i).
        # If Tpv is (patch, voxel), sampleRow(i) doesn't make sense for voxel i.
        # However, Tpv is transpose of Tvp. Tvp is (voxel, patch).
        # So Tpv^T is Tvp.
        # Maybe the text means Tvp in Algorithm 2?
        # "transitions to patch j1 with probability proportional to the i-th row of Tpv"
        # If Tpv is (patch, voxel), row i is a patch.
        # But the walk starts at voxel.
        # Let's assume the text has a typo and means Tvp (voxel, patch) for the first step.
        # Or Tpv is stored as (voxel, patch) despite the definition?
        # "Tpv is the transpose of Tvp".
        # Let's look at Algorithm 2 again.
        # Line 1: sampleRow(Tpv, i).
        # Line 9: sampleRow(Tpp, j). Tpp is (patch, patch). j is patch. Correct.
        # Line 12: sampleRow(Tvp, j). Tvp is (voxel, patch). j is patch. Correct.
        # So Line 1 must be sampling from a matrix where i is the row index.
        # If i is voxel, the matrix must be (voxel, patch). That is Tvp.
        # So Algorithm 2 likely uses Tvp in line 1, not Tpv.
        # Or Tpv is defined as (voxel, patch) in the algorithm context?
        # Given "Tpv is the transpose of Tvp", and Tvp is (voxel, patch), Tpv is (patch, voxel).
        # So Line 1 should be sampleRow(Tvp, i).
        
        # Let's implement based on the logic: Voxel -> Patch -> Patch ... -> Patch -> Voxel
        # Step 1: Voxel i to Patch j. Prob proportional to Tvp(i, j).
        
        (j, s) = sample_row(Tvp, i) # Tvp is (voxel, patch)
        
        while True:
            # Flip patch
            j = Q.flip(j)
            
            # Russian roulette
            q = 0.5 # Stop probability, not given in sources. [G]
            if random() < q:
                s = s / (1 - q)
                break
            
            # Jump to next patch
            (j_next, v) = sample_row(Tpp, j)
            s = s * v / q
            
            j = j_next
            
        # Final jump to voxel
        (i0, v) = sample_row(Tvp, j)
        Phi_Is[i] = Phi_s[i0] * s * v
        
    return Phi_Is

def modular_transfer(Tvv, Tvp, Tpp, Q, Phi_s, num_mc_samples):
    """
    Full Modular Transfer pipeline.
    """
    # 1. Short range
    Phi_short = mat_vec_mult(Tvv, Phi_s)
    
    # 2. Long range via Monte Carlo
    Phi_Is = solve_patch_flux_monte_carlo(Tvp, Tpp, Q, Tvp, Phi_s, num_mc_samples)
    
    # 3. Total
    Phi_m = Phi_short + Phi_Is
    
    return Phi_m
```

**Edge Cases and Pitfalls:**
*   **Division by Zero:** In deposition, $\sigma_t(x', \omega)$ is in the denominator. Since $x'$ is chosen by importance-sampling $\sigma_t$, $\sigma_t(x', \omega) > 0$ [S1].
*   **Energy Loss:** Applying only $\tilde{T}^{vv}$ leads to darkening because paths crossing boundaries are missing. Patches address this [S1].
*   **Matrix Asymmetry:** $U = I - Q\tilde{T}^{pp}$ is asymmetric due to $Q$, so conjugate gradient cannot be used. Jacobi or Monte Carlo must be used [S1].
*   **Storage:** Storing full $\Phi^p$ vector is expensive for large models. Monte Carlo avoids this [S1].
*   **Re-precomputation:** If exemplar blocks change optical properties, full re-precomputation is required [S1].

## Operating characteristics
*   **Time Complexity:** Precomputation is expensive (~4 hours per block). Runtime Monte Carlo matrix inversion is <15% of total rendering time. Source flux computation is <2 minutes [S1].
*   **Memory Complexity:** Storage per block is ~20MB. Sparse vectors for $\Phi^s$ [S1].
*   **Performance:** 12 minutes for MFT vs 1.3 hours for path tracing on complex volumes (10M particles). Outperforms photon mapping (20-60 minutes) [S1].
*   **Strengths:** Handles complex, heterogeneous, anisotropic media. Scalable to millions of blocks. High quality for high-order scattering.
*   **Limitations:** Requires repeated structures (exemplar blocks). Precomputation is slow. Not suitable for dynamic media with changing optical properties [S1].
*   **When to use:** Static volumetric media with repeated patterns (fabrics, wood).
*   **When not to use:** Dynamic media, non-repeating structures, or when precomputation time is prohibitive [S1].

## Background
Modular Transfer was introduced by [S1] to address the difficulty of rendering complex, optically thick media with anisotropic phase functions. It builds on the idea of modularity from Loos et al. [2011] but uses flux instead of light fields to avoid the curse of dimensionality. It was implemented in the Mitsuba renderer and tested on fabrics, wood, and synthetic volumes [S1].

## Sources
[S1] Modular flux transfer, https://doi.org/10.1145/2461912.2461938, fetched 2026-10-27
