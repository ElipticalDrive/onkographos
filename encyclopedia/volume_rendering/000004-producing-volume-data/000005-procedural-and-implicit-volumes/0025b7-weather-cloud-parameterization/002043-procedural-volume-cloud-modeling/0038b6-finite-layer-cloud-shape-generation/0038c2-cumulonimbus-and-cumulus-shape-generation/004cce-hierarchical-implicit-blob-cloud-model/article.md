# Hierarchical Implicit Blob Cloud Model

Also called: Cloud Shape Model, Implicit Surface Cloud

## Overview
The Hierarchical Implicit Blob Cloud Model is a procedural method for generating the irregular, multi-lobed shapes of Cumulus and Cumulonimbus clouds by recursively layering quasi-spherical particles [S1]. It solves the problem of creating realistic cloud tops and bases with continuous furrows, distinguishing itself from explicit tessellation approaches by using a particle system to extract isosurfaces and place subsequent levels [S1].

## How it works

### Inputs and Outputs
**Inputs:**
*   Level 0 particle configuration: A set of initial particles $P_0$, either procedurally generated on a horizontal plane or manually placed by a user to define features like turrets [S1].
*   Global parameters: Flattening range, radius reduction factor, blending amounts, and repulsion offsets [S1].

**Outputs:**
*   A hierarchy of implicit surfaces $S_l$ for $l = 0, \dots, L$, where $S_L$ defines the final cloud shape [S1].
*   A mesh representation of the final surface (via isosurface extraction) suitable for rendering with specific shaders [S1].

### Data Structures
The model relies on a hierarchical list of particles [S1]. Each particle $p_i$ at level $l$ stores:
*   **Position $P_i$**: 3D coordinates on the surface $S_{l-1}$ [S1].
*   **Radius $r_i$**: Scalar value, typically $r_i \approx \frac{1}{3} r_{parent} \times (1 + \delta)$, where $\delta \in [-0.3, 0.3]$ [S1].
*   **Flattening $e_i$**: Scalar value, randomly chosen from $[0.1, 1.0]$ [S1].
*   **Local Potential Field**: Implicit function values used for surface extraction and neighbor interaction [S1].

### Numbered Steps

1.  **Initialize Level 0 ($l=0$):**
    *   If procedural: Generate a small number of particles randomly positioned on a horizontal plane or in a small subspace [S1].
    *   If manual: Allow user placement to shape features (e.g., turrets) [S1].
    *   Define the initial surface $S_0$ as the union of the implicit fields of these particles [S1].

2.  **Iterate for Levels $l = 1$ to $L$:**
    *   **Extract Surface $S_{l-1}$:** Use a particle system to extract the isosurface of the previous level $S_{l-1}$ [S1]. This involves simulating particles that repel each other and grow to populate the available surface area of $S_{l-1}$ [S1].
        *   *Growth Logic:* Particles grow until they reach a target radius or energy [S1]. If a particle grows too much or an area is uncrowded, split the particle [S1]. If an area is overcrowded, remove particles [S1].
        *   *Target Distance:* Maintain a target inter-particle distance (typically twice the target particle radius) to ensure equilibrium [S1].
    *   **Place Level $l$ Particles:** Use the extracted isosurface points from the previous step as the initial positions for new particles at level $l$ [S1].
    *   **Define Particle Properties:**
        *   Set radius $r_i$ based on the parent radius $r_{parent}$: $r_i = \frac{1}{3} r_{parent} \times (1 + \text{random}(-0.3, 0.3))$ [S1].
        *   Set flattening $e_i$ randomly from $[0.1, 1.0]$ [S1].
        *   Set blending parameters $b$ and $I$ based on level $l$ (see Parameters) [S1].
    *   **Compute Implicit Field:** For each particle $i$ at level $l$, compute its implicit field $f_i(P)$ (see Formulas) [S1].
    *   **Combine Level:** The surface $S_l$ is defined by the maximum potential of all particles at level $l$ and the previous level's surface:
        $$ S_l = \{ P \in \mathbb{R}^3 \mid \max(f_{l-1}(P), \max_i f_i(P)) = 1 \} $$
        *(Note: The source defines $f_l(P)$ as $\max(f_{l-1}(P), \max_i f_i(P))$ implicitly by stating the surface is defined from this max [S1].)*

3.  **Render Final Shape ($S_L$):**
    *   Extract the mesh of $S_L$ [S1].
    *   Apply a Gardner-like shader for view-dependent transparency near silhouettes [S1].
    *   Apply Perlin solid texture for lower-scale details [S1].
    *   (Optional) Apply "normal blending" shader to smooth furrows near the base surface $S_{L-1}$ [S1].

### Formulas

**Implicit Surface Definition:**
Each particle $i$ defines a surface $S_i$:
$$ S_i = \{ P \in \mathbb{R}^3 \mid f_i(P) = 1 \} $$
where:
$$ f_i(P) = g_i(P) + h_i(P) $$

**Base Shape $g_i(P)$:**
A flattened sphere:
$$ g_i(P) = \exp \left( 1 - \frac{d_i}{r_i (1 - e_i d_{l-1})} \right) $$
*   $d_i$: Distance from point $P$ to particle center $P_i$.
*   $r_i$: Radius of particle $i$.
*   $e_i$: Flattening factor for particle $i$.
*   $d_{l-1}$: Distance from point $P$ to the previous level surface $S_{l-1}$.

**Deformation Field $h_i(P)$:**
$$ h_i(P) = m_i(P) + n_i(P) + o_i(P) $$

**1. Repulsion Component $m_i(P)$:**
Prevents overlap and creates furrows:
$$ m_i(P) = \sum_j (1 - \epsilon - g_j(P)) \min(1, g_j^2(P)) $$
*   Sum is over neighboring particles $j$ [S1].
*   $\epsilon$: Offset parameter for repulsion strength [S1].

**2. Blending Component $n_i(P)$:**
Enlarges the blob near the base surface $S_{l-1}$ to prevent "popping" [S1]. The source does not provide the explicit mathematical formula for $n_i(P)$, only its parameters [S1].
*   $b$: Blending amount parameter [S1].
*   $I$: Parameter adjusting the fraction of the bubble enlarged [S1].

**3. Floor Repulsion Component $o_i(P)$:**
Creates a flat bottom [S1]. The source does not provide the explicit mathematical formula for $o_i(P)$, only its parameters [S1].
*   $h_0$: Reference height for the cloud floor [S1].
*   $\alpha_h$: Stiffness of repulsion for the flat bottom [S1].

**Rendering Transparency $t$:**
Gardner-like shader for fuzzy layer:
$$ t = (1 - \alpha \vec{N} \cdot \vec{C})^{\alpha_s} $$
*   $\vec{N}$: Local normal (gradient of the field).
*   $\vec{C}$: Camera direction.
*   $\alpha$: Controls thickness of the transparent margin.
*   $\alpha_s$: Controls sharpness of the transparent margin.

**Normal Blending (Optional):**
Interpolates normals near the base surface $S_{l-1}$:
$$ \vec{N}' = (1 - p \alpha_N) \vec{N} + p \alpha_N \vec{g} $$
*   $p$: Normalized distance to the base surface ($1 - \text{potential}$).
*   $\vec{g}$: Normalized gradient of $S_{l-1}$.
*   $\alpha_N$: Controls the blending strength.

### Parameters

| Parameter | Description | Typical Value / Range |
| :--- | :--- | :--- |
| $\epsilon$ | Offset for repulsion in $m_i(P)$. Controls furrow size. | $0.5$ [S1] |
| $b$ | Blending amount in $n_i(P)$. | $0$ for levels 0, 1; $1.0$ for level 2 [S1] |
| $I$ | Fraction of bubble enlarged in $n_i(P)$. | $10$ [S1] |
| $h_0$ | Reference height for cloud floor. | $100$ m [S1] |
| $\alpha_h$ | Stiffness of repulsion for flat bottom in $o_i(P)$. | $100$ m [S1] |
| $e_i$ | Flattening factor for particle $i$. | Randomly chosen between $0.1$ and $1$ [S1] |
| $r_i$ | Radius of particles at level $l$. | $\frac{1}{3} r_{l-1} \times (1 \pm 0.3)$ [S1] |
| $\alpha$ | Transparency thickness control. | Not given in the sources. |
| $\alpha_s$ | Transparency sharpness control. | Not given in the sources. |
| $\alpha_N$ | Normal blending strength. | Not given in the sources. |

### Edge Cases and Pitfalls
*   **Furrows vs. Contact:** Blobs should not stick together; real cloud blobs have furrows [S1]. The offset parameter $\epsilon$ in $m_i(P)$ must be set $>0$ (e.g., $0.5$) to repulse surfaces more than for simple contact [S1].
*   **Level 0 Initialization:** Level 0 has no previous surface $S_{-1}$ [S1]. It must be created procedurally or manually [S1]. Procedural generation on a horizontal plane may result in flat bases; manual placement allows for complex features like turrets [S1].
*   **Particle Splitting/Removal:** During the isosurface extraction for level $l$, particles must be split if they grow too much or if areas are uncrowded, and removed if overcrowded, to maintain a natural, non-uniform distribution [S1].
*   **Formula Gaps:** The explicit mathematical formulas for $n_i(P)$ and $o_i(P)$ are not provided in the source snippets, only their parameters and qualitative descriptions [S1]. Implementers must infer these from standard implicit blob techniques or the full paper [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import random

def distance(p1, p2):
    return math.sqrt(sum((a - b) ** 2 for a, b in zip(p1, p2)))

def dot(v1, v2):
    return sum(a * b for a, b in zip(v1, v2))

def get_dist_to_prev_surface(P):
    """Returns distance from P to the previous level surface S_{l-1}.
    In practice, this is evaluated using the potential field of level l-1."""
    return 0.0  # Placeholder for actual distance field query

def compute_base_shape(P, particle):
    """Computes g_i(P) = exp(1 - d_i / (r_i * (1 - e_i * d_{l-1})))"""
    d_i = distance(P, particle['pos'])
    d_l_minus_1 = get_dist_to_prev_surface(P)
    r = particle['radius']
    e = particle['flattening']
    denom = r * (1 - e * d_l_minus_1)
    if abs(denom) < 1e-6:
        return 0.0
    return math.exp(1 - d_i / denom)

def compute_repulsion(P, particle, neighbors):
    """Computes m_i(P) = sum_j (1 - epsilon - g_j(P)) * min(1, g_j(P)^2)."""
    epsilon = 0.5
    m_sum = 0.0
    for neighbor in neighbors:
        g_j = compute_base_shape(P, neighbor)
        m_sum += (1 - epsilon - g_j) * min(1.0, g_j ** 2)
    return m_sum

def compute_deformation(P, particle, neighbors):
    """Computes h_i(P) = m_i(P) + n_i(P) + o_i(P)."""
    m_i = compute_repulsion(P, particle, neighbors)
    # The explicit formulas for n_i(P) and o_i(P) are not provided in the sources [S1].
    # n_i(P) uses parameters b and I to enlarge the blob near the base surface.
    # o_i(P) uses parameters h0 and alpha_h to create a flat bottom.
    n_i = 0.0
    o_i = 0.0
    return m_i + n_i + o_i

def compute_implicit_field(P, particle, neighbors):
    """Computes f_i(P) = g_i(P) + h_i(P)."""
    g_i = compute_base_shape(P, particle)
    h_i = compute_deformation(P, particle, neighbors)
    return g_i + h_i

def get_neighbors(p, particles, radius):
    return [p2 for p2 in particles if distance(p['pos'], p2['pos']) < radius]

def initialize_particles_on_surface(particles_prev):
    """Generates initial particles on the isosurface of the previous level."""
    # In practice, samples points on the implicit surface of level l-1
    return []  # Placeholder for surface sampling logic

def extract_isosurface_level(particles_prev, target_radius):
    """Extracts isosurface of S_{l-1} using a particle system simulation."""
    current_particles = initialize_particles_on_surface(particles_prev)
    max_iterations = 100
    for _ in range(max_iterations):
        converged = True
        for p in current_particles:
            # Grow particle towards target radius
            p['radius'] += 0.1
            if p['radius'] > target_radius * 1.1:
                p['radius'] = target_radius * 1.1
                converged = False

            neighbors = get_neighbors(p, current_particles, radius=target_radius * 2)
            # Simulate repulsion by adjusting radius based on neighbor density
            repulsion = compute_repulsion(p['pos'], p, neighbors)
            if repulsion > 0.5:
                p['radius'] -= 0.05
            else:
                p['radius'] += 0.05

            # Split if too big or uncrowded
            if p['radius'] > target_radius * 1.05 or len(neighbors) < 2:
                # Split logic: create new particles at p['pos'] with smaller radius
                pass
            # Remove if overcrowded
            if len(neighbors) > 10:
                pass  # Remove particle logic

        if converged:
            break
    return current_particles

def render_cloud(mesh, camera_dir):
    """Renders the final cloud mesh using Gardner-like shader and Perlin texture."""
    for vertex in mesh.vertices:
        N = vertex.normal
        C = camera_dir
        dot_product = dot(N, C)
        alpha = 1.0
        alpha_s = 2.0
        transparency = (1 - alpha * dot_product) ** alpha_s
        texture_val = 0.5  # Placeholder for Perlin noise evaluation
        color = (texture_val, texture_val, texture_val) * (1 - transparency)
    return color
```

## Operating characteristics

**Time Complexity:**
*   Shape generation is recursive. Each level $l$ requires extracting the isosurface of $S_{l-1}$ using a particle system simulation, which is iterative and depends on the number of particles and convergence criteria [S1].
*   The authors state that rendering and animation are fast because the model does not rely on volumetric representation or voxel discretization [S1].

**Memory Complexity:**
*   Stores the hierarchy of particles. The number of particles increases with each level, but radii decrease by a factor of ~3, potentially limiting the total count if surface area coverage is maintained [S1].

**Measured Performance:**
*   The authors state that rendering and animation are fast because the model does not rely on volumetric representation or voxel discretization [S1].
*   Specific hardware or resolution benchmarks are not given in the sources [S1].

**Strengths:**
*   Realistic shapes with continuous furrows [S1].
*   No voxel discretization artifacts [S1].
*   Fast rendering due to explicit mesh/shader approach [S1].
*   Hierarchical structure allows for multi-scale detail [S1].

**Limitations:**
*   Relies on heuristic parameters (e.g., $\epsilon$, $b$, $I$) which may require tuning [S1].
*   The explicit formulas for blending ($n_i$) and floor repulsion ($o_i$) are not fully provided in the source, making implementation ambiguous [S1].
*   Does not include volumetric rendering effects (e.g., light scattering) in the shape model itself, requiring additional shaders for realistic appearance [S1].

**When to Use:**
*   When generating realistic Cumulus/Cumulonimbus cloud shapes for real-time rendering or animation where performance is critical [S1].
*   When explicit mesh output is desired for further processing or animation [S1].

**When Not to Use:**
*   When volumetric light scattering or complex internal cloud structure is required (the model only defines shape) [S1].
*   When exact mathematical continuity of blending/floor effects is critical and the missing formulas cannot be inferred [S1].

## Background

**Origin:**
*   Proposed by Fabrice Neyret and A. Bouthors from EVASION-GRAVIR/IMAG-INRIA [S1].
*   Published in 2004 in the paper "Modeling Clouds Shape" [S1].

**History and Development:**
*   The model draws on and extends several previous approaches, including implicit surfaces [Gas93] and particle-based surface extraction [WH94, CA97] [S1].
*   It contrasts with explicit tessellation methods like [NND96] by using a particle system for level generation [S1].
*   The model is designed to be integrated with particle animation and advanced rendering, though the paper focuses on shape definition [S1].

**Applications:**
*   Procedural generation of cloud shapes for computer graphics, particularly in real-time rendering and animation [S1].

## Sources

[S1] Modeling Clouds Shape, https://inria.hal.science/inria-00537462v1, fetched 2026-10-26
