# Effective BRDF for Planar Sources

Also called: Effective Forward Scattering BRDF, Forward Scattering BRDF, Planar Propagator

## Overview
The Effective BRDF is a computationally efficient approximation for rendering optically dense, approximately planar media under low-frequency illumination, derived as a limiting case of the subsurface scattering model [S1]. It replaces the expensive integration of the subsurface propagator over a surface area with a single bidirectional reflectance evaluation, enabling its use in level-of-detail hierarchies for participating media [S1].

## How it works
**Inputs and Outputs**
*   **Inputs**: Surface query point $\vec{x}_o$, outgoing direction $\hat{\omega}_e^o$, incoming direction $\hat{\omega}_i^i$, surface normal $\hat{n}$, medium scattering parameters ($N, p, F, C, D, E, \sigma_{a_3}$), effective boundary distance $d_e$ (if non-index-matched), and surface BSDF $f_s$ (if non-index-matched).
*   **Outputs**: Effective BRDF value $f_r(\hat{\omega}_i^i \leftrightarrow \hat{\omega}_e^o)$ or a sampled direction for recursive evaluation.

**Data Structures**
No specific data structures are required beyond standard vector/matrix types for directions and normals. The parameters $N, p, F, C, D, E, \sigma_{a_3}$ are constants derived from the medium's optical properties (absorption $\sigma_a$, scattering $\sigma_s$, anisotropy $g$) as defined in the parent [BSSRDF Model](6674-bssrdf-model.md).

**Method Steps**

1.  **Compute the Planar Propagator $G_f^{\text{pb}}$**:
    Determine the boundary-aware planar propagator based on the outgoing direction relative to the surface normal.
    
    Let $\omega_{1n} = \hat{\omega}_1 \cdot \hat{n}$ and $\omega_{0n} = \hat{\omega}_0 \cdot \hat{n}$.
    
    $$
    G_f^{\text{pb}}(s, \hat{\omega}_0, \hat{\omega}_1) = 
    \begin{cases} 
    G_f^{\text{p}}(s, 0, \hat{\omega}_1, \hat{\omega}_0) & \text{if } \omega_{1n} \geq 0 \\
    -G_f^{\text{p}}(s, d_e, \hat{\omega}_1, \hat{\omega}_v) & \text{otherwise}
    \end{cases}
    $$
    
    Where $G_f^{\text{p}}$ is the planar source propagator:
    $$
    G_f^{\text{p}}(s, d, \hat{\omega}, \hat{\omega}') = \frac{\pi N}{p^2 F} \exp \left\{ -C - D (\hat{\omega} \cdot \hat{\omega}') - E p d (\omega_n + \omega'_n) - F (p d)^2 + \frac{E^2}{4F} [2 + 2 (\hat{\omega} \cdot \hat{\omega}') - (\omega_n + \omega'_n)^2] - \sigma_{a_3} \right\}
    $$
    
    Here, $\omega_n = \hat{\omega} \cdot \hat{n}$ and $\omega'_n = \hat{\omega}' \cdot \hat{n}$. The term $d_e$ and $\hat{\omega}_v$ correspond to the effective boundary distance and virtual direction defined in the parent BSSRDF model's boundary handling (Eq. 13 in [S1]).

2.  **Evaluate the Effective BRDF ($f_r$)**:
    
    *   **Index-Matched Medium**:
        The effective BRDF is the integral of the planar propagator over the scattering length $s$:
        $$
        f_r(\hat{\omega}_i^i \leftrightarrow \hat{\omega}_e^o) = R \int G_f^{\text{pb}}(s, \hat{\omega}_e^o, -\hat{\omega}_i^i) \, ds
        $$
        Where $R$ is the Fresnel reflection coefficient (or appropriate scaling factor for index matching). Note: The integral over $s$ is typically handled analytically or via precomputed tables in implementation, as $G_f^{\text{pb}}$ depends on $s$ through the parameters $N, p, F, C, D, E$ which are functions of $s$ [S1].
        
    *   **Non-Index-Matched Medium (Recursive)**:
        The effective BRDF is not a direct function but a recursive process. To evaluate the contribution at $\vec{x}_o$:
        1.  Sample a direction $\hat{\omega}_s$ from the index-matched effective BRDF $f_r(-\hat{\omega}_e^i \leftrightarrow -\hat{\omega}_i^o)$ (note the sign flip for incoming/outgoing consistency in the recursive step).
        2.  Evaluate the surface BSDF $f_s$ at the intersection point of the ray cast in direction $\hat{\omega}_s$.
        3.  If the BSDF generates a query direction pointing *out* of the medium, terminate and return the result.
        4.  Otherwise, recursively evaluate the rendering equation, replacing the integral of the subsurface propagator with $f_r(-\hat{\omega}_e^i \leftrightarrow -\hat{\omega}_i^o)$ and setting the new query point $\vec{x}_i = \vec{x}_o$ (effectively treating the surface interaction as a bounce that re-enters the subsurface model).

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def effective_brdf_planar_source(x_o, omega_e_o, omega_i_i, n, params, fs):
    """
    Computes the effective BRDF contribution for optically dense planar media.
    
    Args:
        x_o: Surface point
        omega_e_o: Outgoing direction
        omega_i_i: Incoming direction
        n: Surface normal
        params: Dictionary containing N, p, F, C, D, E, sigma_a3, d_e, omega_v
        fs: Surface BSDF function
        
    Returns:
        Radiance contribution
    """
    # 1. Compute Planar Propagator G_f_pb
    # Note: In practice, the integral over s is precomputed or approximated.
    # Here we assume a function get_gf_pb that handles the s-integration and boundary logic.
    
    # Determine boundary-aware propagator
    omega_1n = dot(omega_e_o, n)
    
    if omega_1n >= 0:
        # Outgoing direction is above surface
        # G_f_pb(s, omega_0, omega_1) = G_f_p(s, 0, omega_1, omega_0)
        # Note: The formula uses omega_0 and omega_1. 
        # In the BRDF context: omega_0 corresponds to incoming (flipped), omega_1 to outgoing.
        # The integral is over s.
        gf_pb = integrate_gf_p_over_s(s=0, omega_1=omega_e_o, omega_0=-omega_i_i, params=params)
    else:
        # Outgoing direction is below surface (internal)
        # G_f_pb(s, omega_0, omega_1) = -G_f_p(s, d_e, omega_1, omega_v)
        gf_pb = -integrate_gf_p_over_s(s=params['d_e'], omega_1=omega_e_o, omega_0=params['omega_v'], params=params)
        
    # 2. Apply Fresnel/Reflection factor R
    # R is typically derived from the index mismatch. For index-matched, R=1 or specific scaling.
    # The source states: f_r = R * integral(G_f_pb ds)
    # Assuming R is part of the precomputed integral or a separate factor.
    # For index-matched, R is often 1 or handled in the propagator normalization.
    # We assume the integral result includes the necessary scaling for the specific medium.
    
    # For non-index-matched, we use recursion.
    # The source says: replace integral of G_f^b with f_r(-omega_e^i <-> -omega_i^o)
    # This implies f_r is the *index-matched* BRDF used recursively.
    
    # If the medium is index-matched, return the direct value.
    if params['is_index_matched']:
        return gf_pb # Assuming gf_pb already includes R and integration
        
    # If non-index-matched, we sample from the index-matched BRDF and recurse.
    # Sample direction from f_r (index-matched)
    # Note: The recursive step samples from f_r(-omega_e^i <-> -omega_i^o)
    # This is equivalent to sampling from f_r(omega_i_i <-> omega_e_o) due to reciprocity?
    # The source specifies: f_r(-omega_e^i <-> -omega_i^o)
    
    omega_s = sample_effective_brdf_index_matched(-omega_e_o, -omega_i_i, params)
    
    # Intersect ray from x_o in direction omega_s
    x_i, t = intersect_scene(x_o, omega_s)
    
    # Evaluate surface BSDF at x_i
    # The source says: "until f_s generates a query direction that points out of the medium"
    # We assume fs returns a direction and a weight.
    fs_result = fs(x_i, omega_s)
    
    if fs_result.direction_out_of_medium:
        return fs_result.radiance
    else:
        # Recursive step
        # Replace integral of G_f^b with f_r(-omega_e^i <-> -omega_i^o)
        # Set x_i = x_o (effectively, the next subsurface query is at the same point? 
        # No, the source says "setting x_i = x_o" in the context of the integral replacement.
        # This implies the subsurface scattering is evaluated at the same point for the next bounce?
        # Or does it mean the surface interaction is at x_i, but the subsurface part is at x_o?
        # "replacing the integral of G_f^b over x_i with f_r(...) and setting x_i = x_o"
        # This suggests the subsurface scattering is always evaluated at the original point x_o.
        
        # Recursive call with the new incoming direction from the surface bounce
        # The new incoming direction to the subsurface model is the reflected direction from fs
        # But the source says: f_r(-omega_e^i <-> -omega_i^o)
        # Let's map the variables:
        # Original: omega_e_o (out), omega_i_i (in)
        # Recursive: The surface bounce provides a new incoming direction to the surface.
        # The subsurface model sees this as an incoming direction.
        
        # The source says: "keep sampling f_r and f_s repeatedly"
        # So we loop.
        
        current_omega_in = fs_result.direction
        current_omega_out = omega_e_o # The original outgoing direction remains the target?
        # No, the rendering equation integrates over outgoing directions.
        # The recursion is for the *incoming* radiance calculation.
        
        # Let's follow the source strictly:
        # "integral of G_f^b over x_i has been replaced with f_r(-omega_e^i <-> -omega_i^o)"
        # This is for the term L_i in the rendering equation.
        
        # We need to compute the contribution of the subsurface scattering for the new path.
        # The new path enters at x_o, scatters, and exits.
        
        # The source implies a loop:
        while True:
            # Sample from index-matched BRDF
            # The directions for f_r are -omega_e^i and -omega_i^o
            # In the context of the current bounce, omega_e^i is the direction *from* the surface *to* the subsurface?
            # No, omega_e^i is the incoming direction to the surface.
            # The source uses notation: f_r(-omega_e^i <-> -omega_i^o)
            # This is confusing. Let's look at the indices.
            # omega_e^o: outgoing from surface.
            # omega_i^i: incoming to surface.
            # The recursive step replaces the subsurface integral.
            
            # Let's assume the standard recursive BRDF structure:
            # L_o = f_r * L_i
            # L_i comes from the surface bounce.
            
            # The source says: "setting x_i = x_o"
            # This means the subsurface scattering is evaluated at the same point x_o.
            
            # So, we sample a direction from f_r using the current incoming/outgoing directions.
            # But f_r is a BRDF, so it maps incoming to outgoing.
            # The source says: f_r(-omega_e^i <-> -omega_i^o)
            # This suggests the directions are flipped.
            
            # Let's implement the loop as described:
            # 1. Sample f_r (index matched)
            # 2. Evaluate f_s
            # 3. If f_s points out, return.
            # 4. Else, recurse.
            
            # The directions for f_r in the recursive step are:
            # Incoming to f_r: -omega_e^i (which is the direction *from* the surface *into* the medium? No.)
            # Let's stick to the source's explicit formula:
            # f_r(-omega_e^i <-> -omega_i^o)
            
            # In the first step, omega_e^i is the incoming direction to the surface (omega_i_i).
            # omega_i^o is the outgoing direction from the surface (omega_e_o).
            # So we sample f_r(-omega_i_i <-> -omega_e_o).
            
            # This is equivalent to sampling f_r(omega_e_o <-> omega_i_i) due to reciprocity?
            # f_r(a <-> b) = f_r(b <-> a).
            # So we sample f_r(omega_e_o <-> omega_i_i).
            
            # But wait, the source says "replacing the integral of G_f^b over x_i".
            # This is for the incoming radiance L_i.
            
            # Let's assume the loop variable is the current incoming direction to the surface.
            # Initially, it is omega_i_i.
            
            current_omega_in = omega_i_i
            current_omega_out = omega_e_o
            
            while True:
                # Sample from index-matched BRDF
                # The source says: f_r(-omega_e^i <-> -omega_i^o)
                # Here, omega_e^i is the incoming direction (current_omega_in)
                # omega_i^o is the outgoing direction (current_omega_out)
                # So we sample f_r(-current_omega_in <-> -current_omega_out)
                
                omega_sample = sample_effective_brdf_index_matched(-current_omega_in, -current_omega_out, params)
                
                # Intersect
                x_hit, t = intersect_scene(x_o, omega_sample)
                
                # Evaluate BSDF
                fs_res = fs(x_hit, omega_sample)
                
                if fs_res.direction_out_of_medium:
                    return fs_res.radiance
                
                # Update directions for next iteration
                # The new incoming direction to the surface is the reflected direction
                current_omega_in = fs_res.direction
                # The outgoing direction remains the target?
                # The source says "keep sampling f_r and f_s repeatedly"
                # It doesn't specify updating omega_e^o.
                # But in a path tracer, the outgoing direction is fixed for the pixel.
                # The recursion is for the *incoming* radiance.
                
                # So omega_e^o stays omega_e_o.
                # But the source says: f_r(-omega_e^i <-> -omega_i^o)
                # If omega_e^o is fixed, then -omega_i^o is fixed.
                # So we only update -omega_e^i (the incoming direction).
                
                # This matches the logic: sample a subsurface path, hit surface, bounce, repeat.
                
                # However, the source says "setting x_i = x_o".
                # This implies the subsurface scattering is always at x_o.
                # So the intersection x_hit is only for the surface BSDF.
                # The subsurface scattering is evaluated at x_o.
                
                # So the loop is:
                # 1. Sample subsurface direction from x_o using f_r
                # 2. Hit surface at x_hit
                # 3. Evaluate BSDF at x_hit
                # 4. If out, return.
                # 5. Else, new incoming direction is BSDF reflection.
                # 6. Repeat subsurface scattering from x_o with new incoming direction.
                
                # This seems to be the intended algorithm.
                pass # Loop continues
```

**Edge Cases and Pitfalls**
*   **Planarity Assumption**: The method assumes the object is approximately planar on the scale of the reduced scattering length $1/\sigma_s'$. If the curvature is high, the approximation fails [S1].
*   **Low-Frequency Illumination**: The method assumes low-frequency illumination. High-frequency lighting (e.g., sharp shadows, small light sources) will not be captured correctly [S1].
*   **Optically Dense Media**: The approximation is valid for optically dense media. For transparent or weakly scattering media, the full BSSRDF or volume rendering should be used [S1].
*   **Boundary Handling**: The use of $d_e$ and $\hat{\omega}_v$ requires correct implementation of the effective boundary conditions from the parent BSSRDF model. Incorrect values will lead to energy errors [S1].
*   **Recursion Depth**: The recursive evaluation for non-index-matched media may require a maximum depth limit to prevent infinite loops in pathological cases [G].

## Operating characteristics
*   **Time Complexity**: $O(1)$ per surface point for index-matched media (assuming precomputed integral). For non-index-matched media, $O(k)$ where $k$ is the number of recursive bounces until the ray exits the medium.
*   **Memory Complexity**: $O(1)$, no additional data structures required.
*   **Strengths**: Computationally efficient approximation for optically dense media; enables level-of-detail rendering; avoids expensive volumetric integration [S1].
*   **Limitations**: Only valid for planar surfaces and low-frequency illumination; not suitable for transparent or weakly scattering media; accuracy degrades with high curvature [S1].
*   **When to Use**: Rendering optically dense, planar translucent materials (e.g., marble, skin, wax) where performance is critical and lighting is smooth.
*   **When Not to Use**: Transparent materials, high-frequency lighting, highly curved surfaces, or when accurate multiple scattering is required.

## Background
The Effective BRDF was introduced by Roald Frederickx and Philip Dutré in 2017 as part of their "Forward Scattering Dipole Model from a Functional Integral Approximation" [S1]. It is derived as a limiting case of their subsurface scattering model, which itself is based on a dipole approximation for forward scattering media. The method addresses the computational expense of Monte Carlo rendering of strongly forward scattering materials by providing an analytical approximation for optically dense media [S1].

## Sources
[S1] A forward scattering dipole model from a functional integral approximation, https://doi.org/10.1145/3072959.3073681, fetched 2026-10-26
