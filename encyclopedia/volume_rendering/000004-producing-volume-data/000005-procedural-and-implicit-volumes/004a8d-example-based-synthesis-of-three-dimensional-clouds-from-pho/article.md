# Example-based Synthesis of Three-dimensional Clouds from Photographs

Also called: Example-based Cloud Synthesis, Photograph-guided Cloud Volume Synthesis

## Overview
This method synthesizes a three-dimensional cloud volume $V$ storing extinction coefficients $\sigma_t$ such that its rendered image resembles an input photograph $I_{ph}$, using an example volume database $V_{ex}$ created by physically-based fluid simulation [S1]. It differs from purely procedural methods by optimizing the volume density to match photographic statistics and structural patterns found in a database of physically-based cloud simulations, rather than relying solely on noise functions or parametric weather models [S1].

## How it works

### Inputs and Outputs
*   **Input:** An input photograph $I_{ph}$ provided by the user [S1].
*   **Input:** An example cloud volume database $V_{ex} = \{V_1^{ex}, \dots, V_N^{ex}\}$, where each volume is created by physically-based fluid simulation [S1].
*   **Input:** User-specified parameters: sun direction, user-specified depth resolution $N_z$, and pixel block width $w$ [S1].
*   **Output:** A cloud volume $V$ storing the extinction coefficient $\sigma_t$ at each voxel [S1].

### Data Structures
*   **Grid:** An axis-aligned grid with resolution $N_x \times N_y \times N_z$ voxels [S1].
    *   $N_x$ and $N_y$ are equal to the resolution of the input photograph $I_{ph}$ [S1].
    *   $N_z$ is user-specified [S1].
*   **Voxel Indexing:** Each voxel is indexed by $(p, z)$, where $p$ is the corresponding pixel coordinate in the image plane and $z$ is the depth coordinate [S1].
*   **Voxel Size:** The length of a voxel is assumed to be 1 [S1].
*   **Pixel Block $B(p)$:** A set of neighborhood pixels centered around pixel $p$ with width $w$ [S1].
*   **Voxel Block $C(p)$:** A voxel block consisting of $w \times w \times N_z$ voxels, whose projected area onto the image plane is equal to $B(p)$ [S1].

### Rendering Model
The method uses parallel projection with the viewing direction parallel to the z-axis and the image plane on the xy-plane [S1]. It employs a single scattering model with an isotropic phase function and a uniform ambient term to compensate for multiple scattering effects [S1].

**Rendering Equation:**
The radiance $L(p)$ at pixel $p$ is calculated as [S1]:

$$
L(p) = L_{sky} Tr(p, N_z) + \sum_{z=1}^{N_z} L_s(p, z) Tr(p, z-1)
$$

Where:
*   $L_{sky}$ is the sky radiance [S1].
*   $Tr(p, z)$ is the transmittance between voxel $(p, z)$ and $(p, 1)$ [S1].
*   $L_s(p, z)$ is the scattered radiance at voxel $(p, z)$ [S1].

**Transmittance:**
$$
Tr(p, z) = \exp \left( - \sum_{z'=1}^{z} \sigma_t(p, z') \right)
$$

**Scattered Radiance:**
$$
L_s(p, z) = \sigma_t(p, z) (\beta f_p L_{sun} \tau(p, z) + L_a)
$$

Where:
*   $\beta$ is the albedo [S1].
*   $f_p$ is the isotropic phase function, with value $1/4\pi$ [S1].
*   $L_{sun}$ is the sun radiance [S1].
*   $\tau(p, z)$ is the sun transmittance, representing the transmittance between the voxel and the sun, affected by other voxels where incident sunlight travels before reaching $(p, z)$ [S1].
*   $L_a$ is the ambient term [S1].

**Simplified Scattered Radiance:**
Since $\beta$, $f_p$, and $L_{sun}$ are constant, their product is represented by a single parameter $L_0 = \beta f_p L_{sun}$ [S1]. The scattered radiance simplifies to [S1]:

$$
L_s(p, z) = \sigma_t(p, z) (L_0 \tau(p, z) + L_a)
$$

**Final Rendering Equation:**
Substituting $L_s$ into the rendering equation yields [S1]:

$$
L(p) = L_{sky} Tr(p, N_z) + L_0 \sum_{z=1}^{N_z} \tau(p, z) \sigma_t(p, z) Tr(p, z) + L_a \sum_{z=1}^{N_z} \sigma_t(p, z) Tr(p, z)
$$

### Parameters
*   **Sun Direction:** Provided by the user [S1].
*   **Sky Radiance $L_{sky}$:** Assumed to be uniform, determined as the average intensity of the pixels corresponding to the sky in the input photograph $I_{ph}$ [S1].
*   **Combined Sun Parameter $L_0$:** Represents $\beta f_p L_{sun}$ [S1].
*   **Ambient Term $L_a$:** Constant parameter [S1].
*   **Sun Transmittance $\tau$:** Must be computed from other voxel blocks [S1].

### Algorithm Steps

1.  **Preprocess:**
    *   Prepare a set of voxels $C^{ex}$ from the example volume database $V_{ex}$ [S1].
    *   Estimate rendering parameters (e.g., $L_0$, $L_a$) to match the rendered images of $C^{ex}$ with pixel blocks $B_{ph}(p)$ in $I_{ph}$ [S1].

2.  **Synthesis:**
    *   Minimize an energy function measuring two differences:
        1.  The difference between pixels of the rendered image of $V$ and the input photograph $I_{ph}$ [S1].
        2.  The difference between voxels of $V$ and those derived from $V_{ex}$ [S1].
    *   The optimization follows the example-based texture synthesis technique proposed by Kwatra et al. [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
from math import exp

def synthesize_cloud_volume(I_ph, V_ex, sun_direction, N_z, w):
    """
    Synthesize a cloud volume V from photograph I_ph and example volumes V_ex.
    """
    Nx, Ny = I_ph.shape[0], I_ph.shape[1]
    
    # Step 1: Preprocess
    L_sky = compute_average_sky_radiance(I_ph)
    L0, La = estimate_rendering_parameters(I_ph, V_ex, sun_direction, w)
    
    # Step 2: Synthesis via Energy Minimization
    V = initialize_volume(Nx, Ny, N_z)
    V = minimize_energy(V, I_ph, V_ex, L_sky, L0, La, sun_direction, w)
    
    return V

def compute_average_sky_radiance(I_ph):
    """Average intensity of pixels corresponding to sky in I_ph [S1]."""
    sky_mask = detect_sky_pixels(I_ph)  # [G]
    return sum(I_ph[sky_mask]) / len(sky_mask)

def estimate_rendering_parameters(I_ph, V_ex, sun_direction, w):
    """Estimate L0 and La to match rendered images of C_ex with B_ph(p) [S1]."""
    # Solves min_{L0, La} || Render(C_ex) - B_ph(p) ||^2 via least-squares [G]
    L0, La = solve_linear_radiance_fit(V_ex, I_ph, sun_direction, w)
    return L0, La

def initialize_volume(Nx, Ny, N_z):
    """Create Nx x Ny x N_z grid of zeros."""
    return [[[0.0] * N_z for _ in range(Ny)] for _ in range(Nx)]

def minimize_energy(V, I_ph, V_ex, L_sky, L0, La, sun_direction, w):
    """
    Minimize energy function measuring difference between rendered image of V and I_ph,
    and difference between voxels of V and those derived from V_ex [S1].
    Follows example-based texture synthesis technique [S1].
    """
    Nx, Ny, N_z = len(V), len(V[0]), len(V[0][0])
    for iteration in range(100):  # [G]
        for x in range(Nx):
            for y in range(Ny):
                # Patch-based optimization step [S1]
                V[x][y] = update_voxel_block(V[x][y], I_ph, V_ex, L_sky, L0, La, sun_direction, w)
    return V

def update_voxel_block(block, I_ph, V_ex, L_sky, L0, La, sun_direction, w):
    """Iteratively adjust block to minimize energy [S1]."""
    # Standard example-based synthesis update rule [G]
    return block  # [Comment: Exact solver implementation not provided in sources]

def render_voxel_block(sigma_t_block, L_sky, L0, La, tau_sun):
    """
    Render a voxel block C(p) using the single scattering model.
    sigma_t_block: list of sigma_t(p, z) for z=1..N_z
    tau_sun: list of tau(p, z) for z=1..N_z
    """
    N_z = len(sigma_t_block)
    Tr = [1.0] * (N_z + 1)
    for z in range(1, N_z + 1):
        Tr[z] = Tr[z-1] * exp(-sigma_t_block[z-1])
    
    L_val = L_sky * Tr[N_z]
    for z in range(1, N_z + 1):
        sigma_t_z = sigma_t_block[z-1]
        tau_z = tau_sun[z-1]
        Ls_z = sigma_t_z * (L0 * tau_z + La)
        L_val += Ls_z * Tr[z-1]
    return L_val

def compute_sun_transmittance(sigma_t_block, sun_direction):
    """Compute tau(p, z) along sun direction [S1]."""
    N_z = len(sigma_t_block)
    tau = [1.0] * N_z
    for z in range(1, N_z):
        tau[z] = tau[z-1] * exp(-sigma_t_block[z-1])
    return tau
```

### Edge Cases and Pitfalls
*   **Sun Transmittance:** The sun transmittance $\tau(p, z)$ is affected by other voxel blocks [S1].
*   **Sky Radiance:** $L_{sky}$ is assumed uniform [S1].
*   **Single Scattering:** The method uses a single scattering model [S1].

## Operating characteristics

*   **Time Complexity:** Not given in the sources [S1].
*   **Memory Complexity:** Not given in the sources [S1].
*   **Strengths:** Can synthesize realistic cloud structures that match the statistical and structural properties of physically-based fluid simulations, guided by a single photograph [S1].
*   **Limitations:** Relies on the quality and relevance of the example volume database $V_{ex}$ [S1]. The single scattering approximation may limit realism in certain lighting conditions [S1].
*   **When to use:** When a user wants to generate a 3D cloud volume that visually matches a specific photograph, and has access to a database of physically-based cloud simulations [S1].
*   **When not to use:** When real-time performance is critical, or when the input photograph does not contain clear cloud structures, or when the example database does not cover the desired cloud types [S1].

## Background

*   **Origin:** Proposed by Kei Iwasaki, Yoshinori Dobashi, Makoto Okabe from Wakayama University, Hokkaido University, and Shizuoka University [S1].
*   **Publication:** Published in CGI ’17, June 27-30, 2017, Yokohama, Japan [S1].
*   **History:** The method builds on example-based texture synthesis techniques, specifically citing Kwatra et al. [S1]. It addresses the difficulty of synthesizing realistic clouds from a single photograph, a task that is difficult with purely procedural methods [S1].

## Sources

[S1] Example-based synthesis of three-dimensional clouds from photographs, https://doi.org/10.1145/3095140.3095168, fetched 2026-10-29
