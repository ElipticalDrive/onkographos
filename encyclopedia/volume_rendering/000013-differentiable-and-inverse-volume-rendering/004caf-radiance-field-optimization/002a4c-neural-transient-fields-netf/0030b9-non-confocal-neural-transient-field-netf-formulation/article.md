# Non-confocal Neural Transient Field (NeTF) Formulation

Also called: Ellipsoidal NeTF, Non-confocal NeTF

## Overview
This method formulates the transient signal in Non-Line-of-Sight (NLOS) imaging for non-confocal setups, where the illumination spot $P$ and detection spot $P'$ are distinct [S1]. It models the received energy by integrating over semi-ellipsoids defined by foci at $P$ and $P'$ on the relay wall [S1]. It differs from the confocal NeTF formulation by using ellipsoidal coordinates to handle the asymmetric optical paths, allowing the transient to be computed as a double integral over the ellipsoidal surface parameters [S1].

## How it works
**Inputs:**
- Illumination spot position $P$ and detection spot position $P'$ on the relay wall [S1].
- Time $t$ of the transient measurement [S1].
- Scene density $\sigma$ and reflectance $\rho$ represented as functions of ellipsoidal coordinates $(\mu, \nu, \varphi)$ [S1].

**Outputs:**
- Transient signal $\tau(P, P', t)$ received at $P'$ from $P$ [S1].

**Data Structures:**
- Ellipsoidal coordinates $(\mu, \nu, \varphi)$ parameterizing the semi-ellipsoid [S1].
- Geometric constants: semi-major axis $\alpha$, focal length $\gamma$, eccentricity $e$ [S1].

**Parameters:**
- $\alpha = ct/2$: Semi-major axis of the ellipsoid, where $c$ is the speed of light [S1].
- $\gamma = |\vec{OP} - \vec{OP'}|$: Focal length, distance from the origin to each focus (assuming origin $O$ is the midpoint between $P$ and $P'$) [S1].
- $e = \gamma/\alpha$: Eccentricity [S1].
- $\Gamma_0$: Constant system factor derived from illumination power, aperture, and detection parameters (simplified from $\Gamma$) [S1].

**Steps:**

1.  **Define Geometric Constants:**
    Calculate the semi-major axis $\alpha = ct/2$ and the focal length $\gamma$ based on the positions of $P$ and $P'$ [S1]. Compute eccentricity $e = \gamma/\alpha$ [S1].
    *Pitfall:* If $e \ge 1$, the geometry is degenerate (hyperbolic or planar), which is invalid for the standard ellipsoidal formulation [S1]. This occurs if $ct < \gamma$, i.e., the time of flight is less than the direct distance between $P$ and $P'$ [S1].

2.  **Determine Coordinate $\mu$:**
    For a fixed time $t$, the coordinate $\mu$ is constrained by the Dirac delta function in the integral [S1]. The non-zero contribution occurs when:
    $$ \mu = \text{arccosh}\left(\frac{ct}{2\gamma}\right) $$
    [G] Ensure the argument to `arccosh` is $\ge 1$. If $ct < 2\gamma$, the transient is zero.

3.  **Compute Jacobian $J$:**
    The Jacobian for the transformation from Cartesian to ellipsoidal coordinates is:
    $$ J = \gamma^3 \sinh \mu \sin \nu (\sinh^2 \mu + \sin^2 \nu) $$
    [S1]

4.  **Compute Distances $r_1$ and $r_2$:**
    - $r_1$ is the distance from $P$ to the scene point $Q$, and $r_2$ is the distance from $Q$ to $P'$ [S1].
    - In the local coordinate system where $P$ and $P'$ are symmetric about the origin and aligned with the y-axis ($P=(0,-\gamma,0)$, $P'=(0,\gamma,0)$), the Cartesian coordinates of $Q$ are:
      $$ x = \gamma \sinh \mu \sin \nu \cos \varphi $$
      $$ y = \gamma \cosh \mu \cos \nu $$
      $$ z = \gamma \sinh \mu \sin \nu \sin \varphi $$
      [S1]
    - The distances are then computed as $r_1 = \sqrt{x^2 + (y+\gamma)^2 + z^2}$ and $r_2 = \sqrt{x^2 + (y-\gamma)^2 + z^2}$ [G].

5.  **Evaluate Integrand:**
    The simplified transient equation (discarding attenuation) is:
    $$ \tau(P, P', t) = \Gamma_0 \iint_E \frac{J}{r_1^2 r_2^2} \sigma(\mu, \nu, \varphi) \rho(\mu, \nu, \varphi, P, P') \, d\nu \, d\varphi $$
    [S1] Where $\mu$ is fixed as calculated in Step 2.

6.  **Integrate:**
    Perform the double integral over $\nu \in [0, \pi]$ and $\varphi \in [0, 2\pi]$ [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def compute_non_confocal_netf_transient(P, P_prime, t, sigma_func, rho_func, num_samples_nu, num_samples_phi, gamma_val, Gamma0=1.0):
    """
    Computes the transient signal for non-confocal NeTF using the simplified ellipsoidal formulation.
    
    Args:
        P: Array-like, position of illumination spot (unused in local frame calculation).
        P_prime: Array-like, position of detection spot (unused in local frame calculation).
        t: Float, time of flight.
        sigma_func: Function, returns scene density given ellipsoidal coords (mu, nu, phi).
        rho_func: Function, returns reflectance given ellipsoidal coords and spots.
        num_samples_nu: Int, number of samples for nu.
        num_samples_phi: Int, number of samples for phi.
        gamma_val: Float, half the distance between P and P'.
        Gamma0: Float, constant system factor.
        
    Returns:
        Float, transient signal tau.
    """
    c = 299792458.0  # Speed of light in m/s
    
    # Step 1: Geometric Constants
    alpha = c * t / 2.0
    
    # Pitfall: Check for degenerate geometry
    if alpha <= gamma_val:
        return 0.0
    
    # Step 2: Determine mu
    # mu = arccosh(ct / 2*gamma) = arccosh(alpha / gamma)
    mu = np.arccosh(alpha / gamma_val)
    
    # Step 3 & 4: Discretize integration domain and compute local Cartesian coordinates
    # nu in [0, pi], phi in [0, 2pi]
    nus = np.linspace(0, np.pi, num_samples_nu)
    phis = np.linspace(0, 2 * np.pi, num_samples_phi)
    
    # Create meshgrid
    NU, PHI = np.meshgrid(nus, phis, indexing='ij')
    
    # Local Cartesian coordinates (focal axis aligned with Y)
    sinh_mu = np.sinh(mu)
    cosh_mu = np.cosh(mu)
    
    X_local = gamma_val * sinh_mu * np.sin(NU) * np.cos(PHI)
    Y_local = gamma_val * cosh_mu * np.cos(NU)
    Z_local = gamma_val * sinh_mu * np.sin(NU) * np.sin(PHI)
    
    # Step 4: Compute distances r1 and r2
    # P is at (0, -gamma, 0), P' is at (0, gamma, 0)
    r1 = np.sqrt(X_local**2 + (Y_local + gamma_val)**2 + Z_local**2)
    r2 = np.sqrt(X_local**2 + (Y_local - gamma_val)**2 + Z_local**2)
    
    # Step 3: Compute Jacobian J
    sin_nu = np.sin(NU)
    J = (gamma_val**3) * sinh_mu * sin_nu * (sinh_mu**2 + sin_nu**2)
    
    # Step 5: Evaluate sigma and rho
    sigma_vals = sigma_func(mu, NU, PHI)
    rho_vals = rho_func(mu, NU, PHI, P, P_prime)
    
    # Step 5: Compute Integrand
    integrand = Gamma0 * J / (r1**2 * r2**2) * sigma_vals * rho_vals
    
    # Step 6: Integrate
    # Use simple Riemann sum with correct step sizes
    d_nu = np.pi / (num_samples_nu - 1)
    d_phi = 2 * np.pi / (num_samples_phi - 1)
    
    tau = np.sum(integrand) * d_nu * d_phi
    
    return tau
```

**Edge Cases and Pitfalls:**
- **Attenuation Ignored:** The simplified equation discards the attenuation term $\exp(-A \int \sigma ds)$. This ignores occlusions, potentially leading to artifacts in scenes with hidden surfaces [S1].
- **Degenerate Geometry:** If $ct < \gamma$, the ellipsoid is undefined. The transient is zero [S1].
- **Coordinate Alignment:** The derivation assumes a specific coordinate system where $P$ and $P'$ are symmetric about the origin and parallel to the y-axis. Implementation must correctly map local ellipsoidal coordinates to this frame or apply the corresponding rotation [S1].

## Operating characteristics
- **Time Complexity:** Depends on the discretization of the double integral over $\nu$ and $\varphi$. $O(N_\nu N_\varphi)$ [G].
- **Memory Complexity:** $O(N_\nu N_\varphi)$ to store intermediate arrays for the integration [G].
- **Strengths:** Handles non-confocal setups where illumination and detection spots are separated. Provides a closed-form geometric formulation for the transient [S1].
- **Limitations:** Discarding attenuation ignores occlusions. Requires careful coordinate transformation [S1].

## Background
This formulation is part of the Neural Transient Fields (NeTF) method for Non-Line-of-Sight (NLOS) imaging. It extends the confocal NeTF formulation to non-confocal settings. The method is described in "Non-line-of-Sight Imaging via Neural Transient Fields" [S1].

## Sources
[S1] Non-line-of-Sight Imaging via Neural Transient Fields, https://doi.org/10.1109/tpami.2021.3076062, fetched 2026-10-26
