# Modular Transfer

Also called: MFT, Modular Flux Transfer, Modular approach for volume rendering, Modular radiance transfer

## Overview
Modular Transfer (MFT) is a precomputation-based rendering technique for volumetric media with repeated structures, such as fabrics and wood, that approximates high-order multiple scattering by modularly combining precomputed flux transfer matrices for exemplar blocks [S1]. It solves the problem of rendering complex, optically thick media with anisotropic scattering by splitting light transport paths into components precomputed offline and computed at runtime, making path integrals separable via isotropy and diffuser events [S1]. Unlike its siblings, MFT uses flux instead of light fields to address the curse of dimensionality, avoiding the massive storage overhead of directional discretizations [S1]. It builds on [Real-time approximations](36-real-time-approximations.md) by providing a scalable alternative to voxel cone tracing and froxels for heterogeneous media where diffusion approximations fail due to strong heterogeneity and anisotropy [S1].

## How it works
**Inputs:** A volumetric medium modeled as a grid of blocks, where the full volume consists of millions of blocks but only a small number (tens or hundreds) of unique "exemplar blocks" exist [S1]. Light sources and camera position.
**Outputs:** Final radiance at the camera, computed via final gathering using the precomputed multiple-scattered flux field.

**Data Structures:**
*   **Block:** A 3D region of the volume. For woven fabrics, block size is ~0.5 mm [S1].
*   **Voxel ($V_i$):** A subdivision of a block used for precomputation resolution. Each block contains $n$ voxels.
*   **Non-empty voxel subset ($N_i$):** $N_i := \{x \in \tilde{V}_i \mid \sigma_t(x, \omega) > 0 \text{ for some } \omega\}$, where $\tilde{V}_i$ is the 3D space of voxel $V_i$. Used to ensure sparsity [S1].
*   **Interface:** The rectangular shared boundary between two neighboring blocks [S1].
*   **Patch ($P_i$):** An oriented face of a voxel on an interface. Each patch has a normal vector $\mathbf{n}(P_i)$ pointing into a neighboring block. Patches connect transfer between blocks [S1].
*   **Transfer Matrices:** Precomputed for each exemplar block:
    *   $\tilde{T}^{vv}$: Voxel-to-voxel transfer.
    *   $\tilde{T}^{vp}$: Voxel-to-patch transfer.
    *   $\tilde{T}^{pp}$: Patch-to-patch transfer.
    *   $\tilde{T}^{pv} = (\tilde{T}^{vp})^T$: Patch-to-voxel transfer [S1].
    *   Global matrices $\tilde{T}^{vv}$ and $\tilde{T}^{pp}$ are block-diagonal, formed by assigning contiguous indices to voxels/patches within the same block/interface [S1].
*   **Source Flux ($\Phi^s$):** A sparse vector representing single-scattered light arriving at voxels directly from light sources. Non-zero entries $\le$ number of traced particles [S1].
*   **Flip Operator ($Q$):** A permutation matrix on global patch indices. $Q$ maps patch $i$ to $flip(i)$, the patch overlapping with $i$ but with the opposite normal direction, acting as a diffuser event [S1].

**Formulas:**
1.  **Voxel-to-voxel transfer:**
    $$T^{vv}(i, j) = \int_{\Omega(N_i, N_j)} f(\bar{x}) \, d\mu(\bar{x})$$
    where $\Omega(N_i, N_j)$ is the set of paths with endpoints in $N_i$ and $N_j$. $T^{vv}$ is symmetric [S1].
2.  **Voxel-to-patch transfer:**
    $$T^{vp}(i, j) = \int_{\Omega(N_i, P_j)} f(\bar{x}) \, d\mu(\bar{x})$$
    The integrand $f(\bar{x})$ includes a cosine term on the vertex lying on the patch [S1].
3.  **Patch-to-patch transfer:**
    $$T^{pp}(i, j) = \int_{\Omega(P_i, P_j)} f(\bar{x}) \, d\mu(\bar{x})$$
    $T^{pp}$ is symmetric [S1].
4.  **Source flux:**
    $$\Phi^s(i) = \int_{\Omega_1(N_i, C)} \int_{S^2} M(\mathbf{x}_0, \boldsymbol{\omega}, \overline{\mathbf{x}_0\mathbf{x}_1}) f(\bar{x}) \, d\boldsymbol{\omega} \, d\mu(\bar{x})$$
    where $\Omega_1(N_i, C)$ is the set of unscattered paths from $N_i$ to emissive surfaces $C$, and $M$ includes the material term [S1].
5.  **Ground-truth multiple-scattered flux:**
    $$\Phi^{gt}(i) = \int_{\Omega_{2+}(N_i, C)} f(\bar{x}) \, d\mu(\bar{x})$$
    where $\Omega_{2+}$ denotes paths with 2 or more segments. The goal is $\Phi^m \approx \Phi^{gt}$ [S1].
6.  **Approximate multiple-scattered flux:**
    $$\Phi^m = \tilde{T}^{vv}\Phi^s + \tilde{T}^{pv}U^{-1}Q\tilde{T}^{vp}\Phi^s$$
    where $U := I - Q\tilde{T}^{pp}$ [S1].

**Parameters:**
*   **Block size:** ~0.5 mm for woven fabric [S1].
*   **Exemplar block resolution:** ~1000 non-empty voxels and 500 patches per block [S1].
*   **k:** Number of scattering events handled by standard path tracing before MFT lookup. $k=2-3$ for random structures (felt), $k=6$ for structured materials (twill). Authors used $k=6$ [S1].
*   **Single-scattering albedo:** ~0.975 for brightest channel in fabric [S1].
*   **Max scattering depth:** 6 scatterings [S1].

**Steps:**

1.  **Precomputation (Offline):**
    For each unique exemplar block:
    1.  Divide the block into $n$ voxels. Identify non-empty subsets $N_i$.
    2.  Identify interfaces and patches $P_j$.
    3.  Compute $\tilde{T}^{vv}$, $\tilde{T}^{vp}$, and $\tilde{T}^{pp}$ using Monte Carlo particle tracing (see Algorithm 1).
    4.  Store matrices (e.g., as OpenEXR images) [S1].

2.  **Runtime Phase 1: Source Flux Evaluation:**
    1.  Trace particles from light sources to compute $\Phi^s$.
    2.  $\Phi^s(i)$ represents light arriving at voxel $i$ directly from lights (including the first scattering event/material term).
    3.  Store $\Phi^s$ as a sparse vector [S1].

3.  **Runtime Phase 2: Modular Transfer:**
    Compute $\Phi^m = \tilde{T}^{vv}\Phi^s + \tilde{T}^{pv}\Phi^p$, where $\Phi^p$ is the patch flux.
    1.  Compute initial patch flux contribution: $\Psi^{(0)} = Q\tilde{T}^{vp}\Phi^s$.
    2.  Solve for $\Phi^p$ using one of the following methods:
        *   **Jacobi Iteration:** Iterate $\Psi^{(t+1)} = Q\tilde{T}^{pp}\Psi^{(t)} + \Psi^{(0)}$ for a few iterations. Set $\Phi^p = \Psi^{(t)}$. Note: $\tilde{T}^{pp}v$ is computed by patch-to-patch transfer on each block; $Qv$ by permuting elements [S1].
        *   **Monte Carlo Matrix Inversion (Preferred for scalability):** Estimate $\Phi^{Is} = \tilde{T}^{pv}\Phi^p$ element-wise using random walks (Algorithm 2). Then $\Phi^m = \tilde{T}^{vv}\Phi^s + \Phi^{Is}$ [S1].
    3.  Evaluate $\Phi^m$ [S1].

4.  **Runtime Phase 3: Final Gathering:**
    1.  Run standard path tracing from the camera.
    2.  For paths with fewer than $k$ scattering events, compute explicitly.
    3.  For paths with $\ge k$ scattering events, insert an isotropy event at the $k$-th vertex from the camera and the last vertex before the light. Insert diffuser events between them to make integrals separable [S1].
    4.  At the isotropy event, look up the precomputed multiple-scattered flux $\Phi^m(i)$ for the current voxel $i$.
    5.  Approximate indirect illumination at point $\mathbf{x}$ as $\Phi^m(i) / (4\pi |N_i|)$ [S1].

**Algorithm 1: Computing Voxel-to-Voxel Transfer Matrix $T^{vv}$**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_Tvv(block, num_voxels, num_particles_per_voxel):
    # Initialize matrix to zeros
    Tvv = zeros((num_voxels, num_voxels))
    
    for i in range(num_voxels):
        # Get non-empty subset Ni for voxel i
        Ni = get_non_empty_subset(i)
        volume_Ni = volume(Ni)
        
        for t in range(num_particles_per_voxel):
            # Sample ray origin x in Ni and direction omega in S2
            x, omega = sample_ray_in_voxel(Ni)
            
            # Initialize weight
            w = 4 * pi * volume_Ni
            
            while True:
                # Sample distance s using Woodcock tracking
                # p(s) = sigma_t(x + s*omega, omega) * exp(-integral_0^s sigma_t(x + s'*omega, omega) ds')
                s = sample_distance_woodcock(x, omega)
                
                # Get scattering event position
                x_prime = x + s * omega
                
                # Check if scattering event is within the block
                if is_inside_block(x_prime):
                    # Find voxel j containing x_prime
                    j = get_voxel_index_containing(x_prime)
                    
                    # Deposit energy
                    # sigma_t(x_prime, omega) > 0 because of importance sampling
                    Tvv[i, j] += w / sigma_t(x_prime, omega)
                    
                    # Update weight with albedo
                    alpha = get_albedo(x_prime)
                    w = w * alpha
                    
                    # Sample new direction from material term
                    omega = sample_direction_material(x_prime, omega)
                    
                    # Continue path
                    x = x_prime
                else:
                    # Path left the block, terminate
                    break
            
    # Average over particles
    Tvv = Tvv / num_particles_per_voxel
    return Tvv
```

**Algorithm 2: Monte Carlo Matrix Inversion for $\Phi^{Is}(i)$**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sample_row(A, i):
    """
    Returns (index, value) sampled from row i of matrix A
    with probability proportional to the element's value.
    Returns value / probability.
    """
    row = A[i, :]
    total = sum(row)
    if total == 0:
        return None, 0.0
    
    # Sample index j proportional to row[j]
    r = random() * total
    cumulative = 0.0
    for j, val in enumerate(row):
        cumulative += val
        if cumulative >= r:
            prob = val / total
            return j, val / prob
    return len(row) - 1, row[-1] / total

def compute_phi_Is_element(T_pv, T_pp, T_vp, Phi_s, flip, i):
    """
    Estimates the i-th element of Phi_Is = T_pv * U^-1 * Q * T_vp * Phi_s
    using a random walk.
    """
    # Step 1: Jump from voxel i to patch j
    j, s = sample_row(T_pv, i)
    if j is None:
        return 0.0
    
    while True:
        # Step 2: Flip the patch (cross interface)
        j = flip(j)
        
        # Russian roulette to terminate
        q = 0.1 # Stop probability, e.g., 0.1
        if random() < q:
            s = s / (1 - q)
            break
        
        # Step 3: Jump to another patch j'
        j_next, v = sample_row(T_pp, j)
        if j_next is None:
            break
            
        # Update throughput
        s = s * v / q
        j = j_next
        
    # Step 4: Final jump from patch to voxel i0
    i0, v_final = sample_row(T_vp, j)
    if i0 is None:
        return 0.0
    
    # Return estimated contribution
    return Phi_s[i0] * s * v_final
```

**Edge Cases and Pitfalls:**
*   **Division by Zero:** In deposition, the term $\sigma_t(x', \omega)$ is in the denominator. Since $x'$ is chosen by importance-sampling $\sigma_t$, $\sigma_t(x', \omega) > 0$ holds [S1].
*   **Energy Loss at Boundaries:** Applying only $\tilde{T}^{vv}$ leads to darkening because paths crossing boundaries are missing. The patch-based transfer ($\tilde{T}^{vp}, \tilde{T}^{pp}, Q$) addresses this [S1].
*   **Matrix Asymmetry:** $U = I - Q\tilde{T}^{pp}$ is usually asymmetric due to $Q$, so conjugate gradient cannot be used. Jacobi or Monte Carlo methods are required [S1].
*   **Re-precomputation:** If exemplar blocks change optical properties (e.g., albedo), full re-precomputation is required [S1].
*   **Storage:** Storing directional light fields is intractable (1024x inflation for 32 bins). MFT uses flux to avoid this [S1].

## Operating characteristics
*   **Time Complexity:** Precomputation is expensive (~4 hours per block) [S1]. Runtime Monte Carlo matrix inversion takes <15% of total rendering time. Source flux computation takes <2 minutes [S1].
*   **Memory Complexity:** Storage per block is ~20 MB [S1]. The method avoids storing the full vector $\Psi^{(t)}$ (size = total patches) by using Monte Carlo inversion, making it scalable to millions of blocks [S1].
*   **Performance:** MFT achieved 12 minutes rendering time vs. 1.3 hours for path tracing (10M particles) and 20 minutes for photon mapping (100M photons) on a complex volume [S1].
*   **Strengths:** Handles strong heterogeneity and anisotropic scattering where diffusion approximations fail. Significantly faster than path tracing for equivalent quality on complex volumes. Scalable to millions of blocks.
*   **Limitations:** Requires repeated structures (exemplar blocks). Precomputation is slow. Not suitable for fully unique, non-repeating volumes.
*   **When to use:** Rendering complex volumetric materials like fabrics, wood, or synthesized textures with repeated patterns.
*   **When not to use:** Scenes with no repeated structures, or where real-time precomputation is impossible.

## Background
Modular Transfer was introduced in the paper "Modular Flux Transfer" [S1]. It is based on the Mitsuba physically based renderer [Jakob 2010] and tested on Intel Xeon X7560 CPUs. It was applied to felt, twill, velvet, damask, wood, and synthetic coral-like volumes [S1]. The method addresses the limitations of diffusion approximations (which assume isotropy, homogeneity, and flat boundaries) and the high dimensionality of light fields [S1]. It builds on the concept of modularity from Loos et al. [2011] but uses flux instead of light fields to reduce storage [S1].

## Sources
[S1] Modular flux transfer, https://doi.org/10.1145/2461912.2461938, fetched 2026-10-27
