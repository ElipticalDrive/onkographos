# Multiple Scattering

Also called: Multiple Scattering Simulation

## Overview
Multiple scattering simulates light transport where photons undergo more than one scattering event within a participating medium [S1][S2, unofficial]. It solves the problem of rendering dense media, such as thick clouds or smoke, which appear dark and lifeless under single scattering approximations because they fail to capture the internal light bouncing that contributes significantly to their brightness and color [S2, unofficial]. It is distinguished from single scattering by its ability to model the recursive dependency of radiance on itself, though it incurs a multiplicative computational cost for each additional bounce [S1][S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Volume density fields (extinction $\sigma_t$, scattering $\sigma_s$, absorption $\sigma_a$), phase function $p(\omega', \omega)$, and a "Volume Trace Depth" parameter controlling the maximum number of scattering bounces [S2, unofficial].
*   **Outputs:** Final radiance $L$ at the camera pixel, accounting for light that has scattered multiple times before escaping the volume [S1][S2, unofficial].

**Data Structures:**
The method builds on the standard volume rendering data structures defined in the parent article [10687-multiple-scattering-models.md], specifically the radiative transfer equation (RTE) and optical depth calculations [G].

**Steps:**
1.  **Ray Marching:** Cast a ray from the camera into the volume [S2, unofficial].
2.  **Bounce Simulation:** For each scattering event encountered, determine the new direction based on the phase function [S2, unofficial].
3.  **Recursive Integration:** Instead of terminating the path after one scatter (single scattering), continue tracing the ray for up to $N$ bounces, where $N$ is the Volume Trace Depth [S2, unofficial].
4.  **Accumulation:** Accumulate the scattered radiance from each bounce, weighted by the transmittance and scattering albedo $\omega_0 = \sigma_s / \sigma_t$ [S1][S2, unofficial].
5.  **Termination:** Stop when the maximum bounce depth is reached [S2, unofficial].

**Parameters:**
*   **Volume Trace Depth:** Controls the number of scattering bounces simulated. Typical tested values include 1 (single scattering), 5, and 31 [S2, unofficial].
*   **Scattering Strength:** The scattering coefficient must be sufficiently high for the intensity of the scattered light to be visible [S2, unofficial].

**Pitfalls:**
*   **Noise:** Unbiased rendering of multiple scattering can be noisy due to the high variance in path contributions [S1].
*   **Performance:** Every added bounce multiplies the computational work, making dense media rendering significantly slower [S2, unofficial].
*   **Denoising Difficulty:** Volumes are harder to denoise than surfaces because helper data like surface normals does not exist inside a cloud [S2, unofficial].
*   **Degenerate Input:** If scattering strength is too low, multiple scattering effects will be invisible regardless of the trace depth [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_multiple_scattering(ray_origin, ray_direction, volume, max_bounces, rng):
    """
    Simulates multiple scattering for a given ray.
    
    Args:
        ray_origin: 3D vector, camera ray origin.
        ray_direction: 3D vector, camera ray direction.
        volume: Object with methods get_sigma_t(pos), get_sigma_s(pos), get_sigma_a(pos), get_phase(pos), get_emissive(pos).
        max_bounces: The Volume Trace Depth parameter.
        rng: Random number generator returning uniform [0, 1).
        
    Returns:
        L: Accumulated radiance [R, G, B].
    """
    L = [0.0, 0.0, 0.0]
    throughput = 1.0
    current_pos = ray_origin
    current_dir = ray_direction

    for bounce in range(max_bounces):
        # 1. Sample distance to next scattering event using optical depth
        sigma_t = volume.get_sigma_t(current_pos)
        if sigma_t <= 0:
            # Empty space: advance far or break depending on volume bounds
            current_pos += 1000.0 * current_dir
            continue

        # Exponential sampling for free path length: t = -ln(1 - rand) / sigma_t
        t = -log(1.0 - rng()) / sigma_t

        # 2. Check volume boundary
        if t > 1000.0:
            break

        current_pos += t * current_dir

        # 3. Calculate transmittance to this point: T(t) = exp(-sigma_t * t)
        transmittance = exp(-sigma_t * t)

        # 4. Sample scattering direction using phase function p(omega', omega)
        # For isotropic phase function p = 1/(4pi), sample uniformly on sphere
        # omega_s = sample_phase_function(volume.get_phase(current_pos), current_dir, rng)
        # Simplified: assume isotropic scattering for self-containment
        omega_s = current_dir 

        # 5. Update throughput with scattering albedo omega_0 = sigma_s / sigma_t
        sigma_s = volume.get_sigma_s(current_pos)
        omega_0 = sigma_s / sigma_t if sigma_t > 0 else 0.0
        throughput *= transmittance * omega_0

        # 6. Accumulate contribution from this bounce
        emissive = volume.get_emissive(current_pos)
        L[0] += throughput * emissive[0]
        L[1] += throughput * emissive[1]
        L[2] += throughput * emissive[2]

        # 7. Continue ray for next bounce
        current_dir = omega_s

    return L
```
*Note: The pseudocode above is a simplified path-tracing representation. The sources do not provide a specific closed-form formula for the multiple scattering integral solution, but describe the process as following rays through many bounces [S2, unofficial].*

## Subtopics

### Heterogeneous In-scattering with Multiple Scattering
[Link](4934-heterogeneous-in-scattering-with-multiple-scattering.md)

### Most Probable Paths (MPP) Lighting Model
[Link](12244-most-probable-paths-mpp-lighting-model.md)

### Riley et al. Angular Distribution Renderer
[Link](12249-riley-et-al-angular-distribution-renderer.md)

### 3D-Filter / Concrete Directions
[Link](12253-3d-filter-concrete-directions.md)

### Hybrid Hair/Fur Rendering Strategy
[Link](15774-hybrid-hair-fur-rendering-strategy.md)

## Operating characteristics
*   **Time Complexity:** Computational work multiplies with every added bounce [S2, unofficial]. Rendering time increases significantly with higher Volume Trace Depth values [S2, unofficial].
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** In Blender's Cycles, unbiased rendering of multiple scattering is noisy and often disabled by default due to performance costs [S1].
*   **Strengths:** Accurately renders dense media like clouds and smoke, which appear realistic with many scattering bounces [S1][S2, unofficial].
*   **Limitations:** Noisy results; high computational cost; difficult to denoise due to lack of surface normals [S1][S2, unofficial].
*   **When to use:** For dense participating media like clouds, smoke, or thick fog where single scattering appears too dark [S1][S2, unofficial].
*   **When not to use:** For thin media like light haze, where single scattering is sufficient and faster [S2, unofficial]; for materials like skin or milk, where subsurface scattering is more efficient [S1].

## Background
Multiple scattering is a fundamental concept in radiative transfer theory, essential for simulating real-world effects like scattering in clouds and subsurface scattering [S1]. The term "multiple scattering" refers to the effect resulting from more than one scattering bounce [S1]. In computer graphics, it has been a challenge due to its computational expense, leading to the development of various approximations and hybrid methods [S2, unofficial]. The concept is widely used in volume rendering engines, though often limited by default settings to maintain performance [S1].

## Sources
[S1] Volumes — Blender Manual, https://www.blender.org/manual/nb/3.6/render/materials/components/volume.html, fetched 2026-10-26
[S2] What Is Volumetric Rendering? Volume Shaders Explained, from ... - Maxon, https://www.maxon.net/en/article/volumetric-rendering, fetched 2026-10-26 (unofficial)
