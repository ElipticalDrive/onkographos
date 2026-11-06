# Reference Path Tracing

Also called: Path Traced Subsurface Scattering, Random Walk SSS, ReferencePathTracer, Volume SSS

## Overview
Reference Path Tracing is an unbiased volumetric path tracer used as a ground-truth reference renderer for subsurface scattering and participating media [S1, unofficial][S2, unofficial]. It solves the problem of accurate light transport in high-curvature surfaces and complex media by treating the subsurface layer as a volume, performing full random walks from entry to exit points [S2, unofficial]. It sets itself apart from diffusion-based models by explicitly tracking geometric features for accurate shadowing and handling zero- and single-scatter events, while differing from sibling methods like BSDF Sampling for Volumetric Paths by relying on a pure random walk with Next-Event Estimation (NEE) rather than specialized BSDF sampling strategies [S1, unofficial][S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Surface entry point, surface normal, medium extinction coefficient $\sigma_t$ (assumed homogeneous for the basic model), geometry mesh for boundary intersection, maximum bounce count, and Russian roulette start threshold [S2, unofficial][S1, unofficial].
*   **Outputs:** Radiance contribution from the path, weighted by the path throughput and surface response [S2, unofficial].

**Data Structures:**
The method is encapsulated in a struct `ReferencePathTracer` which inherits from `VolumeIntegrationMethod` [S1, unofficial].
*   `max_bounces::Int`: Maximum number of scattering events per path. Default: 64 [S1, unofficial].
*   `rr_start::Int`: Number of bounces after which Russian roulette activation begins. Default: 3 [S1, unofficial].

**Algorithm Steps:**
1.  **Initialization:** At the surface entry point, choose the initial direction $\omega_i$ from a Lambertian distribution around the inverse surface normal $-n$ [S2, unofficial].
2.  **Distance Sampling:** Advance along the current direction by a distance $s$ sampled from the transmittance distribution [S2, unofficial]. For a homogeneous medium with extinction coefficient $\sigma_t$, the transmittance $T$ over distance $s$ follows Beer's Law $T = e^{-\sigma_t s}$ [G]. To sample the distance, draw a uniform random number $\xi \in [0, 1)$ and invert the cumulative distribution function:
    $$ s = -\frac{\ln(1 - \xi)}{\sigma_t} $$
    [S2, unofficial].
3.  **Direction Update:** For isotropic scattering, update the direction for the next step by drawing a sample from a uniform spherical distribution [S2, unofficial].
4.  **Boundary Intersection:** Trace a ray from the current position along the new direction against the scene geometry to check for intersection with the exterior interface [S2, unofficial].
5.  **Termination Check (Exit):** If the ray intersects the exterior interface at distance $s_{hit} < s_{sampled}$:
    *   Terminate the random walk.
    *   Compute direct illumination at the exit point using Next-Event Estimation (NEE) [S1, unofficial].
    *   Weigh the incoming radiance by a Lambertian response function. Do not apply surface coloring effects (albedo) at this stage [S2, unofficial].
    *   Return the contribution.
6.  **Termination Check (Depth):** If the path continues for a fixed depth limit (default 256 steps) without finding an exit point, deem the photon lost and set the final contribution to zero [S2, unofficial].
7.  **Termination Check (Russian Roulette):** If the path has not exited and has not reached the fixed depth limit:
    *   If $b \ge rr\_start$, apply Russian Roulette to terminate the path stochastically based on the path throughput to maintain unbiasedness [S1, unofficial][S2, unofficial].
    *   If the path survives, update the throughput and continue to Step 2 [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import random

def cross(a, b):
    return (a[1]*b[2]-a[2]*b[1], a[2]*b[0]-a[0]*b[2], a[0]*b[1]-a[1]*b[0])

def normalize(v):
    l = math.sqrt(sum(c*c for c in v))
    return (v[0]/l, v[1]/l, v[2]/l)

def sample_lambertian(n):
    u = random.uniform(-1, 1)
    v = random.uniform(-1, 1)
    if u*u + v*v > 1: u, v = -u, -v
    x = math.sqrt(1 - u*u - v*v)
    if abs(n[0]) < 0.999: s = (1,0,0)
    else: s = (0,1,0)
    t = normalize(cross(n, s))
    b = cross(n, t)
    return (x*t[0] + v*b[0] + u*n[0], x*t[1] + v*b[1] + u*n[1], x*t[2] + v*b[2] + u*n[2])

def sample_spherical():
    phi = random.uniform(0, 2*math.pi)
    cos_t = random.uniform(-1, 1)
    sin_t = math.sqrt(1 - cos_t**2)
    return (sin_t*math.cos(phi), sin_t*math.sin(phi), cos_t)

def compute_direct_illumination(point, normal):
    """Queries scene radiance at point for direction -normal [G]."""
    return 1.0

def reference_path_tracing(entry_point, normal, sigma_t, geometry, max_bounces=64, rr_start=3, fixed_depth=256):
    """
    Performs a reference random walk for subsurface scattering.
    geometry.trace_ray(origin, direction, max_distance) returns (point, normal, dist) or None.
    """
    throughput = 1.0
    current_pos = entry_point
    current_dir = sample_lambertian((-normal[0], -normal[1], -normal[2]))
    bounces = 0

    for step in range(fixed_depth):
        xi = random.uniform(0, 1)
        distance = -math.log(1.0 - xi) / sigma_t

        hit = geometry.trace_ray(current_pos, current_dir, max_distance=distance)
        if hit is not None:
            hit_point, hit_normal, hit_dist = hit
            transmittance = math.exp(-sigma_t * hit_dist)
            radiance = compute_direct_illumination(hit_point, hit_normal)
            return throughput * transmittance * radiance

        current_pos = (current_pos[0] + current_dir[0]*distance, current_pos[1] + current_dir[1]*distance, current_pos[2] + current_dir[2]*distance)
        current_dir = sample_spherical()
        bounces += 1

        if bounces >= max_bounces:
            return 0.0

        if bounces >= rr_start:
            prob = min(1.0, max(throughput))
            if random.uniform(0, 1) > prob:
                return 0.0
            throughput /= prob

    return 0.0
```

**Parameters:**
*   `max_bounces`: 64 [S1, unofficial].
*   `rr_start`: 3 [S1, unofficial].
*   `fixed_depth`: 256 [S2, unofficial].

**Edge Cases and Pitfalls:**
*   Precision loss in `math.log(1.0 - xi)` occurs when $\xi$ is extremely close to 1; using `math.log1p(-xi)` is recommended [G].
*   Degenerate normals or zero-extinction coefficients require explicit guards to prevent division by zero [G].
*   Not given in the sources.

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of scattering events and geometry intersection tests [S2, unofficial]. Each step requires a ray trace against the geometry [S2, unofficial].
*   **Memory Complexity:** Low, as it only stores the current path state and does not require storing large voxel grids or photon maps [S1, unofficial][S2, unofficial].
*   **Strengths:** Explicitly handles high-curvature geometry and shadowing, which diffusion models fail at [S2, unofficial]. Accurately models zero- and single-scatter events, preserving energy better than diffusion approximations [S2, unofficial]. Unbiased [S1, unofficial].
*   **Limitations:** Can be noisy due to the stochastic nature of random walks, especially for deep or highly absorbing media [G]. The fixed depth limit of 256 may truncate paths in very thick media, though Russian Roulette mitigates this [S2, unofficial]. Chromatic effects require wavelength-specific sampling which is inefficient if not handled carefully [S2, unofficial].
*   **When to use:** When ground-truth accuracy is required for subsurface scattering, particularly for objects with high curvature or thin features where diffusion models are inaccurate [S2, unofficial].

## Background
Reference Path Tracing originated as a method to provide a ground-truth reference for subsurface scattering in production renderers [S1, unofficial][S2, unofficial]. It was developed to address the limitations of diffusion-based subsurface scattering models, which assume a semi-infinite flat slab and fail in areas of high curvature like the nose or ears of a human face [S2, unofficial]. By treating the subsurface as a volume and using a random walk, it allows for the integration of standard volumetric path tracing techniques into subsurface scattering simulations [S2, unofficial]. The specific implementation details, such as the `ReferencePathTracer` struct, are found in the Lyr.jl library [S1, unofficial].

## Sources
[S1] Lyr.jl, https://github.com/tobiasosborne/Lyr.jl, fetched 2026-10-26 (unofficial)
[S2] Path Traced Subsurface Scattering Model, https://www.scribd.com/document/965284472/Pixar-Path-Traced-SSS-Paper, fetched 2026-10-29 (unofficial)
