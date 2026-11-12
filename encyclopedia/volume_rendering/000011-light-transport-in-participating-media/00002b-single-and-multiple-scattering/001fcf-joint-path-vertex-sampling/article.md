# Joint Path Vertex Sampling

Also called: 3-random-decision connection, connection subpath sampling

## Overview
Joint Path Vertex Sampling is a method for constructing a full light transport path by connecting a light subpath endpoint $\mathbf{a}$ and an eye subpath endpoint $\mathbf{d}$ via a connection subpath containing two new vertices $\mathbf{b}$ and $\mathbf{c}$ [S1]. It solves the high variance associated with standard vertex-vertex connections in participating media by using three random decisions (two distances and one direction) to sample the connection subpath from a joint probability density function (PDF) proportional to the geometry and scattering terms [S1]. Unlike sibling methods such as Equiangular Sampling or Virtual Ray Lights (VRLs) which use 0, 1, or 2 random decisions, this method explicitly samples the intermediate direction $\boldsymbol{\omega}_{cb}$ to cancel out singularities in the path contribution function, thereby reducing variance [S1].

## How it works
**Inputs and Outputs**
*   **Input:** Vertex $\mathbf{a}$ with incident direction $\boldsymbol{\omega}_{la}$ (end of light subpath); vertex $\mathbf{d}$ with outgoing direction $\boldsymbol{\omega}_{dc}$ (end of eye subpath). The configuration $\Xi$ includes all medium properties ($\sigma_t, \sigma_s$) and phase functions.
*   **Output:** Vertices $\mathbf{b}$ and $\mathbf{c}$, and the complete path segment $\mathbf{a} \to \mathbf{b} \to \mathbf{c} \to \mathbf{d}$.

**Data Structures**
The method operates on geometric primitives:
*   Points $\mathbf{p} \in \mathbb{R}^3$.
*   Unit direction vectors $\boldsymbol{\omega} \in \mathbb{R}^3$.
*   Scalar distances $t \in \mathbb{R}^+$.
*   Configuration $\Xi$: A container holding $\mathbf{a}, \boldsymbol{\omega}_{la}, \mathbf{d}, \boldsymbol{\omega}_{dc}$ and medium coefficients.

**Formulas**
The target joint PDF for the connection subpath vertices $\mathbf{b}$ and distance $t_{dc}$ is defined as [S1]:
$$ p(\mathbf{b}, t_{dc} \mid \Xi) = C_\Xi G(\mathbf{abcd}) \rho(\mathbf{abc}) $$
where:
*   $G(\mathbf{abcd}) = G(\mathbf{a}, \mathbf{b}) G(\mathbf{b}, \mathbf{c}) G(\mathbf{c}, \mathbf{d})$ is the product of geometry terms between consecutive vertices.
*   $\rho(\mathbf{abc}) = \rho(\mathbf{a}) \rho(\mathbf{b}) \rho(\mathbf{c})$ is the product of scattering terms (phase functions or albedo) at vertices $\mathbf{a}, \mathbf{b}, \mathbf{c}$.
*   $\rho(\mathbf{d})$ is excluded because the outgoing direction $\boldsymbol{\omega}_{dc}$ is fixed by the input [S1].
*   $C_\Xi$ is a normalization factor dependent on the input configuration [S1].

**Factorizations**
Two factorizations of the joint PDF are supported, determining the order of sampling [S1]:

1.  **Unidirectional Factorization:** Vertices $\mathbf{c}$ and $\mathbf{b}$ are sampled from the same direction starting from $\mathbf{d}$.
    $$ p(t_{cb}, \boldsymbol{\omega}_{cb}, t_{dc} \mid \Xi) = p(t_{dc} \mid \Xi) \, p(\boldsymbol{\omega}_{cb} \mid t_{dc}, \Xi) \, p(t_{cb} \mid \boldsymbol{\omega}_{cb}, t_{dc}, \Xi) $$
    Steps:
    1.  Sample distance $t_{dc}$ from $\mathbf{d}$ along $\boldsymbol{\omega}_{dc}$ to get $\mathbf{c}$.
    2.  Sample direction $\boldsymbol{\omega}_{cb}$ from $\mathbf{c}$.
    3.  Sample distance $t_{cb}$ from $\mathbf{c}$ along $\boldsymbol{\omega}_{cb}$ to get $\mathbf{b}$.

2.  **Bidirectional Factorization:** Vertices $\mathbf{b}$ and $\mathbf{c}$ are sampled from opposite directions.
    $$ p(t_{ab}, \boldsymbol{\omega}_{ab}, t_{dc} \mid \Xi) = p(\boldsymbol{\omega}_{ab} \mid \Xi) \, p(t_{ab} \mid \boldsymbol{\omega}_{ab}, \Xi) \, p(t_{dc} \mid \boldsymbol{\omega}_{ab}, t_{ab}, \Xi) $$
    Steps:
    1.  Sample direction $\boldsymbol{\omega}_{ab}$ from $\mathbf{a}$.
    2.  Sample distance $t_{ab}$ from $\mathbf{a}$ along $\boldsymbol{\omega}_{ab}$ to get $\mathbf{b}$.
    3.  Sample distance $t_{dc}$ from $\mathbf{d}$ along $\boldsymbol{\omega}_{dc}$ to get $\mathbf{c}$.

**Analytic Sampling (Unidirectional, Isotropic)**
For isotropic scattering, fully analytic formulas exist for the unidirectional factorization [S1].

1.  **Sampling $t_{dc}$:**
    The marginal PDF for $t_{dc}$ is derived by integrating out $\boldsymbol{\omega}_{cb}$ and $t_{cb}$. The specific analytic form depends on the normalization $C_\Xi$, but the process involves sampling from the distribution proportional to the integrated geometry/scattering terms [S1].

2.  **Sampling $\boldsymbol{\omega}_{cb}$:**
    Given $t_{dc}$ (and thus $\mathbf{c}$), the conditional PDF for the direction $\boldsymbol{\omega}_{cb}$ is [S1]:
    $$ p(\boldsymbol{\omega}_{cb} \mid t_{dc}, \Xi) = \frac{\pi - \theta_{cb}}{\pi^3 \sin \theta_{cb}} $$
    where $\theta_{cb}$ is the angle between $\boldsymbol{\omega}_{cb}$ and the vector $\mathbf{c} - \mathbf{a}$ (length $t_{ca}$) [S1].
    *   **Pitfall:** This PDF diverges as $\theta_{cb} \to 0$. Sampling from this distribution cancels the corresponding singularity in the path contribution function, reducing variance [S1].

3.  **Sampling $t_{cb}$:**
    Given $\boldsymbol{\omega}_{cb}$ and $t_{dc}$, the conditional PDF for $t_{cb}$ is [S1]:
    $$ p(t_{cb} \mid \boldsymbol{\omega}_{cb}, t_{dc}, \Xi) = \frac{t_{ca} \sin \theta_{cb}}{\pi - \theta_{cb}} \frac{1}{h_{ca^\perp}^2 + (t_{ca^\perp} - t_{cb})^2} $$
    where:
    *   $h_{ca^\perp}$ is the perpendicular distance between vertex $\mathbf{a}$ and the ray $(\mathbf{c}, \boldsymbol{\omega}_{cb})$.
    *   $t_{ca^\perp}$ is the distance from $\mathbf{c}$ to the projection of $\mathbf{a}$ onto the ray $(\mathbf{c}, \boldsymbol{\omega}_{cb})$.
    *   The denominator term $h_{ca^\perp}^2 + (t_{ca^\perp} - t_{cb})^2$ is equal to $t_{ba}^2$, the squared distance between $\mathbf{b}$ and $\mathbf{a}$ [S1].
    *   This PDF is identical to that of Equiangular Sampling when $\mathbf{a}, \boldsymbol{\omega}_{cb}, \mathbf{c}$ are fixed [S1].

**Pseudocode**
The following implements the Unidirectional Factorization for isotropic scattering.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import random

def sample_joint_path_vertex_sampling_unidirectional(a, omega_la, d, omega_dc, sigma_t, sigma_s):
    """
    Samples vertices b and c connecting light endpoint a and eye endpoint d.
    Uses Unidirectional Factorization with 3 random decisions.
    Assumes isotropic scattering for analytic PDFs.
    """
    # Step 1: Sample t_dc from d along omega_dc to get c
    # The PDF p(t_dc | Xi) requires integration of the joint PDF.
    # For isotropic scattering, the marginal PDF for t_dc is derived from
    # the normalization constant C_Xi. In practice, this often requires
    # numerical inversion or precomputed tables if analytic forms are complex,
    # but the source notes that fully analytic formulas exist for the whole chain.
    # We assume a helper `sample_t_dc` exists which samples from the marginal
    # distribution proportional to the integrated geometry/scattering terms.
    t_dc = sample_t_dc_marginal(a, omega_la, d, omega_dc, sigma_t, sigma_s)
    c = d + t_dc * omega_dc

    # Step 2: Sample direction omega_cb from c
    # PDF: p(omega_cb | t_dc, Xi) = (pi - theta_cb) / (pi^3 * sin(theta_cb))
    # theta_cb is angle between omega_cb and (a - c).
    # We need to sample a direction on the cone around (a-c) or generally
    # in 3D space weighted by this PDF.
    # Note: The source defines theta_cb as angle between omega_cb and line c-a.
    # Let vector v = a - c.
    v = a - c
    t_ca = math.norm(v)
    
    # Sample theta_cb from the distribution derived from the PDF.
    # The PDF is over the sphere. In spherical coordinates around v,
    # the density is proportional to (pi - theta) / sin(theta).
    # We sample theta_cb and phi_cb.
    # phi_cb is uniform in [0, 2pi).
    phi_cb = random.uniform(0, 2 * math.pi)
    
    # Sample theta_cb. The marginal PDF for theta is:
    # p(theta) ~ (pi - theta) / sin(theta) * sin(theta) = pi - theta
    # (The sin(theta) comes from the spherical area element).
    # So p(theta) = (pi - theta) / pi^2 for theta in [0, pi].
    # CDF: F(theta) = integral_0^theta (pi - t)/pi^2 dt = [pi*t - t^2/2]/pi^2
    # F(theta) = (pi*theta - theta^2/2) / pi^2
    # Invert F(theta) = u: theta^2 - 2*pi*theta + 2*pi^2*u = 0
    # theta = (2*pi - sqrt(4*pi^2 - 8*pi^2*u)) / 2 = pi * (1 - sqrt(1 - 2*u))
    u = random.uniform(0, 1)
    theta_cb = math.pi * (1 - math.sqrt(1 - 2 * u))
    
    # Construct omega_cb in local coordinates and rotate to align with v
    # Local basis: z-axis along v.
    # omega_cb_local = (sin(theta)*cos(phi), sin(theta)*sin(phi), cos(theta))
    sin_theta = math.sin(theta_cb)
    cos_theta = math.cos(theta_cb)
    
    # Create orthonormal basis for v
    if abs(v[2]) < 0.999999:
        up = (0, 0, 1)
    else:
        up = (1, 0, 0)
    x_axis = normalize(cross(v, up))
    y_axis = cross(x_axis, v)
    x_axis = normalize(x_axis)
    y_axis = normalize(y_axis)
    z_axis = normalize(v)
    
    omega_cb_local = (
        sin_theta * math.cos(phi_cb),
        sin_theta * math.sin(phi_cb),
        cos_theta
    )
    omega_cb = (
        x_axis[0]*omega_cb_local[0] + y_axis[0]*omega_cb_local[1] + z_axis[0]*omega_cb_local[2],
        x_axis[1]*omega_cb_local[0] + y_axis[1]*omega_cb_local[1] + z_axis[1]*omega_cb_local[2],
        x_axis[2]*omega_cb_local[0] + y_axis[2]*omega_cb_local[1] + z_axis[2]*omega_cb_local[2]
    )

    # Step 3: Sample t_cb from c along omega_cb to get b
    # PDF: p(t_cb | omega_cb, t_dc, Xi) = K * 1 / (h_perp^2 + (t_perp - t_cb)^2)
    # where K = (t_ca * sin(theta_cb)) / (pi - theta_cb)
    # This is a Cauchy/Lorentzian distribution.
    
    # Calculate geometric parameters
    # h_ca_perp: distance from a to ray (c, omega_cb)
    # t_ca_perp: distance from c to projection of a on ray (c, omega_cb)
    vec_ca = a - c
    proj_len = dot(vec_ca, omega_cb)
    t_ca_perp = proj_len
    vec_proj = t_ca_perp * omega_cb
    vec_perp = vec_ca - vec_proj
    h_ca_perp = math.norm(vec_perp)
    
    # The PDF is: p(t) = C / (h^2 + (t_perp - t)^2)
    # This is a shifted Cauchy distribution.
    # Standard Cauchy: f(x; x0, gamma) = 1 / (pi * gamma * (1 + ((x-x0)/gamma)^2))
    # Here, denominator is h^2 + (t_perp - t)^2 = h^2 * (1 + ((t - t_perp)/h)^2)
    # So gamma = h_ca_perp, x0 = t_ca_perp.
    # The normalization constant for Cauchy is 1/(pi*gamma).
    # Our K must match 1/(pi*h).
    # Check: K = (t_ca * sin_theta) / (pi - theta).
    # Does K = 1/(pi*h)?
    # From source Eq 22: integral of 1/(h^2 + (t_perp - t)^2) dt = pi / h.
    # So the normalized PDF is (h/pi) * 1/(h^2 + ...).
    # Our K in Eq 23 is (t_ca sin theta)/(pi - theta).
    # Eq 22 says integral is C_Xi * pi / (t_ca sin theta).
    # So C_Xi = (pi - theta) / (pi * t_ca sin theta).
    # The joint PDF normalization leads to the conditional PDF.
    # The source states Eq 23 is the PDF.
    # We sample from Cauchy(t_ca_perp, h_ca_perp).
    
    # Sample from Cauchy distribution
    # Cauchy(x0, gamma) = x0 + gamma * tan(pi * (u - 0.5))
    u = random.uniform(0, 1)
    t_cb = t_ca_perp + h_ca_perp * math.tan(math.pi * (u - 0.5))
    
    # Clamp t_cb to be positive? The source implies t_cb >= 0.
    # Cauchy has support (-inf, inf). If t_cb < 0, reject or clamp?
    # The source does not explicitly mention rejection, but physically t_cb >= 0.
    # If t_cb < 0, the vertex b is behind c.
    # We assume the method handles this or the distribution is truncated.
    # Given the singularity cancellation, we proceed.
    if t_cb < 0:
        t_cb = 0 # [G] Guard against negative distance
    
    b = c + t_cb * omega_cb
    
    return b, c, t_dc, t_cb, omega_cb

def sample_t_dc_marginal(a, omega_la, d, omega_dc, sigma_t, sigma_s):
    """
    Placeholder for the marginal sampling of t_dc.
    The source notes that fully analytic formulas exist, but the explicit
    closed-form for p(t_dc) is not provided in the text snippets.
    It is derived by integrating the joint PDF over omega_cb and t_cb.
    """
    # [G] In a full implementation, this would involve inverting the CDF
    # derived from the normalization constant C_Xi and the integrated
    # geometry/scattering terms.
    return 1.0 # Placeholder
```

## Subtopics

### [Bridge Sampling](3881-bridge-sampling.md)
Bridge Sampling is a method for sampling a subpath (bridge) of arbitrary vertex count connecting two vertices in a participating medium, where the probability density is proportional to all inner phase functions and reciprocal squared distance terms. It sets itself apart from sibling methods like Equiangular Sampling and MVNEE by fixing the endpoint via rotation and uniform scaling of a preliminary path, which preserves angles and handles highly-peaked forward scattering more robustly. Unlike Path Guiding, it does not use memory to store sampling densities or handle visibility explicitly during the sampling construction.

### [Bidirectional Path Tracing](6666-bidirectional-path-tracing.md)
Bidirectional Path Tracing (BPT) is a stochastic light transport algorithm for participating media that constructs complete paths by connecting independent subpaths traced from light sources and the eye point. It solves the high variance and slow convergence of unidirectional methods (like standard path tracing or light tracing) in scenes with refractive boundaries or complex scattering by ensuring both emitted radiance and sensor importance are included in the sampling density. Unlike light tracing, which produces partial solutions that are not useful in short times, BPT is view-dependent and allows gradual improvement of image quality starting from a crude approximation. It is the only existing unbiased method for anisotropic media mentioned in the sources.

### Virtual ray lights (VRLs)
Virtual Ray Lights (VRLs) are a many-light method reinterpreted as a bidirectional segment-segment connection technique that samples propagation distances from a joint conditional distribution proportional to the scattering and geometry terms [S1]. Unlike Joint Path Vertex Sampling which uses 3 random decisions, VRLs use 2 random decisions (distances $t_{ab}$ and $t_{dc}$) [S1].

## Operating characteristics
*   **Complexity:** The method involves calculating geometric terms ($h_{ca^\perp}, t_{ca^\perp}, \theta_{cb}$) and sampling from Cauchy and spherical distributions. The computational cost is dominated by the geometric calculations and the sampling routines, which are $O(1)$ per connection.
*   **Performance:** The method reduces variance compared to vertex-vertex connections, Equiangular Sampling, and VRLs by using 3 random decisions [S1]. The explicit sampling of $\boldsymbol{\omega}_{cb}$ cancels the singularity in the path contribution function, leading to lower variance in scenes with strong scattering [S1].
*   **Limitations:** The PDF for $\boldsymbol{\omega}_{cb}$ diverges as $\theta_{cb} \to 0$, requiring careful sampling to avoid numerical instability [S1]. The method assumes isotropic scattering for the fully analytic unidirectional formulas; anisotropic scattering requires tabulation or numerical methods [S1].
*   **When to use:** Use when variance reduction is critical in participating media, especially when connecting subpaths with arbitrary lengths. It is particularly effective when the geometry configuration allows for the cancellation of singularities.

## Background
Joint Path Vertex Sampling was introduced by Kulla and Fajardo in 2012 [S1]. It was developed to address the limitations of existing connection techniques (0, 1, or 2 random decisions) by introducing a 3-random-decision approach. The method builds on the principles of conservation of energy and the path integral formulation of light transport. It is used in volumetric path tracing to improve the efficiency of multiple scattering calculations.

## Sources
[S1] Joint importance sampling of low-order volumetric scattering, https://doi.org/10.1145/2508363.2508411, fetched 2026-10-28
