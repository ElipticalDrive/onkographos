# Null Interactions Shift

Also called: Prefix Shift

## Overview
Null Interactions Shift is a bijective transformation method used in volumetric multi-view rendering to map medium interaction depths sampled on a base viewing segment to valid, corresponding depths on a target viewing segment [S1]. It solves the problem of reusing Monte Carlo samples across different camera viewpoints by ensuring that shifted interactions remain within the medium, thereby maintaining a well-defined probability density function (pdf) [S1]. Unlike simpler random number replay techniques which may place interactions outside the medium, this method uses the majorant optical depth to guarantee validity [S1]. It builds on [Delta tracking](985-delta-tracking.md) by providing the specific mapping mechanism required when shifting prefixes between cameras [S1].

## How it works
The method constructs a shifted prefix chain connecting a target camera lens to a pivot point (a surface interaction or previous medium event) by reusing the random numbers generated for the base prefix [S1]. The core challenge is mapping the interaction depths $t$ from the base segment to $t'$ on the target segment such that the number of random events remains identical, preserving the probability space [S1].

### Inputs and Outputs
- **Inputs**: Base prefix parameters (lens position, film coordinates, shutter time, sampled depths $t_i$), target camera parameters (lens position, film coordinates, shutter time), pivot position and normal, and majorant extinction coefficient functions $\bar{\mu}(x)$ for both segments [S1].
- **Outputs**: Shifted prefix parameters, shifted interaction depths $t'_i$, and the Jacobian determinant of the transformation [S1].

### Data Structures
The method operates on scalar values and 3D vectors [G]. No complex data structures are required beyond standard ray tracing primitives [G].
- **Depth**: Floating-point distance along the ray.
- **Vector**: 3D position or direction.
- **Majorant Extinction**: Piecewise constant or linear function $\bar{\mu}(x)$ defined over medium sections [S1].

### Steps
1. **Shutter Time Shift**: Reuse the random number used to sample the shutter time $t_{time}$ on the base camera [S1]. Set the target shutter time $t'_{time} = t_{time}$ [S1]. The Jacobian for this shift is $J_{time} = 1$ [S1]. Object motion is ignored to avoid the cost of re-evaluating occlusions and shading on moving geometry [S1].
2. **Lens Shift**: Reuse the random numbers used to sample the lens position $x_{lens}$ and film coordinates $x_{film}$ on the base camera [S1]. Map these to the target camera's lens $x'_{lens}$ and film $x'_{film}$ [S1]. Calculate the geometric Jacobian:
   $$ J_{lens} = \frac{g(x'_{lens}, x'_{pivot})}{g(x'_{lens}, x'_{film})} $$
   where $g(a, b) = D(a, b) \|b - a\|^2$ and $D(a, b) = |\tilde{\omega}_{ab} \cdot \tilde{n}_b|$ if $b$ is on a surface, else $D(a, b) = 1$ [S1].
3. **Check Pivot Validity**: Verify that the pivot is not occluded from the target lens [S1]. Verify that the pivot falls within the target camera's view frustum and exposure time [S1]. If the pivot is invalid, the shift fails [S1].
4. **Null Interactions Shift (Depth Mapping)**: Trace a ray from the target lens to the pivot to collect the target medium sections [S1]. For each medium interaction depth $t_i$ on the base segment:
   - Compute the normalized majorant optical depth on the base segment:
     $$ \bar{\tau}_i = \frac{\bar{\tau}_{base}(t_i)}{\bar{\tau}_{base}(t_{pivot})} $$
     where $\bar{\tau}_{base}(t) = \int_0^t \bar{\mu}_{base}(x) dx$ [S1].
   - Map this normalized value to the target segment:
     $$ \bar{\tau}_{target}(t'_i) = \bar{\tau}_i \cdot \bar{\tau}_{target}(t_{pivot}) $$
   - Solve for $t'_i$ by inverting the majorant optical depth integral on the target segment [S1]. Since $\bar{\mu}$ is piecewise linear or constant, this inversion is closed-form [S1].
   - If neither prefix crosses any medium section, the shift is complete with no medium interactions (only a surface interaction at the pivot) [S1].
5. **Jacobian Calculation**: The total Jacobian is the product of the individual Jacobians [S1]. The depth Jacobian $J_{depth}$ accounts for the change in density due to the transformation of interaction depths [S1]. It is derived from the derivative of the mapping function [S1].
6. **Validation**: If only one of the base or shifted prefixes crosses medium sections, the shift is invalid because the number of random events differs between the two chains [S1]. This occurs on volume silhouettes and leads to undersampling [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
from typing import List, Tuple, Optional

class Ray:
    def __init__(self, origin: Tuple[float, float, float], direction: Tuple[float, float, float]):
        self.origin = origin
        self.direction = direction

class Prefix:
    def __init__(self, lens_pos: Tuple[float, float, float], film_pos: Tuple[float, float, float],
                 shutter_time: float, depths: List[float], pivot_dist: float):
        self.lens_pos = lens_pos
        self.film_pos = film_pos
        self.shutter_time = shutter_time
        self.depths = depths
        self.pivot_dist = pivot_dist

def norm(v: Tuple[float, float, float]) -> float:
    return math.sqrt(v[0]**2 + v[1]**2 + v[2]**2)

def norm_sq(v: Tuple[float, float, float]) -> float:
    return v[0]**2 + v[1]**2 + v[2]**2

def normalize(v: Tuple[float, float, float]) -> Tuple[float, float, float]:
    n = norm(v)
    return (v[0]/n, v[1]/n, v[2]/n)

def is_surface(point: Tuple[float, float, float]) -> bool:
    return point is not None and len(point) == 4  # (x, y, z, normal)

def is_occluded(origin: Tuple[float, float, float], target: Tuple[float, float, float]) -> bool:
    return False  # Standard shadow ray check [G]

def in_frustum(camera: Prefix, point: Tuple[float, float, float]) -> bool:
    return True  # Standard bounding volume check [G]

def trace_medium_sections(ray: Ray, max_t: float) -> List[Tuple[float, float]]:
    return [(0.0, max_t)]  # Simplified medium section collection [G]

def integrate_mu_bar(mu_bar_func, t_start: float, t_end: float) -> float:
    return mu_bar_func(0) * (t_end - t_start)  # Piecewise constant assumption [G]

def invert_mu_bar(mu_bar_func, tau: float) -> float:
    return tau / mu_bar_func(0)  # Piecewise constant inversion [G]

def null_interactions_shift(base_prefix: Prefix, target_camera: Prefix, pivot: Tuple[float, float, float, float],
                            base_mu_bar, target_mu_bar) -> Optional[Prefix]:
    # 1. Shutter Time Shift
    shifted_shutter_time = base_prefix.shutter_time
    j_time = 1.0

    # 2. Lens Shift
    # Reuse random numbers for lens and film coordinates
    # Map base lens/film to target lens/film
    # Calculate geometric term g(a, b)
    def g(a, b):
        omega_ab = normalize((b[0]-a[0], b[1]-a[1], b[2]-a[2]))
        if is_surface(pivot):
            D = abs(omega_ab[0]*pivot[3][0] + omega_ab[1]*pivot[3][1] + omega_ab[2]*pivot[3][2])
        else:
            D = 1.0
        dist_sq = norm_sq((b[0]-a[0], b[1]-a[1], b[2]-a[2]))
        return D * dist_sq

    j_lens = g(target_camera.lens_pos, pivot) / g(target_camera.lens_pos, target_camera.film_pos)

    # 3. Check Pivot Validity
    if is_occluded(target_camera.lens_pos, pivot[:3]):
        return None
    if not in_frustum(target_camera, pivot[:3]):
        return None

    # 4. Null Interactions Shift
    target_ray = Ray(target_camera.lens_pos, normalize((pivot[0]-target_camera.lens_pos[0],
                                                        pivot[1]-target_camera.lens_pos[1],
                                                        pivot[2]-target_camera.lens_pos[2])))
    target_sections = trace_medium_sections(target_ray, norm((pivot[0]-target_camera.lens_pos[0],
                                                              pivot[1]-target_camera.lens_pos[1],
                                                              pivot[2]-target_camera.lens_pos[2])))

    tau_base_pivot = integrate_mu_bar(base_mu_bar, 0.0, base_prefix.pivot_dist)
    tau_target_pivot = integrate_mu_bar(target_mu_bar, 0.0, norm((pivot[0]-target_camera.lens_pos[0],
                                                                  pivot[1]-target_camera.lens_pos[1],
                                                                  pivot[2]-target_camera.lens_pos[2])))

    if tau_base_pivot == 0.0 or tau_target_pivot == 0.0:
        return None

    shifted_depths = []
    j_depth = 1.0

    for t_i in base_prefix.depths:
        tau_base_t = integrate_mu_bar(base_mu_bar, 0.0, t_i)
        normalized_tau = tau_base_t / tau_base_pivot

        tau_target_t_prime = normalized_tau * tau_target_pivot
        t_prime = invert_mu_bar(target_mu_bar, tau_target_t_prime)

        # Check if t_prime is inside a medium section
        inside = False
        for start, end in target_sections:
            if start <= t_prime <= end:
                inside = True
                break
        if not inside:
            return None

        shifted_depths.append(t_prime)

        # Depth Jacobian: derivative of mapping accounts for density change [S1]
        mu_base = base_mu_bar(t_i)
        mu_target = target_mu_bar(t_prime)
        j_depth_i = (mu_base / mu_target) * (tau_target_pivot / tau_base_pivot)
        j_depth *= j_depth_i

    # 5. Construct shifted prefix
    shifted_prefix = Prefix(
        lens_pos=target_camera.lens_pos,
        film_pos=target_camera.film_pos,
        shutter_time=shifted_shutter_time,
        depths=shifted_depths,
        pivot_dist=norm((pivot[0]-target_camera.lens_pos[0],
                         pivot[1]-target_camera.lens_pos[1],
                         pivot[2]-target_camera.lens_pos[2]))
    )

    return shifted_prefix
```

### Parameters
- **Index $i$**: Denotes the pixel used to generate the base prefix [S1].
- **Index $j$**: Denotes another pixel and its shifted prefix [S1].

### Edge Cases and Pitfalls
- **Volume Silhouettes**: If only one of the base or shifted prefixes crosses medium sections, a valid shift cannot be constructed because the number of random events differs [S1]. This leads to undersampling in these areas [S1].
- **Outside Medium**: Simpler shifts like random number replay may generate interactions outside the medium [S1]. The majorant optical depth is used because the true optical depth often lacks a closed-form inverse, whereas the majorant is piecewise linear and always invertible [S1].
- **Occlusion**: The pivot must not be occluded from the target lens [S1]. If it is, the shift fails [S1].

## Subtopics

### [Majorant optical depth shift](2043-majorant-optical-depth-shift.md)
The majorant optical depth shift maps a medium interaction depth $t$ sampled on a base viewing segment to a corresponding depth $t'$ on a target viewing segment. This enables path reuse in multi-view rendering by accounting for differences in volume geometry and extinction coefficients between views. It builds on [Null Interactions Shift](1836-null-interactions-shift.md) by providing the specific mapping function required when the medium's majorant extinction coefficients differ between the base and target segments.

## Operating characteristics
- **Time Complexity**: The shift operation is $O(N)$ where $N$ is the number of medium interactions in the prefix [S1]. The integration and inversion of the majorant optical depth are efficient due to the piecewise linear nature of the majorant [S1].
- **Memory Complexity**: $O(N)$ to store the shifted depths and prefix parameters [G].
- **Strengths**: Ensures shifted interactions are always within the medium, maintaining a well-defined pdf [S1]. Uses closed-form inversion for efficiency [S1].
- **Limitations**: Fails on volume silhouettes where the number of medium sections differs between views, leading to undersampling [S1]. Ignores object motion, which may limit applicability in dynamic scenes [S1].
- **When to Use**: Multi-view rendering of static volumes where path reuse is beneficial [S1].
- **When Not to Use**: Dynamic scenes with significant object motion, or scenes with complex volume silhouettes where undersampling is critical [S1].

## Background
Null Interactions Shift was introduced in the context of volumetric multi-view rendering to enable efficient path reuse across multiple camera viewpoints [S1]. It addresses the challenge of maintaining valid probability densities when shifting Monte Carlo samples between different viewing segments [S1]. The method relies on the properties of delta tracking and the majorant extinction coefficient to ensure robustness [S1].

## Sources
[S1] Volumetric Multi‐View Rendering, https://doi.org/10.1111/cgf.14481, fetched 2026-10-26
