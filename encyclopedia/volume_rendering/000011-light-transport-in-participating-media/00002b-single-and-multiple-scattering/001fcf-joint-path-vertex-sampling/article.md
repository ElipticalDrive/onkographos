# Joint Path Vertex Sampling

Also called: 3-random-decision connection, connection subpath sampling

## Overview
Joint Path Vertex Sampling (JPVS) is a method for constructing a full light transport path by connecting a light subpath endpoint $\mathbf{a}$ and an eye subpath endpoint $\mathbf{d}$ via a connection subpath containing two new vertices $\mathbf{b}$ and $\mathbf{c}$ [S1]. It solves the high variance associated with connecting subpaths in participating media by using three random decisions (two distances and one direction) to sample the connection, rather than the zero, one, or two decisions used by methods like Equiangular Sampling or Virtual Ray Lights [S1]. This approach reduces variance by explicitly sampling the direction $\boldsymbol{\omega}_{cb}$ from a distribution that cancels out the geometric singularity in the path contribution function [S1].

## How it works
**Inputs and Outputs**
*   **Input:** Vertex $\mathbf{a}$ with incident direction $\boldsymbol{\omega}_{la}$ (end of light subpath); Vertex $\mathbf{d}$ with outgoing direction $\boldsymbol{\omega}_{dc}$ (end of eye subpath). The configuration is denoted $\Xi$ [S1].
*   **Output:** Vertices $\mathbf{b}$ and $\mathbf{c}$ forming the connection subpath, such that the full path is $\mathbf{x}_{l} \mathbf{a} \mathbf{b} \mathbf{c} \mathbf{d} \mathbf{x}_{e}$ [S1].

**Data Structures**
The method operates on geometric primitives:
*   $\mathbf{p} \in \mathbb{R}^3$: Position.
*   $\boldsymbol{\omega} \in \mathbb{S}^2$: Unit direction vector.
*   $t \in \mathbb{R}^+$: Path length/distance.
*   $G(\mathbf{u}, \mathbf{v}) = \frac{|\boldsymbol{\omega}_{uv} \cdot \mathbf{n}_v|}{\|\mathbf{v} - \mathbf{u}\|^2}$: Geometry term between vertices [S1].
*   $\rho(\mathbf{u})$: Scattering term (phase function or albedo) at vertex $\mathbf{u}$ [S1].

**Target PDF**
The goal is to sample $\mathbf{b}$ and $t_{dc}$ from a joint PDF proportional to the geometry and scattering terms of the connection subpath [S1]:
$$ p(\mathbf{b}, t_{dc} \mid \Xi) G(\mathbf{c}, \mathbf{d}) = C_\Xi G(\mathbf{abcd}) \rho(\mathbf{abc}) $$
where $G(\mathbf{abcd}) = G(\mathbf{a}, \mathbf{b})G(\mathbf{b}, \mathbf{c})G(\mathbf{c}, \mathbf{d})$ and $\rho(\mathbf{abc}) = \rho(\mathbf{a})\rho(\mathbf{b})\rho(\mathbf{c})$ [S1]. Note that $\rho(\mathbf{d})$ is excluded because $\boldsymbol{\omega}_{dc}$ is given [S1]. $C_\Xi$ is a normalization factor dependent on $\Xi$ [S1].

**Factorizations**
Two factorizations of the joint PDF are defined, corresponding to different sampling orders [S1].

1.  **Unidirectional Factorization:** Vertices $\mathbf{c}$ and $\mathbf{b}$ are sampled from the same direction starting from $\mathbf{d}$ [S1].
    $$ p(t_{cb}, \boldsymbol{\omega}_{cb}, t_{dc} \mid \Xi) = p(t_{dc} \mid \Xi) \cdot p(\boldsymbol{\omega}_{cb} \mid t_{dc}, \Xi) \cdot p(t_{cb} \mid \boldsymbol{\omega}_{cb}, t_{dc}, \Xi) $$
    The measure change for sampling $\mathbf{b}$ as a distance and direction from $\mathbf{c}$ is:
    $$ p(\mathbf{b}, t_{dc} \mid \Xi) = p(t_{cb}, \boldsymbol{\omega}_{cb}, t_{dc} \mid \Xi) G(\mathbf{b}, \mathbf{c}) \quad \text{(Eq. 9)} $$

2.  **Bidirectional Factorization:** Vertices $\mathbf{b}$ and $\mathbf{c}$ are sampled from opposite directions [S1].
    $$ p(t_{ab}, \boldsymbol{\omega}_{ab}, t_{dc} \mid \Xi) = p(\boldsymbol{\omega}_{ab} \mid \Xi) \cdot p(t_{ab} \mid \boldsymbol{\omega}_{ab}, \Xi) \cdot p(t_{dc} \mid \boldsymbol{\omega}_{ab}, t_{ab}, \Xi) $$
    The measure change for sampling $\mathbf{b}$ as a distance and direction from $\mathbf{a}$ is:
    $$ p(\mathbf{b}, t_{dc} \mid \Xi) = p(t_{ab}, \boldsymbol{\omega}_{ab}, t_{dc} \mid \Xi) G(\mathbf{a}, \mathbf{b}) \quad \text{(Eq. 13)} $$

**Analytic Sampling (Unidirectional, Isotropic Scattering)**
For isotropic scattering, fully analytic formulas are derived [S1].

1.  **Sample $t_{dc}$:**
    The marginal PDF for $t_{dc}$ is derived by integrating out $\boldsymbol{\omega}_{cb}$ and $t_{cb}$ [S1].
    $$ p(t_{dc} \mid \Xi) = C_\Xi \frac{\pi}{t_{dc}^2} \quad \text{(Eq. 26, inferred from context of normalization)} $$
    *Note: The exact closed form for $p(t_{dc})$ requires the normalization constant $C_\Xi$ which depends on the specific geometry of $\mathbf{a}$ and $\mathbf{d}$. The source provides the intermediate integral result for the angular part.*

2.  **Sample $\boldsymbol{\omega}_{cb}$:**
    Given $t_{dc}$ (and thus $\mathbf{c}$), sample direction $\boldsymbol{\omega}_{cb}$ [S1].
    $$ p(\boldsymbol{\omega}_{cb} \mid t_{dc}, \Xi) = \frac{\pi - \theta_{cb}}{\pi^3 \sin \theta_{cb}} \quad \text{(Eq. 27)} $$
    Where $\theta_{cb}$ is the angle between $\boldsymbol{\omega}_{cb}$ and the vector $\mathbf{c} - \mathbf{a}$ [S1].
    *Pitfall:* This PDF diverges as $\theta_{cb} \to 0$. This singularity cancels with the singularity in the path contribution function, reducing variance [S1].

3.  **Sample $t_{cb}$:**
    Given $\boldsymbol{\omega}_{cb}$ and $t_{dc}$, sample distance $t_{cb}$ from $\mathbf{c}$ along $\boldsymbol{\omega}_{cb}$ to get $\mathbf{b}$ [S1].
    $$ p(t_{cb} \mid \boldsymbol{\omega}_{cb}, t_{dc}, \Xi) = \frac{t_{ca} \sin \theta_{cb}}{\pi - \theta_{cb}} \frac{1}{h_{ca^\perp}^2 + (t_{ca^\perp} - t_{cb})^2} \quad \text{(Eq. 23)} $$
    Where:
    *   $t_{ca} = \|\mathbf{a} - \mathbf{c}\|$ [S1].
    *   $h_{ca^\perp}$ is the perpendicular distance from $\mathbf{a}$ to the ray $(\mathbf{c}, \boldsymbol{\omega}_{cb})$ [S1].
    *   $t_{ca^\perp}$ is the distance from $\mathbf{c}$ to the projection of $\mathbf{a}$ onto the ray $(\mathbf{c}, \boldsymbol{\omega}_{cb})$ [S1].
    *   The denominator is $t_{ba}^2$, the squared distance between $\mathbf{b}$ and $\mathbf{a}$ [S1].

**Pseudocode (Unidirectional, Isotropic)**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sample_joint_path_vertex_unidirectional(a, omega_la, d, omega_dc):
    """
    Samples vertices b and c connecting light endpoint a and eye endpoint d.
    Assumes isotropic scattering.
    """
    # 1. Sample t_dc
    # The PDF is proportional to 1/t_dc^2. 
    # C_Xi normalization depends on geometry. 
    # For implementation, one typically samples from the derived marginal.
    # Source Eq 26 implies p(t_dc) ~ 1/t_dc^2. 
    # We assume a valid sampling routine for this marginal exists or is precomputed.
    t_dc = sample_from_p_tdc(a, d, omega_dc) 
    
    # Calculate c
    c = d + omega_dc * t_dc
    
    # 2. Sample omega_cb
    # Geometry for theta_cb: angle between omega_cb and (a - c)
    # PDF: (pi - theta_cb) / (pi^3 * sin(theta_cb))
    # This requires sampling a direction on the hemisphere around the vector (a-c)
    # or solving the inverse CDF for the angle theta_cb.
    theta_cb = sample_theta_cb(a, c)
    phi_cb = uniform(0, 2*pi)
    
    # Construct omega_cb in a local frame aligned with (a-c)
    # Note: The source does not provide the explicit coordinate transformation 
    # for generating the vector from theta_cb and phi_cb. 
    # [G] Standard spherical coordinate generation relative to a reference vector.
    ref_vec = normalize(a - c)
    omega_cb = local_to_world_direction(theta_cb, phi_cb, ref_vec)
    
    # 3. Sample t_cb
    # Calculate geometric helpers
    t_ca = length(a - c)
    # Project a onto ray (c, omega_cb)
    # t_ca_perp is the signed distance from c to the projection
    t_ca_perp = dot(a - c, omega_cb)
    # h_ca_perp is the perpendicular distance
    projection_point = c + omega_cb * t_ca_perp
    h_ca_perp = length(a - projection_point)
    
    # PDF: C / (h^2 + (t_perp - t)^2)
    # This is a Cauchy/Lorentzian distribution shifted by t_ca_perp.
    # Scale parameter is h_ca_perp.
    # Sample t_cb from Cauchy distribution centered at t_ca_perp with scale h_ca_perp
    # weighted by the constant factor.
    t_cb = sample_cauchy(t_ca_perp, h_ca_perp)
    
    # Calculate b
    b = c + omega_cb * t_cb
    
    return b, c
```

## Subtopics

### [Bridge Sampling](3881-bridge-sampling.md)
Bridge Sampling is a method for sampling a subpath (bridge) of arbitrary vertex count connecting two vertices in a participating medium, where the probability density is proportional to all inner phase functions and reciprocal squared distance terms. It sets itself apart from sibling methods like Equiangular Sampling and MVNEE by fixing the endpoint via rotation and uniform scaling of a preliminary path, which preserves angles and handles highly-peaked forward scattering more robustly. Unlike Path Guiding, it does not use memory to store sampling densities or handle visibility explicitly during the sampling construction.

### [Bidirectional Path Tracing](6666-bidirectional-path-tracing.md)
Bidirectional Path Tracing (BPT) is a light transport algorithm that constructs a full path by combining a light subpath and a camera subpath, connecting their endpoints with a deterministic segment. It solves the high variance of unidirectional sampling by including both emitted radiance and sensor importance in the path PDF, which is particularly effective for scenes with refractive boundaries or complex light transport. Unlike its sibling [Bridge Sampling](3881-bridge-sampling.md), which uses probabilistic connections, BPT connects subpath endpoints deterministically, yielding $k+2$ different sampling techniques for a $k$-length path.

## Operating characteristics
*   **Complexity:** The unidirectional analytic sampling requires $O(1)$ operations for isotropic scattering [S1]. For anisotropic scattering, tabulation is used, increasing complexity based on table resolution [S1].
*   **Variance:** Uses 3 random decisions (direction and distance from $\mathbf{d}$, direction from $\mathbf{c}$) compared to 0, 1, or 2 for Equiangular Sampling and VRLs, significantly reducing variance [S1].
*   **Singularity Handling:** The sampling of $\boldsymbol{\omega}_{cb}$ cancels the geometric singularity in the path contribution function [S1].
*   **Limitations:** The analytic formulas are derived for isotropic scattering; anisotropic cases require tabulation or numerical integration [S1].

## Background
Joint Path Vertex Sampling was introduced by Kulla and Fajardo in 2012 [S1]. It extends earlier work on equiangular sampling and Virtual Ray Lights (VRLs) [S1]. VRLs were originally a many-light method reinterpreted as a bidirectional segment-segment connection technique [S1]. JPVS generalizes this by adding a direction sampling step to reduce variance further [S1].

## Sources
[S1] Joint importance sampling of low-order volumetric scattering, https://doi.org/10.1145/2508363.2508411, fetched 2026-10-28
