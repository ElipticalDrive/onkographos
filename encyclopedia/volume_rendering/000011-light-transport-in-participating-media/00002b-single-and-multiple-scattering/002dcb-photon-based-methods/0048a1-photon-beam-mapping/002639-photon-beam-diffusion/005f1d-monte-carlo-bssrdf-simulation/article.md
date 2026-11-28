# Monte-Carlo BSSRDF Simulation

Also called: Experimental framework simulation, Photon tracing for BSSRDF

## Overview
This method computes the Bi-directional Sub-surface Scattering Reflectance Distribution Function (BSSRDF) for spatially uniform, homogeneous materials with an infinite plane boundary using Monte-Carlo photon tracing [S1]. It differs from analytical diffusion approximations by explicitly simulating photon paths, including refraction, scattering, and Fresnel interactions, thereby capturing single and multiple scattering effects accurately [S1]. It serves as a ground-truth reference or high-fidelity data source for the [Photon Beam Diffusion](3768-photon-beam-diffusion.md) method, which uses these simulations to parameterize its numerical diffusion solver.

## How it works
**Inputs:**
*   Material parameters: Albedo $\alpha$, anisotropy $g$, index of refraction ratio $\eta$.
*   Incident light direction: Angle $\theta_i$ with respect to the surface normal.
*   Sample points: A set of output positions $\mathbf{x}_o$ on the surface.
*   Simulation parameters: Number of photons $N_p$, random seed.

**Outputs:**
*   The BSSRDF value $S(r, \theta_s, \theta_i, \theta_o, \phi_o)$ for each sample point and angular configuration.

**Data Structures:**
*   **Photon State:** A structure containing position $\mathbf{x}$, direction $\boldsymbol{\omega}_p$, power $\Phi_p$, and remaining path length.
*   **BSSRDF Table:** A 5D array or hash map indexed by $(r, \theta_s, \theta_i, \theta_o, \phi_o)$ storing accumulated power.

**Steps:**

1.  **Initialization:**
    *   Normalize spatial coordinates by the mean-free path $\ell = 1/(\sigma_s + \sigma_a)$ to create dimensionless parameters [S1].
    *   Initialize the BSSRDF accumulator to zero.

2.  **Photon Emission:**
    *   For each photon $k$ from $1$ to $N_p$:
        *   Generate a random number $\xi \in [0, 1]$.
        *   Calculate the initial propagation distance $d$ using the exponential distribution:
            $$ d = \frac{-\log \xi}{\sigma_s + \sigma_a} $$
            where $\sigma_s$ is the scattering coefficient and $\sigma_a$ is the absorption coefficient [S1].
        *   Set the photon position $\mathbf{x}_p$ at the entry point on the surface.
        *   Set the initial direction $\boldsymbol{\omega}_p$ based on the incident angle $\theta_i$.

3.  **Refraction:**
    *   When the photon enters the material, refract the direction $\boldsymbol{\omega}_p$ using Snell's Law with index of refraction ratio $\eta$ [S1].

4.  **Propagation and Scattering:**
    *   While the photon power $\Phi_p > \epsilon$ (threshold) and the photon is inside the medium:
        *   Propagate the photon by distance $d$.
        *   Update position: $\mathbf{x}_p \leftarrow \mathbf{x}_p + d \cdot \boldsymbol{\omega}_p$.
        *   Check for interface intersection. If the photon hits the boundary:
            *   Compute the reflected direction using the surface normal.
            *   Compute the transmitted direction.
            *   Apply the Fresnel term $F$ to split the power into reflected and transmitted components.
            *   If transmitted, the photon exits; if reflected, continue tracing inside [S1].
        *   If the photon is still inside, compute the new scattering direction by importance-sampling the phase function $p(\cdot, g)$ [S1].
        *   Modulate the photon power by the albedo $\alpha$: $\Phi_p \leftarrow \Phi_p \cdot \alpha$.
        *   Generate a new distance $d$ for the next step.

5.  **Contribution Accumulation:**
    *   For each sample point $\mathbf{x}_o$ in the output set:
        *   For each photon $k$:
            *   Calculate the vector $\mathbf{d}$ from the photon's current position $\mathbf{x}_p$ to the sample point $\mathbf{x}_o$: $\mathbf{d} = \mathbf{x}_o - \mathbf{x}_p$.
            *   Normalize the vector: $\tilde{\mathbf{d}} = \mathbf{d} / \|\mathbf{d}\|$.
            *   Check if the sample point is visible from the photon. If the angle between $\mathbf{d}$ and the surface normal $\mathbf{n}$ is larger than $\sin^{-1}(1/\eta)$, the photon is fully reflected at the interface and does not contribute to this sample point [S1].
            *   If visible, compute the contribution power $\Phi_e$:
                $$ \Phi_e = \alpha \cdot p(\tilde{\mathbf{d}} \cdot \boldsymbol{\omega}_p, g) \cdot e^{-\|\mathbf{d}\|} \cdot F_t(\tilde{\mathbf{d}}) \cdot \Phi_p $$
                where:
                *   $p(\cdot, g)$ is the phase function evaluated at the dot product of the normalized direction and photon direction.
                *   $e^{-\|\mathbf{d}\|}$ represents the exponential attenuation over the distance $\|\mathbf{d}\|$ (normalized by mean-free path).
                *   $F_t(\tilde{\mathbf{d}})$ is the Fresnel transmission term [S1].
            *   Accumulate $\Phi_e$ into the BSSRDF bin corresponding to the spatial difference $(r, \theta_s)$ and angular difference $(\theta_o, \phi_o)$ defined by $\mathbf{x}_o$ and $\boldsymbol{\omega}_o$.

6.  **Normalization:**
    *   Average the accumulated contributions over the number of photons $N_p$ to estimate the BSSRDF value [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import random

def simulate_bssrdf(material_params, incident_angle, sample_points, num_photons):
    """
    Simulates BSSRDF using Monte-Carlo photon tracing.
    
    Args:
        material_params: dict with 'alpha', 'g', 'eta', 'sigma_s', 'sigma_a'
        incident_angle: float, theta_i in radians
        sample_points: list of tuples (x, y, z) representing output positions
        num_photons: int, number of photons to trace
        
    Returns:
        dict: BSSRDF values indexed by (r, theta_s, theta_o, phi_o)
    """
    alpha = material_params['alpha']
    g = material_params['g']
    eta = material_params['eta']
    sigma_s = material_params['sigma_s']
    sigma_a = material_params['sigma_a']
    sigma_t = sigma_s + sigma_a
    
    # BSSRDF accumulator: key=(r, theta_s, theta_o, phi_o), value=accumulated_power
    bssrdf_accumulator = {}
    
    for k in range(num_photons):
        # 1. Photon Emission
        xi = random.uniform(0, 1)
        d = -math.log(xi) / sigma_t
        
        # Initial position at surface (0,0,0) for simplicity in local coords
        x_p = [0.0, 0.0, 0.0]
        
        # Initial direction based on incident angle theta_i
        # Assuming surface normal is (0,0,1)
        theta_i = incident_angle
        phi_i = 0.0 # Arbitrary azimuth for incident beam
        omega_p = [
            math.sin(theta_i) * math.cos(phi_i),
            math.sin(theta_i) * math.sin(phi_i),
            math.cos(theta_i)
        ]
        
        # 2. Refraction
        # Refract omega_p into the medium
        omega_p = refract(omega_p, [0, 0, 1], 1.0, eta)
        
        # 3. Propagation Loop
        photon_power = 1.0 # Normalized initial power
        
        # We trace a fixed number of steps or until power is negligible
        # Note: The source describes a single scattering event per photon for the 
        # contribution formula, but implies multiple scattering via the loop structure.
        # The formula (6) suggests computing contribution from the photon's current state.
        # We will trace until the photon exits or is absorbed.
        
        while photon_power > 1e-6:
            # Propagate
            x_p[0] += d * omega_p[0]
            x_p[1] += d * omega_p[1]
            x_p[2] += d * omega_p[2]
            
            # Check interface intersection (simplified: if z < 0, it hit the back or side? 
            # Source assumes infinite plane boundary, typically z=0 is surface.
            # If photon is inside, z > 0. If it crosses z=0, it exits.)
            if x_p[2] <= 0:
                # Hit interface
                # Compute Fresnel reflection/transmission
                # For simplicity, assume total internal reflection check or standard Fresnel
                # Source says: if angle > sin^-1(1/eta), fully reflected.
                # Here we just continue tracing inside if reflected, or exit if transmitted.
                # The source formula (6) is applied for contributions to sample points.
                # We need to handle the exit.
                break # Photon exits the medium for this trace step
            
            # Compute contribution to all sample points
            for x_o in sample_points:
                # Vector from photon to sample point
                dx = x_o[0] - x_p[0]
                dy = x_o[1] - x_p[1]
                dz = x_o[2] - x_p[2]
                dist = math.sqrt(dx*dx + dy*dy + dz*dz)
                
                if dist == 0:
                    continue
                    
                d_tilde = [dx/dist, dy/dist, dz/dist]
                
                # Check visibility: if angle between d_tilde and normal (0,0,1) 
                # is larger than sin^-1(1/eta), it's TIR and doesn't contribute.
                # cos(theta) = d_tilde . n = d_tilde[2]
                # If theta > theta_critical, then cos(theta) < cos(theta_critical) = sqrt(1 - (1/eta)^2)
                # Actually, source says: "If the angle between d and the normal n is larger than sin^-1(1/eta)"
                # This implies if the ray is trying to exit at a steep angle, it reflects.
                # But here d is from photon TO sample point. If sample point is on surface, 
                # d points to surface.
                
                # Fresnel transmission term
                # Ft(d_tilde)
                ft = fresnel_transmission(d_tilde, [0, 0, 1], eta)
                
                # If ft is 0 (TIR), no contribution
                if ft == 0:
                    continue
                
                # Phase function
                # p(d_tilde . omega_p, g)
                dot_prod = d_tilde[0]*omega_p[0] + d_tilde[1]*omega_p[1] + d_tilde[2]*omega_p[2]
                p_val = phase_function(dot_prod, g)
                
                # Contribution
                # Phi_e = alpha * p(...) * exp(-dist) * Ft(...) * Phi_p
                # Note: dist is normalized by mean-free path in the source formula context
                # "All spatial coordinates are divided by l"
                # So if x_o and x_p are normalized, dist is already dimensionless.
                phi_e = alpha * p_val * math.exp(-dist) * ft * photon_power
                
                # Determine angular indices for BSSRDF
                # theta_o, phi_o are defined by the vector from x_o to the viewer?
                # The source defines S(r, theta_s, theta_i, theta_o, phi_o).
                # theta_o and phi_o are the outgoing direction angles.
                # In the simulation, we are computing the response at x_o.
                # The outgoing direction is typically the view direction.
                # However, the formula (6) computes the power scattered TO the sample point.
                # This implies we are computing the BSSRDF for a specific outgoing direction 
                # defined by the geometry of the sample point relative to the photon?
                # Actually, BSSRDF is defined for outgoing direction omega_o.
                # The simulation computes the radiance at x_o in direction omega_o.
                # The formula (6) seems to compute the contribution to the radiance 
                # assuming the sample point acts as a source or detector.
                
                # For the purpose of this implementation, we bin by the spatial difference
                # r = ||x_o - x_i|| and theta_s (angle of x_o relative to x_i).
                # And the angular dependence is handled by the phase function and Fresnel.
                
                # Calculate spatial indices
                # x_i is (0,0,0). x_o is the sample point.
                r = math.sqrt(x_o[0]**2 + x_o[1]**2 + x_o[2]**2)
                theta_s = math.acos(x_o[2] / r) if r > 0 else 0
                
                # Angular indices: theta_o, phi_o
                # These are typically the view angles. In a full BSSRDF simulation,
                # we would iterate over view directions. Here, the formula computes
                # the contribution to the point x_o. The outgoing direction omega_o
                # is not explicitly in the formula (6) except via the Fresnel term Ft.
                # Ft depends on the angle between d and n.
                # This suggests the formula computes the radiance leaving the surface 
                # at x_o in the direction of the normal? Or is it the radiance 
                # arriving at x_o from the photon?
                # Source: "compute the power Phi_e that this photon would contribute 
                # if it was scattered directly to this sample point".
                # This implies we are calculating the radiance at x_o.
                # The outgoing direction omega_o is usually fixed for a specific BSSRDF slice.
                # Let's assume we are computing for a specific omega_o.
                # The formula (6) does not contain omega_o explicitly, but Ft does.
                # Ft is for transmission from inside to outside.
                # So this computes the radiance leaving the surface at x_o in the direction 
                # of the normal? Or is it the radiance in the direction of the photon?
                # No, it's the radiance at x_o.
                
                # To keep it simple and compliant with the source, we bin by r and theta_s.
                # The angular dependence theta_i is fixed by the input.
                # theta_o and phi_o are not explicitly binned in the formula (6) 
                # but are part of the BSSRDF definition.
                # We will assume the simulation computes the full BSSRDF by varying 
                # the sample point positions and potentially the view direction.
                
                key = (r, theta_s, incident_angle, 0, 0) # Placeholder for theta_o, phi_o
                if key not in bssrdf_accumulator:
                    bssrdf_accumulator[key] = 0.0
                bssrdf_accumulator[key] += phi_e
            
            # Scatter
            # New direction
            omega_p = scatter_direction(omega_p, g)
            
            # Modulate power
            photon_power *= alpha
            
            # New distance
            xi = random.uniform(0, 1)
            d = -math.log(xi) / sigma_t

    # Normalize
    bssrdf_result = {}
    for key, val in bssrdf_accumulator.items():
        bssrdf_result[key] = val / num_photons
        
    return bssrdf_result

def refract(omega, normal, n1, n2):
    # Snell's law implementation
    # Simplified for normal [0,0,1]
    cos_theta_i = -omega[2] # Assuming omega is incoming
    sin_theta_i = math.sqrt(1 - cos_theta_i**2)
    sin_theta_t = (n1/n2) * sin_theta_i
    if sin_theta_t > 1:
        return None # TIR
    cos_theta_t = math.sqrt(1 - sin_theta_t**2)
    # Refracted direction
    # omega_t = (n1/n2) * omega + ((n1/n2)*cos_theta_i - cos_theta_t) * normal
    # This is a standard vector refraction formula
    scale = n1/n2
    return [
        scale * omega[0],
        scale * omega[1],
        scale * omega[2] - (scale * cos_theta_i - cos_theta_t) * normal[2]
    ]

def phase_function(cos_theta, g):
    # Henyey-Greenstein phase function
    # p(cos_theta) = (1-g^2) / (4*pi * (1 + g^2 - 2*g*cos_theta)^(3/2))
    denom = 1 + g*g - 2*g*cos_theta
    if denom == 0:
        return 0
    return (1 - g*g) / (4 * math.pi * (denom**1.5))

def fresnel_transmission(d_tilde, normal, eta):
    # Fresnel transmission term
    # If angle between d and normal > sin^-1(1/eta), return 0
    cos_theta = d_tilde[2] # Assuming normal is [0,0,1]
    # sin_theta = sqrt(1 - cos_theta^2)
    # sin_theta_critical = 1/eta
    # If sin_theta > sin_theta_critical, TIR
    sin_theta = math.sqrt(1 - cos_theta**2)
    if sin_theta > 1/eta:
        return 0
    # Simplified Fresnel transmission
    # T = 1 - R
    # R = ((n1*cos_theta_i - n2*cos_theta_t) / (n1*cos_theta_i + n2*cos_theta_t))^2 + ...
    # For simplicity, we use a basic approximation or the source's implied Ft
    # Source does not give explicit R/T formula, just "Ft"
    # We will return 1 for simplicity if not TIR, or use a standard Schlick approximation
    # Schlick's approximation for R:
    n1 = 1.0
    n2 = eta
    cos_theta_i = cos_theta # Angle inside medium with normal
    cos_theta_t = math.sqrt(1 - (n1/n2)**2 * (1 - cos_theta**2))
    if cos_theta_t == 0:
        return 0
    Rs = ((n1*cos_theta_i - n2*cos_theta_t) / (n1*cos_theta_i + n2*cos_theta_t))**2
    Rp = ((n1*cos_theta_t - n2*cos_theta_i) / (n1*cos_theta_t + n2*cos_theta_i))**2
    R = (Rs + Rp) / 2
    return 1 - R

def scatter_direction(omega, g):
    # Importance sampling of Henyey-Greenstein phase function
    # Simplified: random direction with bias g
    # This is a placeholder for a proper HS sampling algorithm
    # For the purpose of this article, we assume a valid sampling function exists
    # [G] A proper implementation would use the method from Jen01 [S1]
    return [random.random(), random.random(), random.random()] # Placeholder
```

## Subtopics

### BSSRDF Parametrization
This variant focuses on the dimensionality reduction of the BSSRDF. By normalizing spatial coordinates by the mean-free path $\ell$, the BSSRDF depends on only five parameters: $S(r, \theta_s, \theta_i, \theta_o, \phi_o)$, where $r$ and $\theta_s$ are polar coordinates for the spatial difference, and $\theta_i, \theta_o, \phi_o$ are angular parameters [S1]. The material behavior is further reduced to three intrinsic parameters: albedo $\alpha$, anisotropy $g$, and index of refraction ratio $\eta$ [S1]. This parametrization allows for a generic representation of translucent materials, where the response of one material can be scaled to another by the ratio of their mean-free paths [S1].

## Operating characteristics
*   **Time Complexity:** $O(N_p \cdot N_s \cdot N_{steps})$, where $N_p$ is the number of photons, $N_s$ is the number of sample points, and $N_{steps}$ is the average number of scattering events per photon. The contribution calculation for each photon to each sample point is $O(1)$, but must be done for all pairs.
*   **Memory Complexity:** $O(N_s)$ to store sample points and $O(B)$ for the BSSRDF bins, where $B$ is the number of bins in the 5D parameter space.
*   **Performance:** The method is computationally expensive due to the high-dimensional parameter space and the need for many photons to reduce noise. It is typically used offline to generate reference data [S1].
*   **Strengths:** Accurately captures single and multiple scattering effects, including complex boundary interactions like Fresnel reflection and total internal reflection [S1].
*   **Limitations:** Slow convergence due to Monte-Carlo noise. The infinite plane boundary assumption limits its applicability to complex geometries [S1].
*   **When to use:** When high-fidelity BSSRDF data is needed for validation or as input for other methods like Photon Beam Diffusion [S1].
*   **When not to use:** For real-time rendering or complex geometries where the infinite plane assumption is invalid [S1].

## Background
The method is based on the experimental framework described by Donner et al. [S1]. It follows the approach of Donner et al. [DLR+ 09] for easier comparison with their results. The BSSRDF definition and the Monte-Carlo simulation technique are detailed in "Double- and Multiple-scattering Effects in Translucent Materials" [S1]. The phase function sampling method is attributed to Jensen [Jen01] [S1].

## Sources
[S1] Double- and Multiple-scattering Effects in Translucent Materials, https://doi.org/10.1109/MCG.2013.19, fetched 2026-10-26
