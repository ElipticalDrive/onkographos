# SVD-Approximated Iterative Transport Equation for Multiple Scattering

Also called: Interactive Multiple Scattering with Separable Phase Function, Interactive Rendering of Multiple Scattering in Participating Media Using Separable Phase Function, SVD Factorization for Transport Equation, SVD-based phase function approximation, Separable phase function rendering

## Overview
This method enables interactive rendering of multiple scattering in non-homogeneous participating media by approximating the scattering phase function using Singular Value Decomposition (SVD) [S1, unofficial]. It solves the discrete transport equation iteratively on GPU hardware, replacing expensive integral calculations with efficient texture lookups [S1, unofficial]. Unlike particle-based methods that require path recalculation upon movement, this grid-based approach allows for dynamic scene rendering without recomputing scattering paths [S1, unofficial].

## How it works

### Inputs and Outputs
**Inputs:**
*   **Density Texture:** A 3D texture storing the density values $\rho(x)$ at each point in the volume.
*   **Emission Density:** $L_e(x, \omega_0)$, the emitted radiance at position $x$ and direction $\omega_0$.
*   **Phase Function Parameters:** The anisotropy factor $g$ for the Henyey-Greenstein phase function.
*   **Precomputed Textures:**
    *   **Incident-Direction Texture:** 2D texture storing $\xi_k u_k(\omega_0)$.
    *   **Outgoing-Direction Texture:** 2D texture storing $\xi_k v_k(\omega_i)$.
*   **Initial Radiance:** $L_0(x, \omega_0)$, typically from single scattering or the previous frame's solution.

**Outputs:**
*   **Radiance Texture:** A 3D texture storing the converged radiance $L(x, \omega_0)$ for all positions and directions.

### Data Structures
The method relies on specific texture layouts to facilitate GPU-based iteration [S1, unofficial]:

1.  **Incident-Direction Texture:** A 2D texture.
    *   **Indexing:** Column $k \in \{1, \dots, N\}$ (SVD term index), Row $i \in \{1, \dots, D\}$ (direction index).
    *   **Content:** Stores values $\xi_k u_k(\omega_0)$. Note: The source describes indexing by direction index $i$ in the row, but the value depends on $\omega_0$. In the context of the fragment program, this texture is sampled using the current outgoing direction $\omega_0$ to retrieve the appropriate coefficient for term $k$.
2.  **Outgoing-Direction Texture:** A 2D texture.
    *   **Indexing:** Column $k \in \{1, \dots, N\}$, Row $i \in \{1, \dots, D\}$.
    *   **Content:** Stores values $\xi_k v_k(\omega_i)$.
3.  **Radiance Texture:** A flat 3D texture (implemented as a 1D texture with 3D coordinates or a 3D texture object).
    *   **Content:** Stores $L_n(x, \omega_d)$ for the current iteration $n$.
4.  **Temporary Outgoing Texture:** A flat 3D texture.
    *   **Content:** Accumulates $\sum_{d=1}^{D} L_{n+1}(x, \omega_d) \xi_k v_k(\omega_d)$ for each $k$.

### Formulas
The differential transport equation for radiance $L$ is [S1, unofficial]:

$$
\frac{dL(x, \omega_0)}{dx} = \kappa_a(x)L_e(x, \omega_0) - \kappa_a(x)L(x, \omega_0) - \kappa_s(x)L(x, \omega_0) + \frac{\kappa_s(x)}{4\pi} \int_{\Omega} L(x, \omega_i)p(\omega_0, \omega_i)d\sigma_{\omega_i}
$$

Where:
*   $\kappa_a(x)$ is the absorption coefficient.
*   $\kappa_s(x)$ is the scattering coefficient.
*   $\kappa_t(x) = \kappa_a(x) + \kappa_s(x)$ is the extinction coefficient.
*   $L_e(x, \omega_0)$ is the emission density.
*   $p(\omega_0, \omega_i)$ is the phase function.

The Henyey-Greenstein (HG) phase function is defined as [S1, unofficial]:

$$
p(\theta) = \frac{1}{4\pi} \frac{1-g^2}{(1+g^2-2g\cos\theta)^{3/2}}
$$

Where $g \in (-1, 1)$ is the asymmetry parameter and $\theta$ is the angle between incident direction $\omega_i$ and outgoing direction $\omega_0$.

The SVD approximation of the phase function is [S1, unofficial]:

$$
p(\omega_0, \omega_i) \approx \sum_{k=1}^{N} \xi_k u_k(\omega_0) v_k(\omega_i)
$$

Where $N$ is the number of retained terms, $\xi_k$ are the singular values, and $u_k, v_k$ are the left and right singular vectors, respectively.

The discrete transport equation, solved iteratively, is [S1, unofficial]:

$$
L_{n+1}(x+dx, \omega_0) \approx L_n(x, \omega_0) + \kappa_a(x)L_e(x, \omega_0)dx - \kappa_t(x)L_n(x, \omega_0)dx + \frac{\kappa_s(x)}{4\pi D} \sum_{d=1}^{D} L_n(x, \omega_d) p(\omega_0, \omega_d)dx
$$

Substituting the SVD approximation into the in-scattering term transforms the equation to [S1, unofficial]:

$$
L_{n+1}(x+dx, \omega_0) \approx L_n(x, \omega_0) + \kappa_a(x)L_e(x, \omega_0)dx - \kappa_t(x)L_n(x, \omega_0)dx + \frac{\kappa_s(x)}{4\pi} \sum_{k=1}^{N} \xi_k u_k(\omega_0) \left[ \sum_{d=1}^{D} L_n(x, \omega_d) \xi_k v_k(\omega_d) \right] dx
$$

### Parameters
*   **$N$ (SVD Terms):** Number of terms in the SVD approximation. Typical values are 8 or 24. For $g=0.5$, 24 terms yield approximately 10% RMSE [S1, unofficial]. Higher $g$ requires more terms.
*   **$D$ (Directions):** Number of discrete directions used to discretize the angular domain. Typical values are 128 or 256 [S1, unofficial].
*   **$g$ (Anisotropy):** The asymmetry factor of the HG phase function. Typical value is 0.5 [S1, unofficial].
*   **Albedo:** Single-scattering albedo $\kappa_s / \kappa_t$. Typical value is 0.9 [S1, unofficial].
*   **$dx$:** Step size along the ray.

### Pseudocode

The implementation uses a fragment program executed $D$ times per iteration step. The following pseudocode describes the logic within the fragment program for a single direction $d$ and the accumulation logic.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Constants
N = 24  # Number of SVD terms
D = 128 # Number of directions
dx = 1.0 # Step size (normalized)

# Textures (accessed via GPU texture sampling)
# IncidentTexture[k, dir_idx] -> stores xi_k * u_k(w_0)
# OutgoingTexture[k, dir_idx] -> stores xi_k * v_k(w_i)
# DensityTexture[x, y, z] -> stores density rho(x)
# RadianceTexture[x, y, z, dir_idx] -> stores L_n(x, w_d)
# TempOutgoingTexture[x, y, z, k] -> stores sum_d L_{n+1}(x, w_d) * xi_k * v_k(w_d)

def fragment_program(current_direction_index d, current_iteration n):
    # 1. Determine the current outgoing direction w_0 and incoming direction w_d
    # Note: In the source, the update is for L_{n+1}(x, w_0) using L_n(x, w_d).
    # The fragment program is invoked D times, once for each d.
    # w_0 is the direction for which we are computing the new radiance.
    # w_d is the current direction in the loop.
    
    w_0 = get_direction_from_index(d) # Or fixed w_0 depending on shader structure
    w_d = get_direction_from_index(d) # The source implies the loop iterates d
    
    # Retrieve physical coefficients from Density Texture
    # Assuming density rho is stored, kappa_a and kappa_s are derived
    rho = sample_3d_texture(DensityTexture, current_position)
    kappa_a = rho * absorption_coefficient_constant # [G] Constant assumed
    kappa_s = rho * scattering_coefficient_constant # [G] Constant assumed
    kappa_t = kappa_a + kappa_s
    
    # Retrieve emission
    L_e = sample_emission(current_position, w_0)
    
    # Retrieve previous radiance L_n(x, w_0)
    # Note: The source updates L_{n+1}(x, w_0). 
    # In a typical GPU implementation, we might read L_n(x, w_0) from a previous pass or texture.
    # However, the source describes updating L_{n+1} using L_n.
    # Let's assume L_n(x, w_0) is available.
    L_n_w0 = sample_radiance(RadianceTexture, current_position, w_0)
    
    # Retrieve the precomputed SVD coefficient for w_0 and term k
    # This is done for each k in the summation. 
    # The source structure suggests the fragment program handles one d, 
    # but the in-scattering sum over k is computed.
    # To avoid loops in fragment shaders, the summation over k might be unrolled or handled via multiple passes.
    # The source says: "Update L_{n+1} ... with L_n and sum_{d} L_n xi v".
    # And "Accumulate L_{n+1} xi v to Temp Outgoing".
    
    # Calculate the in-scattering contribution for this d
    # We need sum_{k=1}^N xi_k u_k(w_0) * [sum_{d'} L_n(x, w_{d'}) xi_k v_k(w_{d'})]
    # The term in brackets is stored in OutgoingTexture[k] from the PREVIOUS iteration or accumulated in TempOutgoing.
    # The source states: "Outgoing Texture contains the value of sum_{d} L_n(x, w_d) xi_k v_k(w_d)".
    
    in_scattering_sum = 0.0
    for k in range(1, N + 1):
        # Retrieve xi_k * u_k(w_0) from Incident Texture
        # Indexing: column k, row corresponding to w_0
        u_coeff = sample_2d_texture(IncidentTexture, k, get_direction_index(w_0))
        
        # Retrieve sum_{d} L_n(x, w_d) xi_k v_k(w_d) from Outgoing Texture
        # This value was computed in the previous iteration's loop over d
        outgoing_sum = sample_3d_texture(OutgoingTexture, current_position, k)
        
        in_scattering_sum += u_coeff * outgoing_sum
        
    # Update Radiance
    # L_{n+1}(x, w_0) = L_n(x, w_0) + kappa_a * L_e * dx - kappa_t * L_n(x, w_0) * dx + (kappa_s / 4pi) * in_scattering_sum * dx
    L_n_plus_1_w0 = L_n_w0 + kappa_a * L_e * dx - kappa_t * L_n_w0 * dx + (kappa_s / (4.0 * 3.14159265)) * in_scattering_sum * dx
    
    # Write L_{n+1}(x, w_0) to Radiance Texture (or a temporary buffer for next iteration)
    write_radiance(RadianceTexture, current_position, w_0, L_n_plus_1_w0)
    
    # Accumulate for the NEXT iteration's Outgoing Texture
    # We need to compute sum_{d} L_{n+1}(x, w_d) xi_k v_k(w_d)
    # This fragment program is called for each d. So we accumulate into TempOutgoing.
    for k in range(1, N + 1):
        # Retrieve xi_k * v_k(w_d) from Outgoing Texture
        # Indexing: column k, row corresponding to w_d
        v_coeff = sample_2d_texture(OutgoingTexture, k, get_direction_index(w_d))
        
        # L_{n+1}(x, w_d) is the value we just computed (or if w_d != w_0, we need to handle that)
        # The source says: "Accumulate L_{n+1}(x, w_d) xi_k v_k(w_d)".
        # If the fragment program is structured such that it updates L_{n+1} for a specific w_0,
        # and then accumulates for the current d, there is a dependency.
        # The source Figure 5a shows:
        # 1. Update L_{n+1}(x, w_0)
        # 2. Accumulate L_{n+1}(x, w_d) xi_k v_k(w_d)
        # This implies that for the current d, we use the newly computed L_{n+1} if d corresponds to the w_0 being updated,
        # or we might need to read L_{n+1} for other directions.
        # However, the source says "L_{n+1}(x, w_0) in direction d ... is updated".
        # This suggests the fragment program is specialized for a specific w_0, and iterates d.
        # But the accumulation step uses L_{n+1}(x, w_d).
        # If w_d is the current direction in the loop, and we are updating L_{n+1}(x, w_0),
        # then L_{n+1}(x, w_d) is only available if d == 0 (or the specific direction being updated).
        # The source text is slightly ambiguous here. 
        # "Accumulate L_{n+1}(x, w_d) xi_k v_k(w_d) to Temporary Outgoing Texture".
        # If the loop is over d, and we are computing L_{n+1}(x, w_0) for a FIXED w_0,
        # then L_{n+1}(x, w_d) for d != w_0 is not yet computed in this pass.
        # However, the source says "L_{n+1}(x, w_0) in all D directions has been updated" after d reaches D.
        # This implies the fragment program is run D times, each time for a different w_0?
        # No, "For each iteration step n, loop through discrete directions d from 1 to D."
        # "Update Radiance: ... update L_{n+1}(x, w_0) ... using L_n(x, w_0) and incoming scattering from all directions."
        # This implies L_{n+1}(x, w_0) is computed using the FULL sum over d.
        # So the fragment program computes L_{n+1}(x, w_0) for ONE w_0, using the pre-computed Outgoing Texture.
        # Then, how is the NEW Outgoing Texture for the NEXT iteration computed?
        # "Accumulate L_{n+1}(x, w_d) xi_k v_k(w_d)".
        # This suggests that after L_{n+1} is fully computed for all directions (which takes D passes?),
        # we then compute the new Outgoing Texture.
        # But the source says "During each pass of fragment program... L_{n+1} is updated... AND sum is calculated."
        # This implies the fragment program does BOTH.
        # If the fragment program is invoked for direction d, it updates L_{n+1}(x, w_d)?
        # Let's re-read: "L_{n+1}(x, w_0) in direction d ... is updated".
        # This phrasing suggests w_0 is the direction associated with the current pass d.
        # So, Pass d updates L_{n+1}(x, w_d).
        # And it also accumulates L_{n+1}(x, w_d) into TempOutgoing.
        
        # Revised Logic based on "L_{n+1}(x, w_0) in direction d":
        # The fragment program is executed D times.
        # In iteration d, we compute L_{n+1}(x, w_d).
        # To compute L_{n+1}(x, w_d), we need the in-scattering sum from L_n(x, w_{all}).
        # This sum is available in OutgoingTexture (from iteration n).
        
        # So:
        # 1. Identify w_d for this pass.
        # 2. Compute L_{n+1}(x, w_d) using OutgoingTexture (which holds sum_{d'} L_n(x, w_{d'}) xi_k v_k(w_{d'})).
        # 3. Accumulate L_{n+1}(x, w_d) xi_k v_k(w_d) into TempOutgoingTexture.
        
        pass # Logic handled below

def render_iteration(n):
    # Initialize TempOutgoingTexture to 0
    clear_texture(TempOutgoingTexture)
    
    # Loop over D directions
    for d in range(1, D + 1):
        w_d = get_direction_from_index(d)
        
        # 1. Compute L_{n+1}(x, w_d)
        # Retrieve L_n(x, w_d)
        L_n_wd = sample_radiance(RadianceTexture, current_position, w_d)
        
        # Retrieve coefficients for w_d
        # Incident Texture stores xi_k u_k(w_0). Here w_0 = w_d.
        # Outgoing Texture stores xi_k v_k(w_i). Here we need the sum over d' of L_n(x, w_{d'}) xi_k v_k(w_{d'}).
        # This sum is stored in OutgoingTexture[k] from the previous iteration.
        
        in_scattering_sum = 0.0
        for k in range(1, N + 1):
            u_coeff = sample_2d_texture(IncidentTexture, k, get_direction_index(w_d))
            outgoing_sum = sample_3d_texture(OutgoingTexture, current_position, k)
            in_scattering_sum += u_coeff * outgoing_sum
            
        rho = sample_3d_texture(DensityTexture, current_position)
        kappa_a = rho * absorption_coefficient_constant
        kappa_s = rho * scattering_coefficient_constant
        kappa_t = kappa_a + kappa_s
        L_e = sample_emission(current_position, w_d)
        
        L_n_plus_1_wd = L_n_wd + kappa_a * L_e * dx - kappa_t * L_n_wd * dx + (kappa_s / (4.0 * 3.14159265)) * in_scattering_sum * dx
        
        # Write L_{n+1}(x, w_d) to a temporary radiance texture or update in place if careful
        write_radiance(RadianceTexture, current_position, w_d, L_n_plus_1_wd)
        
        # 2. Accumulate for next iteration's Outgoing Texture
        for k in range(1, N + 1):
            v_coeff = sample_2d_texture(OutgoingTexture, k, get_direction_index(w_d))
            # Add L_{n+1}(x, w_d) * xi_k * v_k(w_d) to TempOutgoing
            # Note: OutgoingTexture[k] stores xi_k v_k. We need to multiply by L_{n+1}.
            contribution = L_n_plus_1_wd * v_coeff
            add_to_texture(TempOutgoingTexture, current_position, k, contribution)
            
    # After D passes, TempOutgoingTexture contains sum_{d} L_{n+1}(x, w_d) xi_k v_k(w_d)
    # Swap TempOutgoingTexture with OutgoingTexture
    swap_textures(OutgoingTexture, TempOutgoingTexture)
```

### Edge Cases and Pitfalls
*   **Approximation Error:** The SVD approximation generally yields smaller reconstructed values than the original phase function, leading to darker rendered images, especially in thick parts of the medium [S1, unofficial].
*   **Convergence in Dynamic Scenes:** For highly dynamic scenes, the transport equation may not converge before the scene changes, causing errors to accumulate [S1, unofficial].
*   **Precision:** The method relies on flat 3D textures. Precision issues may arise from texture sampling interpolation if not handled carefully [G].
*   **Boundary Conditions:** The method assumes a volume grid. Boundary conditions for the volume must be defined, typically as zero radiance outside the volume [G].

## Operating characteristics

**Time Complexity:**
*   Per iteration: $O(D \cdot N \cdot V)$, where $V$ is the number of voxels.
*   The SVD approximation reduces the complexity of the in-scattering integral from $O(D^2)$ to $O(D \cdot N)$ per voxel per iteration, assuming $N \ll D$ [S1, unofficial].

**Memory Complexity:**
*   $O(V \cdot D)$ for the Radiance Texture.
*   $O(V \cdot N)$ for the Outgoing and Temporary Outgoing Textures.
*   $O(N \cdot D)$ for the Incident and Outgoing Direction Textures (negligible compared to volume textures).

**Measured Performance:**
*   On an NV7800GT graphics card with 64x64x64 volume resolution and $D=128$, the scene is simulated at 2fps [S1, unofficial].
*   With $D=256$, 1.5fps is achieved [S1, unofficial].
*   Compared to the unfactorized model (no SVD), this method achieves a rendering speed of 1.5fps vs 0.2fps on the same hardware (NV7800GT, 64x64x64 resolution, D=256) [S1, unofficial].

**Strengths and Limitations:**
*   **Strengths:** Interactive rendering rates; handles dynamic scenes; little constraints on lighting conditions compared to other near real-time algorithms like Harris (2001) and Sloan (2002) [S1, unofficial].
*   **Limitations:** Approximation error leads to darker images; convergence issues in highly dynamic scenes; accuracy depends on $N$ and $g$ [S1, unofficial].

**When to Use:**
*   Use for interactive rendering of multiple scattering in participating media where dynamic scenes are required and high accuracy is not critical.
*   Avoid for highly dynamic scenes where convergence cannot be achieved, or when precise physical accuracy of the phase function is required.

## Background

**Origin:**
The method was proposed by Gong Z., Ren Z., Wang Z., and Peng Q. in 2007 [S1, unofficial]. It is based on the application of Singular Value Decomposition (SVD) to the scattering phase function in the transport equation.

**History and Development:**
Separable decomposition techniques were introduced by Kautz (1999) for BRDF rendering. Wang (2004) and Liu (2004) applied SVD to glossy BRDFs in PRT. Wang (2005) introduced SVD to the approximation of BSSRDFs by separating the Henyey-Greenstein phase function [S1, unofficial]. This method extends these ideas to interactive multiple scattering in participating media.

**Usage:**
Implemented in a Direct3D/HLSL environment [S1, unofficial].

## Sources

[S1] Interactive Rendering of Multiple Scattering in Participating Media Using Separable Phase Function, https://www.scitepress.org/PublishedPapers/2007/20840/, fetched 2026-10-30 (unofficial)
