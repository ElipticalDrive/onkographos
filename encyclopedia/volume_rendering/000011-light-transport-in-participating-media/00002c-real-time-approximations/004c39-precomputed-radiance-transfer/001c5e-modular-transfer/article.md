# Modular Transfer

Also called: Modular Flux Transfer, Modular approach for volume rendering, Modular radiance transfer

## Overview
Modular Transfer (MFT) is a precomputation-based rendering technique for volumetric media with repeated structures, designed to approximate high-order multiple scattering efficiently [S1]. It solves the problem of rendering complex, optically thick media with anisotropic scattering by splitting light transport paths into components that are precomputed offline and computed at runtime, making path integrals separable via isotropy and diffuser events [S1]. Unlike its sibling [Precomputed Lighting (Baking)](2169-precomputed-lighting-baking.md) which typically bakes lighting into textures or lightmaps, MFT precomputes flux transfer matrices between voxels and patches to handle the "curse of dimensionality" inherent in general light transport by using flux instead of light fields [S1].

## How it works
### Inputs and Outputs
**Inputs:**
*   A volumetric medium modeled as a grid of blocks, where the full volume consists of millions of blocks but only a small number (tens or hundreds) of unique "exemplar blocks" [S1].
*   Material properties (scattering coefficient $\sigma_s$, extinction coefficient $\sigma_t$, single-scattering albedo $\alpha$, phase function) for each exemplar block [S1].
*   Light source definitions [S1].
*   Camera position and path [S1].

**Outputs:**
*   Final radiance at the camera, computed via final gathering using the precomputed multiple-scattered flux field [S1].

### Data Structures
**1. Exemplar Blocks:**
*   Each exemplar block is divided into $n$ voxels $V_1 \dots V_n$ [S1].
*   **Non-empty voxel subset $N_i$:** Defined as $N_i := \{x \in \tilde{V}_i \mid \sigma_t(x, \omega) > 0 \text{ for some } \omega\}$, where $\tilde{V}_i \subset \mathbb{R}^3$ is the 3D space of voxel $V_i$ [S1].
*   **Patches $P_i$:** Oriented faces of voxels on the interface between two neighboring blocks [S1]. Each patch has a normal vector $\mathbf{n}(P_i)$ pointing into a neighboring block [S1].
*   **Transfer Matrices:** For each exemplar block, three matrices are precomputed: $\tilde{T}^{vv}$ (voxel-to-voxel), $\tilde{T}^{vp}$ (voxel-to-patch), and $\tilde{T}^{pp}$ (patch-to-patch) [S1].
*   **Global Matrices:** $\tilde{T}^{vv}$ and $\tilde{T}^{pp}$ are constructed as block-diagonal matrices by assigning contiguous indices to voxels/patches within the same block/interface, representing an imaginary stacking of precomputed matrices for different exemplar blocks [S1].
*   **Flip Operator $Q$:** A permutation matrix on global patch indices where $Q$ maps patch $i$ to $flip(i)$, the index of the patch overlapping with $i$ but with the opposite normal direction [S1].

**2. Flux Vectors:**
*   **Source Flux $\Phi^s$:** A sparse vector where $\Phi^s(i)$ represents single-scattered light arriving at voxel $i$ directly from light sources [S1].
*   **Multiple-Scattered Flux $\Phi^m$:** The approximate ground-truth multiple-scattered flux $\Phi^{gt}$ for paths with 2 or more segments [S1].
*   **Patch Flux $\Phi^p$:** The energy emitted by each patch contributed by light paths going across one or more interfaces [S1].

### Formulas
**Transfer Matrix Definitions:**
The transfer matrices are integrals over paths $\bar{x}$ with weight $f(\bar{x})$ and measure $d\mu(\bar{x})$ [S1]:

$$T^{vv}(i, j) = \int_{\Omega(N_i, N_j)} f(\bar{x}) \, d\mu(\bar{x})$$
$$T^{vp}(i, j) = \int_{\Omega(N_i, P_j)} f(\bar{x}) \, d\mu(\bar{x})$$
$$T^{pp}(i, j) = \int_{\Omega(P_i, P_j)} f(\bar{x}) \, d\mu(\bar{x})$$

Where $\Omega(A, B)$ is the set of paths with endpoints in sets $A$ and $B$. $T^{pv}$ is the transpose of $T^{vp}$ [S1].

**Source Flux:**
$$\Phi^s(i) = \int_{\Omega_1(N_i, C)} \int_{S^2} M(\mathbf{x}_0, \boldsymbol{\omega}, \overline{\mathbf{x}_0\mathbf{x}_1}) f(\bar{x}) \, d\boldsymbol{\omega} \, d\mu(\bar{x})$$
Where $\Omega_1(N_i, C)$ is the set of unscattered paths from voxel subset $N_i$ to emissive surfaces $C$, and $M$ includes the material term [S1].

**Multiple-Scattered Flux:**
The exact expression for multiple-scattered flux is:
$$\Phi^m = \tilde{T}^{vv}\Phi^s + \tilde{T}^{pv} \left( \sum_{a=0}^{\infty} \left( Q\tilde{T}^{pp} \right)^a \right) Q\tilde{T}^{vp}\Phi^s$$
Letting $U := I - Q\tilde{T}^{pp}$, this is rewritten using the Neumann series:
$$\Phi^m = \tilde{T}^{vv}\Phi^s + \tilde{T}^{pv}U^{-1}Q\tilde{T}^{vp}\Phi^s$$
The term $\Phi^{Is} := \tilde{T}^{pv} \Phi^p = \tilde{T}^{pv} U^{-1} Q \tilde{T}^{vp} \Phi^s$ requires solving the linear system $U\Phi^p = Q\tilde{T}^{vp}\Phi^s$ [S1].

### Parameters
*   **Block Size:** Approximately 0.5 mm for woven fabric data (yarn crossing) [S1].
*   **Exemplar Block Resolution:** ~1000 non-empty voxels and 500 patches per exemplar block [S1].
*   **k (Scattering Depth):** Number of scattering events handled by standard path tracing before using MFT lookup. $k=2$ or $3$ for random structures (e.g., felt); $k=6$ for structured materials with parallel shiny fibers (e.g., twill). The authors used $k=6$ for all results [S1].
*   **Single-Scattering Albedo:** ~0.975 for the brightest color channel in single-colored fabric materials [S1].

### Steps

#### Precomputation Stage
For each unique exemplar block:
1.  Compute $\tilde{T}^{vv}$ using Monte Carlo particle tracing (Algorithm 1) [S1].
2.  Compute $\tilde{T}^{vp}$ using Monte Carlo particle tracing, similar to $\tilde{T}^{vv}$ but originating from patches [S1].
3.  Compute $\tilde{T}^{pp}$ using Monte Carlo particle tracing, analogous to $\tilde{T}^{vp}$ but depositing at other patches [S1].

**Algorithm 1: Computing Voxel-to-Voxel Transfer Matrix $T^{vv}$**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_Tvv(block, num_voxels, num_samples):
    # Initialize matrix to zeros
    Tvv = [[0.0] * num_voxels for _ in range(num_voxels)]
    for i in range(num_voxels):
        for _ in range(num_samples):
            # Sample ray origin x in non-empty subset Ni and direction omega in S^2
            x = uniform_sample_voxel_volume(block, i)
            omega = uniform_sample_unit_sphere()
            # Weight initialization: 4 * pi * |Ni|
            w = 4.0 * math.pi * block.voxel_non_empty_volume(i)
            while True:
                # Sample distance s using Woodcock tracking
                s = sample_woodcock_distance(block, x, omega)
                x_prime = x + s * omega
                if is_inside_block(block, x_prime):
                    j = find_voxel_containing(block, x_prime)
                    # Deposit energy: Tvv(i, j) += w / sigma_t(x', omega)
                    sigma_t = block.extinction_coefficient(x_prime, omega)
                    Tvv[i][j] += w / sigma_t
                    # Update weight: w *= alpha(x')
                    w *= block.single_scattering_albedo(x_prime)
                    # Sample new direction omega' from material term
                    omega = sample_phase_function(block, x_prime, omega)
                    x = x_prime
                else:
                    break
    # Average over samples
    for i in range(num_voxels):
        for j in range(num_voxels):
            Tvv[i][j] /= num_samples
    return Tvv

# Helper definitions for Algorithm 1:
# uniform_sample_voxel_volume(block, i): samples x uniformly in the non-empty subset N_i.
# uniform_sample_unit_sphere(): samples omega uniformly on the unit sphere S^2.
# sample_woodcock_distance(block, x, omega): samples s from p(s) = σ_t(x+sω,ω) exp(-∫_0^s σ_t(x+s'ω,ω) ds').
# is_inside_block(block, x): returns True if x is within the block's spatial bounds.
# find_voxel_containing(block, x): returns the voxel index j containing x.
# block.extinction_coefficient(x, omega): returns σ_t(x, omega).
# block.single_scattering_albedo(x): returns α(x).
# sample_phase_function(block, x, omega): samples ω' from the material phase function at x.
```

#### Runtime Phase 1: Source Flux
1.  Evaluate source flux vector $\Phi^s$ by tracing particles from light sources to voxels. This includes the material term (isotropy event) by integrating over all directions [S1].
2.  Store $\Phi^s$ as a sparse vector [S1].

#### Runtime Phase 2: Modular Transfer
1.  **Step 1:** Compute short-range transport within blocks: $\tilde{T}^{vv}\Phi^s$ [S1].
2.  **Step 2:** Transfer voxel fluxes into patch fluxes: $Q\tilde{T}^{vp}\Phi^s$. This gives the initial patch flux entering the interface [S1].
3.  **Step 3:** Solve for patch flux $\Phi^p$ which accounts for long-range transport across block boundaries. The system is $U\Phi^p = Q\tilde{T}^{vp}\Phi^s$ where $U = I - Q\tilde{T}^{pp}$ [S1].
    *   Since $U$ is asymmetric, use **Monte Carlo Matrix Inversion** (preferred for scalability) or **Jacobi Iteration** [S1].

**Algorithm 2: Monte Carlo Matrix Inversion for $\Phi^{Is}(i)$**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def monte_carlo_matrix_inversion(T_pv, T_pp, T_vp, Phi_s, flip, num_samples):
    # Phi_Is[i] estimates the i-th element of T_pv * U^-1 * Q * T_vp * Phi_s
    Phi_Is = [0.0] * len(Phi_s)
    for i in range(len(Phi_s)):
        total_estimate = 0.0
        for _ in range(num_samples):
            # 1. Jump from voxel i to patch j
            j, s = sample_row(T_pv, i)
            while True:
                # 2. Flip patch to neighboring block
                j = flip[j]
                # 3. Russian roulette
                q = 0.1 # Stop probability
                if random.random() < q:
                    s = s / (1.0 - q)
                    break
                # 4. Jump to another patch
                j, v = sample_row(T_pp, j)
                # Update throughput
                s = s * v / q
            # 5. Final jump to voxel i0
            i0, v = sample_row(T_vp, j)
            # Accumulate estimate
            if i0 < len(Phi_s):
                total_estimate += Phi_s[i0] * s
        Phi_Is[i] = total_estimate / num_samples
    return Phi_Is

# Helper definitions for Algorithm 2:
# flip: a permutation array where flip[j] returns the index of the patch overlapping with j but with opposite normal.
# sample_row(A, i): samples index k with probability A[i,k] / sum(A[i,:]), returns (k, A[i,k]/probability).
```
*Note: In Jacobi Iteration, start with $\Psi^{(0)} = Q\tilde{T}^{vp}\Phi^s$. In iteration $t$, compute $\Psi^{(t+1)} = Q\tilde{T}^{pp}\Psi^{(t)} + \Psi^{(0)}$. Set $\Phi^p = \Psi^{(t)}$ after a few iterations [S1].*

4.  **Step 4:** Evaluate final multiple-scattered flux: $\Phi^m = \tilde{T}^{vv}\Phi^s + \tilde{T}^{pv}\Phi^p$ [S1].

#### Runtime Phase 3: Final Gathering
1.  Run standard path tracing from the camera [S1].
2.  For paths with fewer than $k$ scattering events, compute explicitly [S1].
3.  For paths with $k$ or more scattering events, insert an isotropy event at the $k$-th vertex from the camera and the last vertex before the light. Insert zero or more diffuser events in between [S1].
4.  Look up the precomputed scattered flux field $\Phi^m$ to approximate the indirect illumination after $k$ scatterings. Specifically, approximate indirect illumination at point $x$ as $\Phi^m(i) / (4\pi |N_i|)$ where $i$ is the voxel containing $x$ [S1].

### Edge Cases and Pitfalls
*   **Division by Zero:** In the deposition step of Algorithm 1, the term $\sigma_t(x', \omega)$ is in the denominator. Since $x'$ is chosen by importance-sampling $\sigma_t$, it holds that $\sigma_t(x', \omega) > 0$, preventing division-by-zero issues [S1].
*   **Energy Loss at Boundaries:** Applying only voxel-to-voxel transfer leads to darkening because paths crossing boundaries are missing. Adding transfer across boundaries (via patches) addresses this energy loss [S1].
*   **Asymmetry of U:** The matrix $U$ is usually asymmetric because of the flip operator $Q$, so conjugate gradient based algorithms cannot be applied. Two methods are introduced: Jacobi Iteration and Monte Carlo Matrix Inversion [S1].
*   **Re-precomputation:** Requires full re-precomputation if exemplar blocks change optical properties (e.g., single-scattering albedo) [S1].

## Operating characteristics
*   **Time Complexity:** Precomputation per block takes ~4 hours. Source flux computation takes <2 minutes. Monte Carlo matrix inversion takes <15% of total rendering time [S1].
*   **Memory Complexity:** Storage per block is ~20MB. The method avoids storing the full vector $\Phi^p$ explicitly when using Monte Carlo matrix inversion, which is crucial for large models with millions of block interfaces [S1].
*   **Measured Performance:** On a scene with over one million blocks, MFT took 12 minutes (with 10M particles traced), compared to 1.3 hours for path tracing and 20 minutes for volume photon mapping (with 100M photons) [S1].
*   **Strengths:** Handles complex volumetric media with strong heterogeneity and anisotropic scattering, unlike diffusion approximations which assume isotropy, homogeneity, and flat boundaries [S1].
*   **Limitations:** Requires the volume to be composed of repeated exemplar blocks. The precomputation stage is computationally expensive [S1].

## Background
Modular Transfer was introduced in the paper "Modular Flux Transfer" [S1]. It is implemented in the Mitsuba physically based renderer [S1]. Experiments were conducted on an Intel server equipped with four Xeon X7560 eight-core CPUs. The method was applied to felt, twill, velvet, damask, wood, and synthetic coral-like volumes [S1]. It addresses the difficulty of rendering complex, optically thick media with anisotropic phase functions by precomputing light transport within blocks, inspired by the modularity of Loos et al.'s [2011] approach but using flux instead of light fields to avoid the curse of dimensionality [S1].

## Sources
[S1] Modular flux transfer, https://doi.org/10.1145/2461912.2461938, fetched 2026-10-27
