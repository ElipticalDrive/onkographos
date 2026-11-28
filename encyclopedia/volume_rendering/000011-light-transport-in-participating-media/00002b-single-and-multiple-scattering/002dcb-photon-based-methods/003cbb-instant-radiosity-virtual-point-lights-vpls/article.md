# Instant Radiosity / Virtual Point Lights (VPLs)
Also called: Instant Radiosity, Virtual Point Lights

## Overview
Instant Radiosity, also known as Virtual Point Lights (VPLs), is a method for computing indirect illumination that approximates the global illumination solution by treating stored photons as discrete light sources emitting into the scene [S1]. It differs from density estimation methods (such as [Photon-based Estimation](2521-photon-based-estimation.md)) by replacing the smoothing kernel summation with a visibility-based gathering pass, where each photon acts as a point light [S1]. This approach requires significantly fewer samples than density estimation but introduces singularity artifacts due to potential proximity between VPLs and shading points [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of photons stored from a previous lighting pass, each containing position $\mathbf{p}$, direction $\mathbf{d}$, and power $\Phi$ [S1].
*   **Output:** The final radiance or irradiance at shading points in the scene [S1].

**Data Structures:**
*   **VPL List:** A list of Virtual Point Lights [G]. Each VPL stores:
    *   Position $\mathbf{p}_i$ (3D vector) [G].
    *   Power $\Phi_i$ (scalar, typically spectral or RGB) [G].
    *   Direction $\mathbf{d}_i$ (3D vector, often the incoming photon direction) [G].
    *   *Note:* The number of VPLs is much smaller than the total number of photons because each VPL requires a visibility computation [S1].

**Steps:**
1.  **Lighting Pass:** Shoot photons into the scene using a method similar to photon mapping [S1]. Store photons that hit surfaces [S1].
2.  **VPL Generation:** Treat every stored photon as a virtual point light that emits light into the scene [S1].
3.  **Rendering Pass (Gathering):** For each shading point $\mathbf{x}$ on a surface, gather contributions from these VPLs, requiring visibility computation for each VPL [S1].

**Formulas:**
The irradiance $E(\mathbf{x})$ at a point $\mathbf{x}$ is approximated by the sum of the contributions from all point lights [S2, unofficial]:
$$ E(\mathbf{x}) = \sum_{i} \Phi_i \cdot V_i \cdot \frac{\cos \theta_i}{r_i^2} \cdot f_r(\mathbf{x}, \mathbf{d}_i, \mathbf{v}_i) $$
Where:
*   $\Phi_i$: Power of the $i$-th VPL.
*   $V_i$: Visibility (1 if visible, 0 if occluded).
*   $r_i$: Distance between $\mathbf{x}$ and $\mathbf{p}_i$.
*   $\theta_i$: Angle between the surface normal at $\mathbf{x}$ and the vector $\mathbf{v}_i$.
*   $f_r$: The Bidirectional Reflectance Distribution Function (BRDF).

**Parameters:**
*   **Number of VPLs:** Significantly fewer than the total number of photons [S1].

**Edge Cases and Pitfalls:**
*   **Singularity Artifacts:** Tiny distances between VPLs and shading points can cause spike artifacts in the rendered results [S1].
*   **Mitigation:** These artifacts are removed by clamping distances or blurring the contributions, which introduces bias or missing energy [S1].
*   **Bias Compensation:** Novák et al. proposed a method to approximately compensate for the bias introduced by clamping or blurring [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def compute_irradiance(shading_point, vpl_list):
    """
    Computes irradiance at a shading point by gathering contributions from VPLs.
    shading_point: object with .position, .normal, .material
    vpl_list: list of objects with .position, .power, .direction
    """
    total_irradiance = 0.0
    normal = shading_point.normal
    position = shading_point.position

    for vpl in vpl_list:
        # Vector from shading point to VPL
        v = [vpl.position[j] - position[j] for j in range(3)]
        distance = math.sqrt(sum(c*c for c in v))
        
        # Avoid division by zero and singularity spikes
        if distance < 1e-4:
            distance = 1e-4
            
        # Visibility check
        visible = cast_shadow_ray(position, vpl.position)
        
        if visible:
            # Cosine term: angle between surface normal and incoming light direction
            light_dir = [v[j] / distance for j in range(3)]
            cos_theta = max(0.0, sum(normal[j] * light_dir[j] for j in range(3)))
            
            # Geometric attenuation (inverse square law)
            attenuation = 1.0 / (distance * distance)
            
            # BRDF evaluation
            brdf = evaluate_brdf(shading_point.material, normal, light_dir, light_dir)
            
            # Contribution
            contribution = vpl.power * attenuation * cos_theta * brdf
            total_irradiance += contribution
            
    return total_irradiance

def cast_shadow_ray(origin, target):
    """Standard visibility test: returns True if no geometry blocks the path."""
    # Implementation iterates over scene primitives to find nearest intersection.
    # Returns True if intersection distance > |target - origin|.
    return True

def evaluate_brdf(material, wo, wi, wv):
    """Standard BRDF evaluation for the given material model."""
    # Implementation evaluates the material's reflectance function.
    # Returns scalar reflectance value.
    return 1.0
```

## Subtopics

### Multi-dimensional Lightcuts
Extends VPLs to participating media and other effects like motion blur using efficient hierarchy pruning [S1]. It sets itself apart by pruning the VPL list dynamically during rendering, making it suitable for complex scenes with many light sources [S1]. Choose it when rendering participating media or motion blur where full VPL visibility checks are too expensive [S1]. See [Multi-dimensional Lightcuts](link).

### Lightcuts with Diffusion Dipoles
Combines lightcuts with diffusion dipoles to approximate subsurface scattering [S1]. It differs from standard VPLs by modeling light transport within translucent materials using dipole approximations rather than surface-only gathering [S1]. Choose it when rendering subsurface scattering effects in participating media [S1]. See [Lightcuts with Diffusion Dipoles](link).

## Operating characteristics
*   **Complexity:** Requires visibility computation for each VPL, making it more expensive per sample than density estimation but requiring fewer samples overall [S1].
*   **Performance:** Acceleration techniques using hierarchical pruning have been developed to speed up visibility computation [S1].
*   **Strengths:** Widely used for indirect illumination on surfaces; requires significantly fewer samples than density estimation methods [S1].
*   **Limitations:** Suffers from singularity artifacts (spikes) due to small distances between VPLs and shading points; mitigation techniques (clamping/blurring) introduce bias or missing energy [S1].

## Background
Instant Radiosity was proposed by Keller [S1]. It has been used in a large number of surface rendering algorithms [S1]. Later, VPLs were extended to participating media [S1].

## Sources
[S1] A survey on rendering homogeneous participating media, https://doi.org/10.1007/s41095-021-0249-1, fetched 2026-10-26
[S2] Massive Data Management for Distributed Volume Visualization, http://hdl.handle.net/1951/47671, fetched 2026-10-26 (unofficial)
