# Joint Path Vertex Sampling

Also called: 3-random-decision connection, connection subpath sampling

## Overview
Joint Path Vertex Sampling is a method for constructing a full light transport path by connecting a light subpath endpoint $\mathbf{a}$ and an eye subpath endpoint $\mathbf{d}$ via a connection subpath containing two new vertices $\mathbf{b}$ and $\mathbf{c}$ [S1]. It solves the high variance associated with standard vertex-vertex connections in participating media by using three random decisions (two distances and one direction) to sample the connection subpath from a joint probability density function (PDF) proportional to the geometry and scattering terms [S1]. Unlike sibling methods such as [Equiangular Sampling](link), [Virtual Ray Lights](6665-virtual-ray-lights-vrls.md), and [Bridge Sampling](3881-bridge-sampling.md) which use 0, 1, or 2 random decisions, this method explicitly samples the intermediate direction $\boldsymbol{\omega}_{cb}$ to cancel out singularities in the path contribution function, thereby reducing variance [S1].

## How it works
**Inputs and Outputs**
*   **Input:** Vertex $\mathbf{a}$ with incident direction $\boldsymbol{\omega}_{la}$ (end of light subpath); vertex $\mathbf{d}$ with outgoing direction $\boldsymbol{\omega}_{dc}$ (end of eye subpath) [S1].
*   **Output:** Vertices $\mathbf{b}$ and $\mathbf{c}$, and the complete path segment $\mathbf{a} \to \mathbf{b} \to \mathbf{c} \to \mathbf{d}$ [S1].

**Data Structures**
The method operates on geometric primitives [S1]:
*   Points $\mathbf{p} \in \mathbb{R}^3$.
*   Unit direction vectors $\boldsymbol{\omega} \in \mathbb{R}^3$.
*   Scalar distances $t \in \mathbb{R}^+$.
*   Configuration $\Xi$: A container holding $\mathbf{a}, \boldsymbol{\omega}_{la}, \mathbf{d}, \boldsymbol{\omega}_{dc}$ and medium coefficients [S1].

**Formulas**
The target joint PDF for the connection subpath vertices $\mathbf{b}$ and distance $t_{dc}$ is defined as [S1]:
$$ p(\mathbf{b}, t_{dc} \mid \Xi) = C_\Xi G(\mathbf{abcd}) \rho(\mathbf{abc}) $$
where:
*   $G(\mathbf{abcd}) = G(\mathbf{a}, \mathbf{b}) G(\mathbf{b}, \mathbf{c}) G(\mathbf{c}, \mathbf{d})$ is the product of geometry terms between consecutive vertices [S1].
*   $\rho(\mathbf{abc}) = \rho(\mathbf{a}) \rho(\mathbf{b}) \rho(\mathbf{c})$ is the product of scattering terms (phase functions or albedo) at vertices $\mathbf{a}, \mathbf{b}, \mathbf{c}$ [S1].
*   $\rho(\mathbf{d})$ is excluded because the outgoing direction $\boldsymbol{\omega}_{dc}$ is fixed by the input [S1].
*   $C_\Xi$ is a normalization factor dependent on the input configuration [S1].

**Factorizations**
Two factorizations of the joint PDF are supported, determining the order of sampling [S1]:

1.  **Unidirectional Factorization:** Vertices $\mathbf{c}$ and $\mathbf{b}$ are sampled from the same direction starting from $\mathbf{d}$ [S1].
    $$ p(t_{cb}, \boldsymbol{\omega}_{cb}, t_{dc} \mid \Xi) = p(t_{dc} \mid \Xi) \, p(\boldsymbol{\omega}_{cb} \mid t_{dc}, \Xi) \, p(t_{cb} \mid \boldsymbol{\omega}_{cb}, t_{dc}, \Xi) $$
    Steps:
    1.  Sample distance $t_{dc}$ from $\mathbf{d}$ along $\boldsymbol{\omega}_{dc}$ to get $\mathbf{c}$ [S1].
    2.  Sample direction $\boldsymbol{\omega}_{cb}$ from $\mathbf{c}$ [S1].
    3.  Sample distance $t_{cb}$ from $\mathbf{c}$ along $\boldsymbol{\omega}_{cb}$ to get $\mathbf{b}$ [S1].

2.  **Bidirectional Factorization:** Vertices $\mathbf{b}$ and $\mathbf{c}$ are sampled from opposite directions [S1].
    $$ p(t_{ab}, \boldsymbol{\omega}_{ab}, t_{dc} \mid \Xi) = p(\boldsymbol{\omega}_{ab} \mid \Xi) \, p(t_{ab} \mid \boldsymbol{\omega}_{ab}, \Xi) \, p(t_{dc} \mid \boldsymbol{\omega}_{ab}, t_{ab}, \Xi) $$
    Steps:
    1.  Sample direction $\boldsymbol{\omega}_{ab}$ from $\mathbf{a}$ [S1].
    2.  Sample distance $t_{ab}$ from $\mathbf{a}$ along $\boldsymbol{\omega}_{ab}$ to get $\mathbf{b}$ [S1].
    3.  Sample distance $t_{dc}$ from $\mathbf{d}$ along $\boldsymbol{\omega}_{dc}$ to get $\mathbf{c}$ [S1].

**Analytic Sampling (Unidirectional, Isotropic)**
For isotropic scattering, fully analytic formulas exist for the unidirectional factorization [S1].

1.  **Sampling $t_{dc}$:**
    The marginal PDF for $t_{dc}$ is derived by integrating out $\boldsymbol{\omega}_{cb}$ and $t_{cb}$ [S1].

2.  **Sampling $\boldsymbol{\omega}_{cb}$:**
    Given $t_{dc}$ (and thus $\mathbf{c}$), the conditional PDF for the direction $\boldsymbol{\omega}_{cb}$ is [S1]:
    $$ p(\boldsymbol{\omega}_{cb} \mid t_{dc}, \Xi) = \frac{\pi - \theta_{cb}}{\pi^3 \sin \theta_{cb}} $$
    where $\theta_{cb}$ is the angle between $\boldsymbol{\omega}_{cb}$ and the vector $\mathbf{c} - \mathbf{a}$ (length $t_{ca}$) [S1].
    *   **Pitfall:** This PDF diverges as $\theta_{cb} \to 0$. Sampling from this distribution cancels the corresponding singularity in the path contribution function, reducing variance [S1].

3.  **Sampling $t_{cb}$:**
    Given $\boldsymbol{\omega}_{cb}$ and $t_{dc}$, the conditional PDF for $t_{cb}$ is [S1]:
    $$ p(t_{cb} \mid \boldsymbol{\omega}_{cb}, t_{dc}, \Xi) = \frac{t_{ca} \sin \theta_{cb}}{\pi - \theta_{cb}} \frac{1}{h_{ca^\perp}^2 + (t_{ca^\perp} - t_{cb})^2} $$
    where:
    *   $h_{ca^\perp}$ is the perpendicular distance between vertex $\mathbf{a}$ and the ray $(\mathbf{c}, \boldsymbol{\omega}_{cb})$ [S1].
    *   $t_{ca^\perp}$ is the distance from $\mathbf{c}$ to the projection of $\mathbf{a}$ onto the ray $(\mathbf{c}, \boldsymbol{\omega}_{cb})$ [S1].
    *   The denominator term $h_{ca^\perp}^2 + (t_{ca^\perp} - t_{cb})^2$ is equal to $t_{ba}^2$, the squared distance between $\mathbf{b}$ and $\mathbf{a}$ [S1].
    *   This PDF is identical to that of Equiangular Sampling when $\mathbf{a}, \boldsymbol{\omega}_{cb}, \mathbf{c}$ are fixed [S1].

**Pseudocode**
The following implements the Unidirectional Factorization for isotropic scattering.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import random

def dot(v1, v2):
    return v1[0]*v2[0] + v1[1]*v2[1] + v1[2]*v2[2]

def cross(v1, v2):
    return (v1[1]*v2[2] - v1[2]*v2[1],
            v1[2]*v2[0] - v1[0]*v2[2],
            v1[0]*v2[1] - v1[1]*v2[0])

def norm(v):
    return math.sqrt(v[0]**2 + v[1]**2 + v[2]**2)

def normalize(v):
    n = norm(v)
    return (v[0]/n, v[1]/n, v[2]/n)

def sample_joint_path_vertex_sampling_unidirectional(a, omega_la, d, omega_dc):
    """
    Samples vertices b and c connecting light endpoint a and eye endpoint d.
    Uses Unidirectional Factorization with 3 random decisions.
    Assumes isotropic scattering for analytic PDFs.
    """
    # Step 1: Sample t_dc from d along omega_dc to get c
    # [S1] The marginal PDF for t_dc is not explicitly given in the sources; 
    # it requires numerical inversion or precomputed tables based on C_Xi.
    t_dc = 1.0  # Placeholder for numerical sampling routine
    c = (d[0] + t_dc * omega_dc[0], d[1] + t_dc * omega_dc[1], d[2] + t_dc * omega_dc[2])

    # Step 2: Sample direction omega_cb from c
    # PDF: p(omega_cb | t_dc, Xi) = (pi - theta_cb) / (pi^3 * sin(theta_cb))
    # theta_cb is angle between omega_cb and (a - c).
    v = (a[0] - c[0], a[1] - c[1], a[2] - c[2])
    t_ca = norm(v)
    
    # Sample theta_cb from the marginal distribution p(theta) ~ pi - theta
    # CDF: F(theta) = (2*pi*theta - theta^2) / pi^2
    # Inversion: theta = pi * (1 - sqrt(1 - u))
    u = random.uniform(0, 1)
    theta_cb = math.pi * (1 - math.sqrt(1 - u))
    phi_cb = random.uniform(0, 2 * math.pi)
    
    sin_theta = math.sin(theta_cb)
    cos_theta = math.cos(theta_cb)
    
    # Create orthonormal basis with z-axis along v
    if abs(v[2]) < 0.999999:
        up = (0, 0, 1)
    else:
        up = (1, 0, 0)
    x_axis = normalize(cross(v, up))
    y_axis = normalize(cross(x_axis, v))
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
    # PDF: p(t_cb | ...) = K / (h_perp^2 + (t_perp - t_cb)^2)
    # This is a shifted Cauchy distribution with scale h_ca_perp and location t_ca_perp.
    
    vec_ca = (a[0] - c[0], a[1] - c[1], a[2] - c[2])
    t_ca_perp = dot(vec_ca, omega_cb)
    vec_proj = (t_ca_perp * omega_cb[0], t_ca_perp * omega_cb[1], t_ca_perp * omega_cb[2])
    vec_perp = (vec_ca[0] - vec_proj[0], vec_ca[1] - vec_proj[1], vec_ca[2] - vec_proj[2])
    h_ca_perp = norm(vec_perp)
    
    # Sample from Cauchy(t_ca_perp, h_ca_perp)
    u = random.uniform(0, 1)
    t_cb = t_ca_perp + h_ca_perp * math.tan(math.pi * (u - 0.5))
    
    # Guard against negative distance [G]
    if t_cb < 0:
        t_cb = 0.0
    
    b = (c[0] + t_cb * omega_cb[0], c[1] + t_cb * omega_cb[1], c[2] + t_cb * omega_cb[2])
    
    return b, c, t_dc, t_cb, omega_cb
```

## Subtopics

### [Bridge Sampling](3881-bridge-sampling.md)
Bridge Sampling is a method for sampling a subpath (bridge) of arbitrary vertex count connecting two vertices in a participating medium, where the probability density is proportional to all inner phase functions and reciprocal squared distance terms. It sets itself apart from sibling methods like Equiangular Sampling and MVNEE by fixing the endpoint via rotation and uniform scaling of a preliminary path, which preserves angles and handles highly-peaked forward scattering more robustly. Unlike Path Guiding, it does not use memory to store sampling densities or handle visibility explicitly during the sampling construction.

### [Bidirectional Path Tracing](6666-bidirectional-path-tracing.md)
Bidirectional Path Tracing (BPT) is a stochastic light transport algorithm for participating media that constructs complete paths by connecting independent subpaths traced from light sources and the eye point. It solves the high variance and slow convergence of unidirectional methods (like standard path tracing or light tracing) in scenes with refractive boundaries or complex scattering by ensuring both emitted radiance and sensor importance are included in the sampling density. Unlike light tracing, which produces partial solutions that are not useful in short times, BPT is view-dependent and allows gradual improvement of image quality starting from a crude approximation. It is the only existing unbiased method for anisotropic media mentioned in the sources.

### [Virtual ray lights (VRLs)](6665-virtual-ray-lights-vrls.md)
Virtual ray lights (VRLs) is a many-light rendering method for participating media that reinterprets the accumulation of light transport as a bidirectional segment-segment connection technique. It traces rays from light sources, stores the first ray segment after a surface interaction as a "Virtual Ray Light," and organizes these segments into a grid hierarchy to efficiently compute multiple scattering contributions to camera rays using precomputed tables. Unlike standard path tracing which samples paths stochastically, VRLs use a deterministic lookup based on geometric relationships between light segments and eye segments, trading memory for speed in dense media. Unlike Joint Path Vertex Sampling which uses 3 random decisions, VRLs use 1 random decision.

## Operating characteristics
*   **Complexity:** The method involves calculating geometric terms ($h_{ca^\perp}, t_{ca^\perp}, \theta_{cb}$) and sampling from Cauchy and spherical distributions. The computational cost is dominated by the geometric calculations and the sampling routines, which are $O(1)$ per connection [S1].
*   **Performance:** The method reduces variance compared to vertex-vertex connections, Equiangular Sampling, and VRLs by using 3 random decisions [S1]. The explicit sampling of $\boldsymbol{\omega}_{cb}$ cancels the singularity in the path contribution function, leading to lower variance in scenes with strong scattering [S1].
*   **Limitations:** The PDF for $\boldsymbol{\omega}_{cb}$ diverges as $\theta_{cb} \to 0$, requiring careful sampling to avoid numerical instability [S1]. The method assumes isotropic scattering for the fully analytic unidirectional formulas; anisotropic scattering requires tabulation or numerical methods [S1].
*   **When to use:** Use when variance reduction is critical in participating media, especially when connecting subpaths with arbitrary lengths. It is particularly effective when the geometry configuration allows for the cancellation of singularities [S1].

## Background
Joint Path Vertex Sampling was introduced by Kulla and Fajardo in 2012 [S1]. It was developed to address the limitations of existing connection techniques (0, 1, or 2 random decisions) by introducing a 3-random-decision approach. The method builds on the principles of conservation of energy and the path integral formulation of light transport. It is used in volumetric path tracing to improve the efficiency of multiple scattering calculations [S1].

## Sources
[S1] Joint importance sampling of low-order volumetric scattering, https://doi.org/10.1145/2508363.2508411, fetched 2026-10-28
