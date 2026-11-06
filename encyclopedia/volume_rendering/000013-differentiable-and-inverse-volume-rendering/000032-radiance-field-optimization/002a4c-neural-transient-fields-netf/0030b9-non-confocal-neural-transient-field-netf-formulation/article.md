# Non-confocal Neural Transient Field (NeTF) Formulation

Also called: Ellipsoidal NeTF, Non-confocal NeTF

## Overview
This method formulates the transient signal in Non-Line-of-Sight (NLOS) imaging for non-confocal setups, where the illumination spot $P$ and detection spot $P'$ are distinct. It models the received energy by integrating over semi-ellipsoids defined by foci at $P$ and $P'$ on the relay wall. It differs from the confocal NeTF formulation by using ellipsoidal coordinates to handle the asymmetric optical paths, allowing the transient to be computed as a double integral over the ellipsoidal surface parameters.

## How it works
**Inputs:**
- Illumination spot position $P$ and detection spot position $P'$ on the relay wall.
- Time $t$ of the transient measurement.
- Scene density $\sigma$ and reflectance $\rho$ represented as functions of ellipsoidal coordinates $(\mu, \nu, \varphi)$.

**Outputs:**
- Transient signal $\tau(P, P', t)$ received at $P'$ from $P$.

**Data Structures:**
- Ellipsoidal coordinates $(\mu, \nu, \varphi)$ parameterizing the semi-ellipsoid.
- Geometric constants: semi-major axis $\alpha$, focal length $\gamma$, eccentricity $e$.

**Parameters:**
- $\alpha = ct/2$: Semi-major axis of the ellipsoid, where $c$ is the speed of light.
- $\gamma = |\vec{OP} - \vec{OP'}|$: Focal length, distance between foci $P$ and $P'$ (assuming origin $O$ is midpoint).
- $e = \gamma/\alpha$: Eccentricity.
- $\Gamma_0$: Constant factor derived from system parameters (simplified from $\Gamma$).

**Steps:**

1.  **Define Geometric Constants:**
    Calculate the semi-major axis $\alpha = ct/2$ and the focal length $\gamma$ based on the positions of $P$ and $P'$. Compute eccentricity $e = \gamma/\alpha$.
    *Pitfall:* If $e \ge 1$, the geometry is degenerate (hyperbolic or planar), which is invalid for the standard ellipsoidal formulation. This occurs if $ct < \gamma$, i.e., the time of flight is less than the direct distance between $P$ and $P'$.

2.  **Determine Coordinate $\mu$:**
    For a fixed time $t$, the coordinate $\mu$ is constrained by the Dirac delta function in the integral. The non-zero contribution occurs when:
    $$ \mu = \text{arccosh}\left(\frac{ct}{2\gamma}\right) $$
    [G] Ensure the argument to `arccosh` is $\ge 1$. If $ct < 2\gamma$, the transient is zero.

3.  **Compute Jacobian $J$:**
    The Jacobian for the transformation from Cartesian to ellipsoidal coordinates is:
    $$ J = \gamma^3 \sinh \mu \sin \nu (\sinh^2 \mu + \sin^2 \nu) $$

4.  **Compute Distances $r_1$ and $r_2$:**
    - $r_1$ is the distance from $P$ to the scene point $Q$. In ellipsoidal coordinates, $r_1$ is related to $\mu$ and $\nu$. From the coordinate transformation:
      $$ r_1 \cos \theta = \gamma \cosh \mu \cos \nu $$
      However, for the simplified implementation, $r_1$ and $r_2$ are often computed from the Cartesian position $Q$ derived from $(\mu, \nu, \varphi)$ or approximated. The source notes the simplified equation uses $r_1$ and $r_2$ explicitly.
      *Note:* The source provides the transformation equations:
      $$ r_1 \sin \theta \cos \phi = \gamma \sinh \mu \sin \nu \cos \varphi $$
      $$ r_1 \sin \theta \sin \phi = \gamma \sinh \mu \sin \nu \sin \varphi $$
      $$ r_1 \cos \theta = \gamma \cosh \mu \cos \nu $$
      To implement, one typically converts $(\mu, \nu, \varphi)$ to Cartesian $(x,y,z)$ to compute $r_1 = |Q-P|$ and $r_2 = |Q-P'|$.

5.  **Evaluate Integrand:**
    The simplified transient equation (discarding attenuation) is:
    $$ \tau(P, P', t) = \Gamma_0 \iint_E \frac{J}{r_1^2 r_2^2} \sigma(\mu, \nu, \varphi) \rho(\mu, \nu, \varphi, P, P') \, d\nu \, d\varphi $$
    Where $\mu$ is fixed as calculated in Step 2.

6.  **Integrate:**
    Perform the double integral over $\nu \in [0, \pi]$ and $\varphi \in [0, 2\pi]$ (or appropriate bounds for the semi-ellipsoid).

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def compute_non_confocal_netf_transient(P, P_prime, t, sigma_func, rho_func, num_samples_nu, num_samples_phi):
    """
    Computes the transient signal for non-confocal NeTF.
    
    Args:
        P: Array-like, position of illumination spot.
        P_prime: Array-like, position of detection spot.
        t: Float, time of flight.
        sigma_func: Function, returns scene density given ellipsoidal coords (mu, nu, phi).
        rho_func: Function, returns reflectance given ellipsoidal coords and spots.
        num_samples_nu: Int, number of samples for nu.
        num_samples_phi: Int, number of samples for phi.
        
    Returns:
        Float, transient signal tau.
    """
    c = 299792458.0  # Speed of light in m/s
    
    # Step 1: Geometric Constants
    # Assuming P and P_prime are symmetric about origin O for the derivation
    # In practice, gamma is half the distance between P and P_prime if O is midpoint
    # Or gamma is the distance from origin to focus. 
    # Source: gamma = |OP - OP'|. If O is midpoint, gamma = |P - P'| / 2.
    # Let's assume the coordinate system is set such that P and P' are symmetric about O.
    # If not, the coordinate transformation needs adjustment. 
    # The source states: "set up the coordinate system so that P and P' are symmetric about origin O"
    
    dist_PP_prime = np.linalg.norm(np.array(P) - np.array(P_prime))
    gamma = dist_PP_prime / 2.0
    
    alpha = c * t / 2.0
    
    # Pitfall: Check for degenerate geometry
    if alpha <= gamma:
        return 0.0
    
    # Step 2: Determine mu
    # mu = arccosh(ct / 2gamma) = arccosh(alpha / gamma)
    mu = np.arccosh(alpha / gamma)
    
    # Step 3: Precompute terms dependent on mu
    sinh_mu = np.sinh(mu)
    cosh_mu = np.cosh(mu)
    sinh_mu_sq = sinh_mu ** 2
    
    # Jacobian factor J depends on mu, nu, phi
    # J = gamma^3 * sinh(mu) * sin(nu) * (sinh^2(mu) + sin^2(nu))
    
    # Step 4: Discretize integration domain
    # nu in [0, pi], phi in [0, 2pi]
    nus = np.linspace(0, np.pi, num_samples_nu)
    phis = np.linspace(0, 2 * np.pi, num_samples_phi)
    
    # Create meshgrid
    NU, PHI = np.meshgrid(nus, phis, indexing='ij')
    
    # Step 5: Compute Cartesian coordinates for each sample point
    # r1 * sin(theta) * cos(phi) = gamma * sinh(mu) * sin(nu) * cos(phi)
    # r1 * sin(theta) * sin(phi) = gamma * sinh(mu) * sin(nu) * sin(phi)
    # r1 * cos(theta) = gamma * cosh(mu) * cos(nu)
    
    # Let X = gamma * sinh(mu) * sin(nu) * cos(phi)
    # Let Y = gamma * sinh(mu) * sin(nu) * sin(phi)
    # Let Z = gamma * cosh(mu) * cos(nu)
    
    X = gamma * sinh_mu * np.sin(NU) * np.cos(PHI)
    Y = gamma * sinh_mu * np.sin(NU) * np.sin(PHI)
    Z = gamma * cosh_mu * np.cos(NU)
    
    # The coordinate system in the source has P and P' parallel to y-axis.
    # We need to map (X,Y,Z) to the actual world coordinates.
    # Assuming the local ellipsoidal frame is aligned with the world frame such that
    # the major axis is along the line connecting P and P'.
    # If P and P' are on the y-axis, then the Z coordinate in the formula corresponds to the
    # direction perpendicular to the P-P' line? 
    # The source says: "P and P' parallel to y-axis".
    # And "r1 cos theta = gamma cosh mu cos nu".
    # Usually, in ellipsoidal coordinates, the coordinate along the focal axis is associated with cosh/cos.
    # Let's assume the standard mapping where the focal axis is the Z-axis in the local frame,
    # then rotate to world.
    # However, the source defines the transformation directly.
    # Let's assume the local frame (X,Y,Z) corresponds to the world frame where P and P' are on the Y axis.
    # Then the focal axis is Y.
    # In standard ellipsoidal coordinates, the focal axis is usually Z.
    # The source equations:
    # r1 sin theta cos phi = gamma sinh mu sin nu cos phi  -> X_coord
    # r1 sin theta sin phi = gamma sinh mu sin nu sin phi  -> Y_coord
    # r1 cos theta = gamma cosh mu cos nu                  -> Z_coord
    
    # If P and P' are on the Y axis, the focal axis is Y.
    # The coordinate associated with the focal axis in the source is the one with cosh mu cos nu.
    # That is the Z coordinate in the source's local frame.
    # So, in the source's local frame, the focal axis is Z.
    # But the text says "P and P' parallel to y-axis".
    # This implies a rotation.
    # Let's assume the source's local frame has the focal axis along Z.
    # Then we must rotate the points (X,Y,Z) to align the Z-axis with the vector P' - P.
    
    # Vector from P to P'
    vec_PP_prime = np.array(P_prime) - np.array(P)
    dist = np.linalg.norm(vec_PP_prime)
    if dist == 0:
        return 0.0
    unit_vec = vec_PP_prime / dist
    
    # We need to construct a rotation matrix that maps the local Z-axis (0,0,1) to unit_vec.
    # And local Y-axis (0,1,0) to something perpendicular.
    # This is complex. A simpler approach for implementation if the source assumes a specific alignment:
    # The source says "set up the coordinate system so that P and P' are symmetric about origin O and P P' parallel to y-axis".
    # This means in the source's derivation, the focal axis is Y.
    # But the equations given:
    # r1 cos theta = gamma cosh mu cos nu
    # This term is usually the one along the focal axis in standard definitions.
    # If the focal axis is Y, then Y_local = gamma cosh mu cos nu.
    # And X_local, Z_local are the other two.
    # The source equations:
    # r1 sin theta cos phi = gamma sinh mu sin nu cos phi  (Let's call this X_local)
    # r1 sin theta sin phi = gamma sinh mu sin nu sin phi  (Let's call this Z_local? No, usually Y)
    # r1 cos theta = gamma cosh mu cos nu                  (Let's call this Y_local)
    
    # Let's assume:
    # X_local = gamma * sinh(mu) * sin(nu) * cos(phi)
    # Y_local = gamma * cosh(mu) * cos(nu)  <-- Focal Axis
    # Z_local = gamma * sinh(mu) * sin(nu) * sin(phi)
    
    # Then rotate (X_local, Y_local, Z_local) to world coordinates.
    # We need a basis where Y_world is along vec_PP_prime.
    
    # Let's define the world basis.
    # Y_world = unit_vec
    # Choose an arbitrary vector not parallel to Y_world, e.g., (1,0,0).
    # X_world = normalize(cross(Y_world, (1,0,0))) if not parallel, else cross(Y_world, (0,1,0))
    # Z_world = cross(X_world, Y_world)
    
    up = np.array([1.0, 0.0, 0.0])
    if np.abs(np.dot(unit_vec, up)) > 0.99:
        up = np.array([0.0, 1.0, 0.0])
        
    X_world = np.cross(unit_vec, up)
    X_world = X_world / np.linalg.norm(X_world)
    Z_world = np.cross(X_world, unit_vec)
    
    # Construct rotation matrix R such that R * [X_local, Y_local, Z_local]^T = [X_world, Y_world, Z_world]^T * local_coords
    # Point_world = X_local * X_world + Y_local * Y_world + Z_local * Z_world
    
    X_local = gamma * sinh_mu * np.sin(NU) * np.cos(PHI)
    Y_local = gamma * cosh_mu * np.cos(NU)
    Z_local = gamma * sinh_mu * np.sin(NU) * np.sin(PHI)
    
    # World coordinates
    X_world_coords = X_local * X_world[0] + Y_local * unit_vec[0] + Z_local * Z_world[0]
    Y_world_coords = X_local * X_world[1] + Y_local * unit_vec[1] + Z_local * Z_world[1]
    Z_world_coords = X_local * X_world[2] + Y_local * unit_vec[2] + Z_local * Z_world[2]
    
    Q = np.stack([X_world_coords, Y_world_coords, Z_world_coords], axis=-1)
    
    # Step 6: Compute r1 and r2
    # r1 = distance from P to Q
    # r2 = distance from Q to P'
    P_arr = np.array(P)
    P_prime_arr = np.array(P_prime)
    
    r1 = np.linalg.norm(Q - P_arr, axis=-1)
    r2 = np.linalg.norm(Q - P_prime_arr, axis=-1)
    
    # Step 7: Compute Jacobian J
    # J = gamma^3 * sinh(mu) * sin(nu) * (sinh^2(mu) + sin^2(nu))
    sin_nu = np.sin(NU)
    sin_nu_sq = sin_nu ** 2
    J = (gamma**3) * sinh_mu * sin_nu * (sinh_mu_sq + sin_nu_sq)
    
    # Step 8: Evaluate sigma and rho
    # sigma_func takes (mu, nu, phi)
    sigma_vals = sigma_func(mu, NU, PHI)
    # rho_func takes (mu, nu, phi, P, P')
    rho_vals = rho_func(mu, NU, PHI, P, P_prime)
    
    # Step 9: Compute Integrand
    # Integrand = Gamma0 * J / (r1^2 * r2^2) * sigma * rho
    # Gamma0 is a constant. The source defines Gamma = A a r2_0 Ep / pi.
    # In the simplified equation, it becomes Gamma0.
    # We assume Gamma0 is provided or computed from system parameters.
    # For the purpose of the NeTF formulation, we often normalize or treat Gamma0 as part of the MLP output scaling.
    # Here we return the integral part.
    
    integrand = J / (r1**2 * r2**2) * sigma_vals * rho_vals
    
    # Step 10: Integrate
    # Use simple Riemann sum
    d_nu = nus[1] - nus[0]
    d_phi = phis[1] - phis[0]
    
    # Sum over nu and phi
    # The integral is double integral over nu and phi
    tau = np.sum(integrand) * d_nu * d_phi
    
    return tau
```

**Edge Cases and Pitfalls:**
- **Attenuation Ignored:** The simplified equation discards the attenuation term $\exp(-A \int \sigma ds)$. This ignores occlusions, potentially leading to artifacts in scenes with hidden surfaces.
- **Degenerate Geometry:** If $ct < \gamma$, the ellipsoid is undefined. The transient is zero.
- **Coordinate Alignment:** The derivation assumes a specific coordinate system where $P$ and $P'$ are symmetric about the origin and parallel to the y-axis. Implementation must correctly rotate the local ellipsoidal coordinates to the world frame.

## Operating characteristics
- **Time Complexity:** Depends on the discretization of the double integral over $\nu$ and $\varphi$. $O(N_\nu N_\varphi)$.
- **Memory Complexity:** $O(N_\nu N_\varphi)$ to store intermediate arrays for the integration.
- **Strengths:** Handles non-confocal setups where illumination and detection spots are separated. Provides a closed-form geometric formulation for the transient.
- **Limitations:** Discarding attenuation ignores occlusions. Requires careful coordinate transformation.

## Background
This formulation is part of the Neural Transient Fields (NeTF) method for Non-Line-of-Sight (NLOS) imaging. It extends the confocal NeTF formulation to non-confocal settings. The method is described in "Non-line-of-Sight Imaging via Neural Transient Fields" [S1].

## Sources
[S1] Non-line-of-Sight Imaging via Neural Transient Fields, https://doi.org/10.1109/tpami.2021.3076062, fetched 2026-10-26
