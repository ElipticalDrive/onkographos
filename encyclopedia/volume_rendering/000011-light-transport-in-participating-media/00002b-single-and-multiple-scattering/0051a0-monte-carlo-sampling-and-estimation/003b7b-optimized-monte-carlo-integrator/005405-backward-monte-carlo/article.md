# Backward Monte Carlo

Also called: Forward MC, Forward Monte Carlo, Forward Monte Carlo (FMC) with Local Estimation

## Overview
Backward Monte Carlo (BMC) is a photon tracking method where photons are launched from camera pixels back into the scene to estimate radiance, offering stability for in-situ (inside the medium) rendering compared to Forward Monte Carlo [S1, unofficial]. While Forward Monte Carlo (FMC) traces photons from the light source to the camera, BMC traces them from the detector to the source [S1, unofficial]. The key distinction is that BMC avoids the $|V_{s \to c}|^{-2}$ variance explosion inherent in FMC's local estimation when the camera is inside the scattering medium, because the sun is effectively at infinity [S1, unofficial].

## How it works
**Inputs:**
*   Camera position $X_c$ and pixel direction $\omega_p$.
*   Sun direction $\omega_{sun}$ (assumed constant and at infinity).
*   Volumetric extinction field $\beta(X)$, composed of aerosol and air components.
*   Aerosol number density $n$.
*   Aerosol scattering coefficient $\sigma_{aerosol}$.
*   Air scattering coefficient $\beta_{air}$.
*   Aerosol single scattering albedo $\$_{aerosol}$.
*   Air single scattering albedo $\$_{air} \approx 1$.
*   Anisotropy parameter $g$ for Henyey-Greenstein phase function.
*   Initial photon intensity $I_0$.
*   Number of initial photons $N_{photons}$ (e.g., $10^7$) [S1, unofficial].

**Outputs:**
*   Radiance estimate for pixel $p$ at camera $c$.

**Data Structures:**
*   **Ray:** Defined by origin $X$ and direction $\omega$.
*   **Voxel Grid:** Stores $\beta_{aerosol}$, $\beta_{air}$, and $n$ for spatial queries.
*   **Photon Packet:** Stores current position $X_s$, direction $\omega_s$, and intensity $I_s$.

**Parameters:**
*   $g$: Anisotropy parameter, range $-1 \le g \le 1$ [S1, unofficial].
*   $\$_{aerosol}$: Aerosol single scattering albedo [S1, unofficial].
*   $\$_{air}$: Air single scattering albedo, approximately 1 in the visible range [S1, unofficial].

**Steps:**

1.  **Initialization:** For each pixel $p$ at camera $c$, launch a photon-packet from $X_c$ in direction $-\omega_p$. This is the initial ray $R_0$. The packet has initial intensity $I_0$ [S1, unofficial].
2.  **Propagation Distance:** Find the distance $l_{random}$ on ray $R_s$ to which the photon-packet propagates. Calculate optical depth $\tau_{random} = -\ln(1-u)$ where $u \sim U[0,1]$. Numerically seek $l_{random}$ such that:
    $$ \int_0^{l_{random}} (\sigma_{aerosol} n + \beta_{air}) dl = \tau_{random} $$
    The 3D position is $X_s$ [S1, unofficial].
3.  **Boundary Check:** If $X_s$ is outside the domain, terminate the packet. If the ray $R_s$ is parallel to $\omega_{sun}$ (i.e., the photon back-traces into the Sun), the packet is counted as contributing to pixel $p$ [S1, unofficial].
4.  **Particle Selection:** If $X_s$ is inside the domain, randomly determine the particle type (air molecule or aerosol) based on relative extinction coefficients ($\beta_{air}$ vs $\beta_{aerosol}$) at the voxel containing $X_s$ [S1, unofficial].
5.  **Intensity Update:**
    *   If the particle is an aerosol, attenuate intensity: $I_{s+1} = \$_{aerosol} I_s$.
    *   For air molecules, intensity is maintained ($I_{s+1} = I_s$).
    *   If $I_{s+1}$ is lower than a threshold, stochastically terminate the packet [S1, unofficial].
6.  **Scattering:** Scatter the photon to a new random direction using inverse transform sampling based on the phase function of the particle.
    *   **Aerosol (Henyey-Greenstein):**
        $$ \Phi_{scatter} = \arccos\left( \frac{1+g^2 - \left(\frac{1-g^2}{1-g+2gu}\right)^2}{2g} \right) $$
        Sample azimuth from $U[0, 2\pi]$ [S1, unofficial].
    *   **Molecule (Rayleigh):**
        $$ \Phi_{scatter} = \arccos(\gamma^3 - \gamma - \sqrt{3}) $$
        where $\gamma = \sqrt{4u - 2 + (4u - 2)^2 + 1}$ [S1, unofficial].
    *   Trace new ray $R_{s+1}$ and repeat step 2 [S1, unofficial].
7.  **Local Estimation:** At every scattering event (step 6), calculate contribution $W_{le}$ to pixel $p$.
    *   Let $V_{s \to sun}$ be the vector from scattering point $X_s$ to the Top of Atmosphere (TOA), directed to $-\omega_{sun}$.
    *   Let $t_{s \to sun}$ be the transmittance along $V_{s \to sun}$.
    *   Let $\Phi_{R_s, sun}$ be the angle between $R_s$ and $V_{s \to sun}$.
    *   Let $\$$ and $P$ be the albedo and phase function of the particle.
    *   Contribution:
        $$ W_{le} = \$ I_s P(\Phi_{R_s, sun}) t_{s \to sun} $$
    *   Note: There is no $|V_{s \to sun}|^{-2}$ factor because the Sun is at infinity [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import random

def sample_henyey_greenstein(g, u):
    """Sample scattering angle for aerosol using Henyey-Greenstein phase function."""
    # Eq. (11) from [S1, unofficial]
    # Phi_scatter = arccos( (1 + g^2 - ((1 - g^2) / (1 - g + 2*g*u))^2) / (2*g) )
    # Note: The source formula in [S1, unofficial] Eq 11 is:
    # arccos( (1+g^2 - ((1-g^2)/(1-g+2gu))^2) / 2g )
    # However, standard HG sampling usually involves:
    # cos(theta) = (1 + g^2 - ((1-g^2)/(1-g+2gu))^2) / 2g is NOT the standard form.
    # Standard form: cos(theta) = (1 + g^2 - ((1-g^2)/(1-g+2gu))^2) / 2g is derived from inverse CDF.
    # Let's stick strictly to the provided text Eq 11:
    # Phi = arccos( (1 + g^2 - ( (1 - g**2) / (1 - g + 2 * g * u) )**2 ) / (2 * g) )
    
    # Re-reading Eq 11 carefully:
    # Phi_scatter = arccos( (1 + g^2 - ( (1 - g^2) / (1 - g + 2gu) )^2 ) / 2g )
    # Wait, the text says:
    # Phi_scatter = arccos( (1 + g^2 - ( (1 - g^2) / (1 - g + 2gu) )^2 ) / 2g )
    # Actually, looking at the raw text:
    # arccos( (1+g^2 - ((1-g^2)/(1-g+2gu))^2) / 2g )
    # Let's implement exactly as written in Eq 11.
    
    term = (1 - g**2) / (1 - g + 2 * g * u)
    cos_theta = (1 + g**2 - term**2) / (2 * g)
    # Clamp for numerical stability
    cos_theta = max(-1.0, min(1.0, cos_theta))
    return math.acos(cos_theta)

def sample_rayleigh(u):
    """Sample scattering angle for air molecules using Rayleigh phase function."""
    # Eq. (12) from [S1, unofficial]
    # gamma = sqrt(4u - 2 + (4u - 2)^2 + 1)
    # Phi_scatter = arccos(gamma^3 - gamma - sqrt(3))
    
    val = 4 * u - 2
    gamma = math.sqrt(val + val**2 + 1)
    cos_theta = gamma**3 - gamma - math.sqrt(3)
    # Clamp for numerical stability
    cos_theta = max(-1.0, min(1.0, cos_theta))
    return math.acos(cos_theta)

def backward_monte_carlo(camera_pos, pixel_dir, sun_dir, voxel_grid, num_photons, g, sigma_aerosol, n_aerosol, beta_air, albedo_aerosol, threshold):
    """
    Perform Backward Monte Carlo simulation.
    
    Args:
        camera_pos: 3D vector X_c
        pixel_dir: 3D vector omega_p
        sun_dir: 3D vector omega_sun
        voxel_grid: Object providing get_extinction(x) -> (sigma_aerosol * n, beta_air)
        num_photons: Number of photons to trace
        g: Anisotropy parameter
        sigma_aerosol: Aerosol scattering coefficient
        n_aerosol: Aerosol number density
        beta_air: Air scattering coefficient
        albedo_aerosol: Aerosol single scattering albedo
        threshold: Intensity threshold for Russian Roulette
        
    Returns:
        radiance_estimate: Float
    """
    total_contribution = 0.0
    
    # Initial intensity
    I_0 = 1.0
    
    for _ in range(num_photons):
        # Step 1: Launch photon from camera
        current_pos = camera_pos
        current_dir = -pixel_dir # Direction -omega_p
        current_intensity = I_0
        
        while True:
            # Step 2: Find propagation distance
            # Calculate tau_random
            u = random.random()
            tau_random = -math.log(1 - u)
            
            # Numerically seek l_random such that integral of extinction = tau_random
            # This requires a solver or lookup table on the voxel grid.
            # For pseudocode, we assume a function solve_distance(ray, tau) exists.
            l_random = solve_distance(current_pos, current_dir, tau_random, voxel_grid)
            
            # Update position
            next_pos = current_pos + l_random * current_dir
            
            # Step 3: Boundary Check
            if not voxel_grid.is_inside(next_pos):
                break # Terminated
            
            # Check if ray is parallel to sun (back-traces into sun)
            # This is a simplified check; in practice, check if next_pos is "in the sun"
            if is_parallel_to_sun(current_dir, sun_dir):
                total_contribution += current_intensity
                break
            
            # Step 4: Particle Selection
            ext_aerosol, ext_air = voxel_grid.get_extinction(next_pos)
            total_ext = ext_aerosol + ext_air
            
            if total_ext == 0:
                break # No scattering medium
                
            # Probability of aerosol
            prob_aerosol = ext_aerosol / total_ext
            is_aerosol = random.random() < prob_aerosol
            
            # Step 5: Intensity Update
            if is_aerosol:
                current_intensity *= albedo_aerosol
            else:
                # Air molecule: intensity maintained
                pass
                
            if current_intensity < threshold:
                # Russian Roulette
                if random.random() > current_intensity:
                    break
                current_intensity /= current_intensity # Keep intensity, but this logic is tricky. 
                # Usually: if survive, intensity = current_intensity / probability_of_survival.
                # Source says "stochastically terminated". 
                # Standard RR: p = current_intensity. If survive, I_new = I_old / p.
                # Here p = current_intensity (assuming max I=1).
                current_intensity = 1.0 # Reset to 1 if survived? Or keep relative?
                # Source doesn't specify RR weight adjustment, only termination. 
                # Assuming standard RR for unbiasedness:
                # current_intensity /= current_intensity # This makes it 1.0.
                # Let's assume standard RR:
                # current_intensity = 1.0 
                pass

            # Step 6: Scattering
            u = random.random()
            if is_aerosol:
                phi_scatter = sample_henyey_greenstein(g, u)
            else:
                phi_scatter = sample_rayleigh(u)
            
            # Sample azimuth
            azimuth = random.uniform(0, 2 * math.pi)
            
            # Construct new direction
            # This requires rotating current_dir by phi_scatter around a random axis perpendicular to current_dir
            # For simplicity, we assume a helper function exists
            new_dir = scatter_direction(current_dir, phi_scatter, azimuth)
            
            # Step 7: Local Estimation
            # Vector from scattering point to TOA directed to -omega_sun
            # V_s->sun is along -omega_sun
            # Transmittance t_s->sun
            t_s_sun = compute_transmittance(next_pos, -sun_dir, voxel_grid)
            
            # Angle between R_s (current_dir) and V_s->sun (-sun_dir)
            # cos(phi) = dot(current_dir, -sun_dir)
            cos_phi = dot(current_dir, -sun_dir)
            cos_phi = max(-1.0, min(1.0, cos_phi))
            phi_rs_sun = math.acos(cos_phi)
            
            # Phase function
            if is_aerosol:
                # Henyey-Greenstein
                # P_HG = (1 / (4*pi)) * (1 - g^2) / (1 + g^2 - 2*g*cos(phi))^(3/2)
                denom = 1 + g**2 - 2 * g * cos_phi
                if denom <= 0: denom = 1e-9
                phase = (1 - g**2) / (4 * math.pi * (denom ** 1.5))
                albedo = albedo_aerosol
            else:
                # Rayleigh
                # P_Ray = (3 / (16*pi)) * (1 + cos^2(phi))
                phase = (3 / (16 * math.pi)) * (1 + cos_phi**2)
                albedo = 1.0 # $air approx 1
            
            # Contribution
            w_le = albedo * current_intensity * phase * t_s_sun
            total_contribution += w_le
            
            # Update for next iteration
            current_pos = next_pos
            current_dir = new_dir
            current_intensity = current_intensity # Already updated in step 5
            
    # Average contribution
    return total_contribution / num_photons

def dot(v1, v2):
    return v1[0]*v2[0] + v1[1]*v2[1] + v1[2]*v2[2]

def is_parallel_to_sun(dir, sun_dir):
    # Check if dir is parallel to sun_dir (back-tracing into sun)
    # In practice, this might mean hitting a specific boundary or direction cone
    return False # Placeholder for specific geometry check
```

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of photons and the number of scattering events per photon. Each photon trace involves numerical integration for distance sampling and phase function evaluation.
*   **Memory Complexity:** Depends on the voxel grid resolution storing extinction coefficients.
*   **Performance:** BMC is stable for in-situ rendering, whereas FMC becomes unstable due to the $|V_{s \to c}|^{-2}$ factor in local estimation when the camera is inside the medium [S1, unofficial].
*   **Strengths:** Stability for in-situ cameras; no variance explosion near the camera [S1, unofficial].
*   **Limitations:** Estimates radiance for one camera and one pixel at a time, making it less efficient for multi-view simulations compared to FMC with local estimation, which can contribute to multiple pixels in parallel [S1, unofficial].
*   **Scaling:** Requires $10^7$ initial photons for stable results in typical atmospheric simulations [S1, unofficial].

## Background
Backward Monte Carlo (BMC) was proposed as an alternative to Forward Monte Carlo (FMC) for in-situ multi-scattering tomography and atmospheric rendering [S1, unofficial]. It addresses the instability of FMC when the detector is inside the scattering medium by reversing the photon path and using a local estimation that does not suffer from the inverse-square law singularity associated with the camera position [S1, unofficial]. The method is described in the context of simulating sky images and atmospheric tomography [S1, unofficial].

## Sources
[S1] In-situ multi-scattering tomography, https://archive.org/details/arxiv-1512.02110, fetched 2026-10-27 (unofficial)
