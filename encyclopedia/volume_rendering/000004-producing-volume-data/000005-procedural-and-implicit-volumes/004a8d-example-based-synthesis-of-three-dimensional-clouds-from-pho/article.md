# Example-based Synthesis of Three-dimensional Clouds from Photographs

Also called: Example-based Cloud Synthesis, Photograph-guided Cloud Volume Synthesis

## Overview
This method synthesizes a three-dimensional cloud volume storing extinction coefficients $\sigma_t$ such that its rendered image resembles an input photograph $I_{ph}$, using an example volume database $V_{ex}$ created by fluid simulation [S1]. It differs from purely procedural methods by optimizing the volume density to match photographic statistics and structural patterns found in a database of physically-based cloud simulations, rather than relying solely on noise functions or parametric weather models [S1].

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
def synthesize_cloud_volume(I_ph, V_ex, sun_direction, N_z, w):
    """
    Synthesize a cloud volume V from photograph I_ph and example volumes V_ex.
    
    Args:
        I_ph: Input photograph (Nx x Ny x 3).
        V_ex: List of example cloud volumes created by fluid simulation.
        sun_direction: 3D vector for sun direction.
        N_z: User-specified depth resolution.
        w: Width of the pixel/voxel block.
        
    Returns:
        V: Cloud volume (Nx x Ny x N_z) storing extinction coefficients sigma_t.
    """
    Nx, Ny = I_ph.shape[0], I_ph.shape[1]
    
    # Step 1: Preprocess
    # Estimate rendering parameters by matching rendered example volumes to photograph patches
    # This involves finding L0 and La such that rendered C_ex matches B_ph(p)
    L0, La = estimate_rendering_parameters(I_ph, V_ex, sun_direction, w)
    
    # Determine sky radiance as average intensity of sky pixels in I_ph
    L_sky = compute_average_sky_radiance(I_ph)
    
    # Step 2: Synthesis via Energy Minimization
    # Initialize V with random or zero extinction coefficients
    V = initialize_volume(Nx, Ny, N_z)
    
    # Define Energy Function E(V)
    # E(V) = E_image(V, I_ph) + E_example(V, V_ex)
    # E_image: Difference between rendered image of V and I_ph
    # E_example: Difference between voxels of V and those derived from V_ex
    
    # Minimize E(V) using optimization technique from Kwatra et al. [S1]
    # This typically involves iterative patch-based optimization
    V = minimize_energy(V, I_ph, V_ex, L_sky, L0, La, sun_direction, w)
    
    return V

def render_voxel_block(V_block, L_sky, L0, La, sun_direction, tau_from_other_blocks):
    """
    Render a voxel block C(p) using the single scattering model.
    
    Args:
        V_block: Voxel block (w x w x N_z) of extinction coefficients sigma_t.
        L_sky: Sky radiance.
        L0: Combined sun parameter.
        La: Ambient term.
        sun_direction: 3D vector.
        tau_from_other_blocks: Sun transmittance from neighboring blocks.
        
    Returns:
        L: Rendered radiance for the block.
    """
    w, _, N_z = V_block.shape
    L = zeros((w, w))
    
    for y in range(w):
        for x in range(w):
            p = (x, y) # Local pixel coordinate in block
            sigma_t_block = V_block[y, x, :] # sigma_t(p, z) for z=1..N_z
            
            # Compute transmittance Tr(p, z)
            Tr = zeros(N_z + 1)
            Tr[0] = 1.0
            for z in range(1, N_z + 1):
                Tr[z] = Tr[z-1] * exp(-sigma_t_block[z-1])
            
            # Compute sun transmittance tau(p, z)
            # Note: tau depends on sun direction and may require integration or lookup
            # from tau_from_other_blocks if the sun path crosses block boundaries
            tau = compute_sun_transmittance(p, sigma_t_block, sun_direction, tau_from_other_blocks)
            
            # Compute scattered radiance Ls(p, z)
            Ls_sum = 0.0
            for z in range(N_z):
                sigma_t_z = sigma_t_block[z]
                tau_z = tau[z]
                Tr_z = Tr[z]
                
                # Ls(p, z) = sigma_t(p, z) * (L0 * tau(p, z) + La)
                Ls_z = sigma_t_z * (L0 * tau_z + La)
                
                # Add to accumulated radiance: Ls(p, z) * Tr(p, z-1)
                # Note: Tr(p, z-1) is Tr[z-1] in 0-indexed array if Tr[0] is top
                # The formula uses Tr(p, z-1) where z goes 1..N_z
                # In 0-indexed sigma_t_block, z=1 corresponds to index 0
                # Tr(p, z-1) for z=1 is Tr(p, 0) which is 1.0
                L += Ls_z * Tr[z] # Tr[z] corresponds to Tr(p, z) in formula? 
                # Formula: sum_{z=1}^{N_z} Ls(p,z) Tr(p, z-1)
                # Tr(p, z-1) is transmittance from surface to z-1.
                # If Tr(p, z) is transmittance to z, then Tr(p, z-1) is Tr[z-1] in 0-indexed list where Tr[0]=1.
                # Let's align indices:
                # z_formula = 1..N_z
                # sigma_t_block[z_formula-1]
                # Tr_formula[z_formula-1] = Tr(p, z_formula-1)
                # So we use Tr[z_formula-1]
                pass 
            
            # Correct loop implementation based on formula (5)
            # L(p) = L_sky * Tr(p, N_z) + L0 * sum(tau * sigma_t * Tr(p, z)) + La * sum(sigma_t * Tr(p, z))
            # Note: Formula (5) uses Tr(p, z) in the sum, not Tr(p, z-1). 
            # Wait, formula (1) uses Tr(p, z-1). Formula (5) is derived from (1).
            # Let's re-read formula (5):
            # L(p) = L_sky Tr(p, N_z) + L0 sum_{z=1}^{N_z} tau(p,z) sigma_t(p,z) Tr(p,z) + La sum_{z=1}^{N_z} sigma_t(p,z) Tr(p,z)
            # This implies Tr(p,z) in the sum is transmittance FROM z TO 1? 
            # Formula (2): Tr(p,z) = exp(-sum_{z'=1}^z sigma_t). This is transmittance from z to 1 (or 0 to z depending on coord system).
            # In formula (1), Tr(p, z-1) is used.
            # If Tr(p, z) is transmittance from z to 1, then Tr(p, z-1) is transmittance from z-1 to 1.
            # The term Ls(p,z) is scattered at z. It travels from z to 1. So it should be multiplied by Tr(p, z).
            # Formula (1) says Tr(p, z-1). This suggests z is indexed from top (1) to bottom (N_z).
            # If z=1 is top, Tr(p, 0) = 1. Tr(p, 1) = exp(-sigma_t(1)).
            # Light scattered at z=1 travels to camera (z=0). Transmittance is Tr(p, 1)? Or Tr(p, 0)?
            # Formula (2) defines Tr(p, z) as exp(-sum_{z'=1}^z sigma_t).
            # So Tr(p, 1) = exp(-sigma_t(1)).
            # Formula (1): sum_{z=1}^{N_z} Ls(p,z) Tr(p, z-1).
            # For z=1, term is Ls(p,1) Tr(p, 0) = Ls(p,1) * 1.
            # For z=2, term is Ls(p,2) Tr(p, 1) = Ls(p,2) * exp(-sigma_t(1)).
            # This matches standard volume rendering where light from z travels through voxels 1..z-1.
            # Formula (5) seems to have a typo or different definition of Tr in the sum.
            # "L0 sum_{z=1}^{N_z} tau(p,z) sigma_t(p,z) Tr(p,z)"
            # If Tr(p,z) in formula (5) means Tr(p, z-1) from formula (1), then it's consistent.
            # However, formula (5) explicitly writes Tr(p,z).
            # Let's stick to the explicit formula (5) provided in the source text for implementation, 
            # but note the potential ambiguity. The source text says "By substituting Ls into Eq. (1), L(p) is calculated by the following equation."
            # Eq (1): sum Ls Tr(p, z-1).
            # Eq (5): sum ... Tr(p, z).
            # This implies Tr(p, z) in Eq (5) might be a typo for Tr(p, z-1) OR Tr(p, z) in Eq (5) is defined differently.
            # Given Eq (2) defines Tr(p, z), and Eq (1) uses Tr(p, z-1), Eq (5) likely contains a typo in the source or uses a shifted index.
            # Standard rendering: L = L_sky * T(N) + sum_{z=1}^N L_s(z) * T(z-1).
            # I will implement according to Eq (1) and (2) which are physically consistent.
            
            # Re-implementing based on Eq (1) and (2) for correctness:
            L_val = L_sky * Tr[N_z] # Tr(p, N_z)
            for z in range(1, N_z + 1):
                sigma_t_z = sigma_t_block[z-1]
                tau_z = tau[z-1] # tau(p, z)
                Tr_z_minus_1 = Tr[z-1] # Tr(p, z-1)
                
                Ls_z = sigma_t_z * (L0 * tau_z + La)
                L_val += Ls_z * Tr_z_minus_1
            
            L[y, x] = L_val
            
    return L

def minimize_energy(V, I_ph, V_ex, L_sky, L0, La, sun_direction, w):
    """
    Minimize the energy function E(V) = E_image + E_example.
    Uses example-based texture synthesis technique [S1].
    """
    # This is a complex optimization problem.
    # The source cites Kwatra et al. [11] for the technique.
    # A full implementation requires defining the specific energy terms and optimization solver.
    # E_image: || Render(V) - I_ph ||^2
    # E_example: Sum over patches of || Patch(V) - BestMatchPatch(V_ex) ||^2
    # This is typically solved via iterative patch-based optimization (e.g., graph-cut or iterative sampling).
    # Since the source does not provide the full optimization algorithm, we mark this as a placeholder for the specific solver.
    pass
```

### Edge Cases and Pitfalls
*   **Sun Transmittance:** The sun transmittance $\tau(p, z)$ is affected by other voxel blocks. Computing this accurately requires handling boundary conditions between voxel blocks [S1].
*   **Sky Radiance:** $L_{sky}$ is assumed uniform. If the sky in $I_{ph}$ is not uniform, this assumption may introduce errors [S1].
*   **Single Scattering:** The method uses a single scattering model. While an ambient term $L_a$ compensates for multiple scattering, it is an approximation and may not fully capture complex lighting in dense clouds [S1].

## Operating characteristics

*   **Time Complexity:** Not given in the sources. The synthesis involves minimizing an energy function, which is typically iterative and computationally expensive, similar to example-based texture synthesis [S1].
*   **Memory Complexity:** Not given in the sources. Depends on the size of the example volume database $V_{ex}$ and the resolution $N_x \times N_y \times N_z$ [S1].
*   **Strengths:** Can synthesize realistic cloud structures that match the statistical and structural properties of physically-based fluid simulations, guided by a single photograph [S1].
*   **Limitations:** Relies on the quality and relevance of the example volume database $V_{ex}$ [S1]. The single scattering approximation may limit realism in certain lighting conditions [S1].
*   **When to use:** When a user wants to generate a 3D cloud volume that visually matches a specific photograph, and has access to a database of physically-based cloud simulations [S1].
*   **When not to use:** When real-time performance is critical, or when the input photograph does not contain clear cloud structures, or when the example database does not cover the desired cloud types [S1].

## Background

*   **Origin:** Proposed by Kei Iwasaki, Yoshinori Dobashi, Makoto Okabe from Wakayama University, Hokkaido University, and Shizuwa University [S1].
*   **Publication:** Published in CGI ’17, June 27-30, 2017, Yokohama, Japan [S1].
*   **History:** The method builds on example-based texture synthesis techniques, specifically citing Kwatra et al. [11] (not provided in sources, but cited in [S1]) [S1]. It addresses the difficulty of synthesizing realistic clouds from a single photograph, a task that is difficult with purely procedural methods [S1].

## Sources

[S1] Example-based synthesis of three-dimensional clouds from photographs, https://doi.org/10.1145/3095140.3095168, fetched 2026-10-29
