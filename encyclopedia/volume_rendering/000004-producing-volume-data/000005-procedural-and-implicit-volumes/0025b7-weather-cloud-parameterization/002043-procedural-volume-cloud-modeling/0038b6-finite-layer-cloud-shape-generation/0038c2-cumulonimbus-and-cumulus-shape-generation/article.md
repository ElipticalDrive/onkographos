# Cumulonimbus and Cumulus Shape Generation

Also called: Cumulus and Cumulonimbus Shape Generation

## Overview
Cumulonimbus and Cumulus Shape Generation is a procedural method for generating non-spherical, multi-lobed cloud shapes for Cumulus (type 9) and Cumulonimbus (type 10) clouds, creating irregular tops and bases rather than simple spheres [S1, unofficial]. It differs from its parent method, Finite Layer Cloud Shape Generation, which is designed for layer cloud types (Cirrus, Altostratus, Nimbostratus, Stratocumulus, Stratus) using a chain of connected 3D ellipsoidal lobes [8292-finite-layer-cloud-shape-generation.md]. While the parent method ensures clouds do not degenerate into simple cylinders along the Z-axis, this method specifically addresses the vertical development and puffy, irregular structure characteristic of convective clouds.

## How it works
**Inputs and Outputs:**
*   **Inputs:** Cloud type ID (9 for Cumulus, 10 for Cumulonimbus), style offset (31009 for realistic, 41003 for cartoon), and a random number generator (RNG).
*   **Outputs:** A list of 3D ellipsoidal lobes, each defined by a center position $(cx, cy, cz)$ and radii $(rx, ry, rz)$.

**Data Structures:**
Each lobe is represented as a tuple or struct containing:
*   `cx, cy, cz`: Float coordinates of the lobe center.
*   `rx, ry, rz`: Float radii along the X, Y, and Z axes.

**Steps:**

1.  **Initialize RNG:**
    Set the random seed using the formula:
    $$ \text{seed} = \text{base\_seed} + \text{type\_id} \times 1009 + \text{style\_offset} $$
    Where `type_id` is 9 for Cumulus and 10 for Cumulonimbus. `style_offset` is 31009 for realistic style and 41003 for cartoon style [S1, unofficial].

2.  **Generate Center:**
    Generate the base center coordinates $(center\_x, center\_y)$ from a uniform distribution over the interval $[-0.12, 0.12]$ [S1, unofficial].

3.  **Generate Lobes based on Type:**

    *   **If Type 9 (Cumulus):**
        *   **Base Lobes:** Generate 3 to 6 base lobes.
            *   $cx, cy$: Generated from a uniform distribution around $center\_x, center\_y$.
            *   $cz$: Generated from uniform distribution in $[-0.42, -0.16]$.
            *   $rx$: Generated from uniform distribution in $[0.27, 0.42]$.
            *   $ry$: Generated from uniform distribution in $[0.23, 0.37]$.
            *   $rz$: Generated from uniform distribution in $[0.14, 0.25]$.
        *   **Tower Lobes:** Generate 5 to 10 tower lobes.
            *   $direction$: Generated from uniform distribution in $[-\pi, \pi]$.
            *   $radial$: Generated from uniform distribution in $[0.06, 0.52]$.
            *   $cx, cy$: Offset from the center by $radial$ in the direction $direction$.
            *   $cz$: Generated from uniform distribution in $[-0.06, 0.54]$.
            *   $rx$: Generated from uniform distribution in $[0.17, 0.34]$.
            *   $ry$: Generated from uniform distribution in $[0.16, 0.32]$.
            *   $rz$: Generated from uniform distribution in $[0.18, 0.36]$.

    *   **If Type 10 (Cumulonimbus):**
        *   Generate a variable number of lobes (implied by the use of index).
        *   Calculate progress: $progress = \frac{index}{lobe\_count - 1}$.
        *   $cx, cy$: Spread increases with progress. The base spread is $0.13 + progress \times 0.08$.
        *   $cz$: Rises from $-0.58$ to $0.56$, calculated as $-0.58 + progress \times 1.14$.
        *   $rx, ry$: Increase with progress, calculated as $0.24 + progress \times 0.10$.
        *   $rz$: Not explicitly defined in the source for Cumulonimbus, but implied to follow similar scaling or default to a value consistent with the vertical growth [S1, unofficial].

    *   **Other Layer Clouds (Types 1, 7):**
        *   Although this article focuses on Cumulus/Cumulonimbus, the source notes that for other layer clouds, 3-10 lobes are generated with $radial$ from uniform $[0.03, 0.52]$, $theta$ from uniform $[-\pi, \pi]$, and $cz$ from uniform $[-0.20, 0.20]$ [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import random
import math

def generate_cumulus_cumulonimbus_lobs(type_id, style_offset, base_seed):
    # 1. Initialize RNG
    seed = base_seed + type_id * 1009 + style_offset
    rng = random.Random(seed)
    
    # 2. Generate Center
    center_x = rng.uniform(-0.12, 0.12)
    center_y = rng.uniform(-0.12, 0.12)
    
    lobes = []
    
    if type_id == 9:  # Cumulus
        # Base Lobes: 3-6
        num_base = rng.randint(3, 6)
        for _ in range(num_base):
            cx = rng.uniform(center_x - 0.12, center_x + 0.12) # Approximation of "around center"
            cy = rng.uniform(center_y - 0.12, center_y + 0.12)
            cz = rng.uniform(-0.42, -0.16)
            rx = rng.uniform(0.27, 0.42)
            ry = rng.uniform(0.23, 0.37)
            rz = rng.uniform(0.14, 0.25)
            lobes.append({'cx': cx, 'cy': cy, 'cz': cz, 'rx': rx, 'ry': ry, 'rz': rz})
            
        # Tower Lobes: 5-10
        num_tower = rng.randint(5, 10)
        for _ in range(num_tower):
            direction = rng.uniform(-math.pi, math.pi)
            radial = rng.uniform(0.06, 0.52)
            cx = center_x + radial * math.cos(direction)
            cy = center_y + radial * math.sin(direction)
            cz = rng.uniform(-0.06, 0.54)
            rx = rng.uniform(0.17, 0.34)
            ry = rng.uniform(0.16, 0.32)
            rz = rng.uniform(0.18, 0.36)
            lobes.append({'cx': cx, 'cy': cy, 'cz': cz, 'rx': rx, 'ry': ry, 'rz': rz})
            
    elif type_id == 10:  # Cumulonimbus
        # Number of lobes not explicitly specified, assume a fixed or derived count.
        # Using a placeholder count for implementation completeness.
        lobe_count = 15 
        for i in range(lobe_count):
            progress = i / (lobe_count - 1)
            
            # Spread increases with progress
            spread = 0.13 + progress * 0.08
            direction = rng.uniform(-math.pi, math.pi)
            radial = rng.uniform(0, spread) # Assuming radial is within the spread
            
            cx = center_x + radial * math.cos(direction)
            cy = center_y + radial * math.sin(direction)
            
            # CZ rises from -0.58 to 0.56
            cz = -0.58 + progress * 1.14
            
            # RX, RY increase with progress
            rx = 0.24 + progress * 0.10
            ry = 0.24 + progress * 0.10
            
            # RZ not specified, using a default or derived value [G]
            rz = 0.2 + progress * 0.2 
            
            lobes.append({'cx': cx, 'cy': cy, 'cz': cz, 'rx': rx, 'ry': ry, 'rz': rz})
            
    return lobes
```

**Parameters:**
*   `style_offset`: 31009 for realistic style, 41003 for cartoon style [S1, unofficial].
*   `type_id`: 9 for Cumulus, 10 for Cumulonimbus [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Division by Zero:** In Cumulonimbus generation, if `lobe_count` is 1, `progress` calculation involves division by zero. Ensure `lobe_count > 1` [G].
*   **Precision:** The source uses floating-point uniform distributions. Ensure consistent random number generation across platforms if deterministic results are required [S1, unofficial].
*   **Boundary Conditions:** The coordinates and radii are normalized. Ensure the rendering pipeline handles these normalized coordinates correctly [S1, unofficial].

## Subtopics

### [Hierarchical Implicit Blob Cloud Model](13310-hierarchical-implicit-blob-cloud-model.md)
The Hierarchical Implicit Blob Cloud Model is a procedural method for generating the irregular, multi-lobed shapes of Cumulus and Cumulonimbus clouds by recursively layering quasi-spherical particles. It solves the problem of creating realistic cloud tops and bases with continuous furrows, distinguishing itself from explicit tessellation approaches (like [NND96]) by using a particle system to extract isosurfaces and place subsequent levels.

### [3D Gaussian Cloud Generation](13374-3d-gaussian-cloud-generation.md)
An algorithm for generating Cumulus (type 9) and Cumulonimbus (type 10) clouds using a sparse set of 3D Gaussian-distributed pseudo-spheroid primitives. It differs from particle-based systems by requiring significantly fewer primitives (typically 35) to achieve realistic results, and from hierarchical implicit blobs by using explicit geometric primitives with pre-processing filters to optimize rendering performance.

### Superellipsoid Cloud Profiling
A vertical profile function that sculpts realistic 'water balloon on a table' puffy clouds using a superellipsoid shape [S2, unofficial]. It sculpts voluminous, puffy vertical structures inside `getCloud3DProfile` / `getCloud3DSDF` using a superellipsoid profile driven by eccentricity, curvature exponent, and center distance [S2, unofficial]. Eccentricity determines the widest point of the cloud, decoded from the cell ID hash [S2, unofficial]. Curvature exponent is a parameter driving the superellipsoid profile shape [S2, unofficial].

### L-System Cloud Generation
Uses L-systems (Lindenmayer systems) with grammar rules to generate cloud shapes [S3]. It differs from Kang et al. [8] by including a proportional random factor in sphere radius and distance between primitives in each recursive call [S3]. It uses volumetric rendering and density function from Equation (2) [S3]. The host-side C++ interpreter runs before GLSL rendering, acting as a tiny interpreter for recursive generation of spheres based on an axiom and rules, generating a string of symbols [S3]. A lexical analyser processes the generated string, inserting a sphere with random radius and length for each 'X' or 'Y' item, and applying rotation angles given by '+' and '&' operators [S3]. The BNF Grammar for generation is $X \rightarrow X+YF++YF-FX-XFXF-YF+/$, where '+' means turn left with angle using a rotation matrix, and '&' means pitch down with angle using a different rotation matrix [S3]. Example parameters: Iterations = 7, Angle = 10° produces thin/strange spiral shapes; Angle [50°, 100°] with same grammar produces dense cumulus formations [S3]. The string produced by Equation (18) generates 128 spheres, ready to transfer to OpenGL shader for real-time animation [S3].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of lobes generated. The initialization and parameter generation are constant time operations per lobe.
*   **Memory Complexity:** Linear with respect to the number of lobes, as each lobe is stored as a small struct.
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** The method scales well with the number of lobes, but the visual complexity increases linearly.
*   **Strengths:** Simple to implement, provides a good base for convective cloud shapes, supports different styles (realistic/cartoon).
*   **Limitations:** Does not handle internal density variations or lighting effects; relies on downstream rendering techniques for final appearance.
*   **When to use:** When generating procedural cloud volumes for Cumulus and Cumulonimbus types where irregular, multi-lobed shapes are required.
*   **When not to use:** For layer clouds (use Finite Layer Cloud Shape Generation) or when high-fidelity density and lighting are required without additional post-processing.

## Background
The method originates from the "Noise-Cloud" project, an unofficial repository [S1, unofficial]. It is part of a broader effort to create procedural cloud models for real-time rendering. The specific parameters and generation logic for Cumulus and Cumulonimbus clouds are detailed in the source code and documentation of this project [S1, unofficial].

## Sources
[S1] mingtu149411/Noise-Cloud, https://github.com/mingtu149411/Noise-Cloud, fetched 2026-10-27 (unofficial)
[S2] Optimize cloud rendering with eroded Voronoi weather maps and stable linear raymarching, https://github.com/ricecake/boidsish/pull/679, fetched 2026-10-28 (unofficial)
[S3] Efficient Algorithms for Real-Time GPU Volumetric Cloud Rendering with Enhanced Geometry, https://doi.org/10.3390/sym10040125, fetched 2026-10-26
