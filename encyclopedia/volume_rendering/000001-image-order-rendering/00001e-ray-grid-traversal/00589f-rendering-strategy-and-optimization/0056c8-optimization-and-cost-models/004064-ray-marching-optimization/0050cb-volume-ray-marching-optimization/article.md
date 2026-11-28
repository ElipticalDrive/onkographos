# Volume Ray Marching Optimization


## Overview
Volume Ray Marching Optimization refers to a set of shader-level techniques in the Unreal Engine 5.8 Volumetric Cloud Component designed to reduce the computational cost of evaluating participating media [S1]. It distinguishes itself by optimizing per-sample material evaluation (grayscale inputs), limiting scattering approximation octaves, and substituting secondary ray marching with cascaded shadow maps, rather than relying on spatial acceleration structures [S1].

## How it works
**Inputs and Outputs:**
- **Input:** Ray parameters (origin, direction, step size), material parameters (density, color, scattering coefficients, conservative density vector), optimization flags, camera and light positions [G].
- **Output:** Accumulated color and opacity per sample [G].

**Data Structures:**
- **Material Parameters:** Struct containing density, color, and scattering coefficients [G].
- **Conservative Density:** X component of a `Vector3` used for early-out culling [S1].

**Numbered Steps:**

1. **Conservative Density Culling:** Check the X component of the conservative density vector. If the value is greater than 0, evaluate the material; otherwise, skip the expensive material evaluation and proceed directly to the next sample [S1].
2. **Grayscale Material Evaluation:** If the "Gray Scale Material" optimization is enabled, restrict the material's input parameter evaluation to only the Red (R) channel. The resulting material response will be grayscale, but lighting calculations remain colored [S1].
3. **Multi-Scattering Approximation:** Limit the number of octaves for the Multi-Scattering Approximation to save performance. The default is single scattering (0 octaves), with a maximum of two approximation octaves allowed [S1].
4. **Shadowing Strategy Selection:** Toggle between secondary ray marching and cascaded shadow maps using the "Ray March Volume Shadow" checkbox. Unchecking the box enables cascaded shadow maps, providing a performance boost and infinite shadow length at the cost of accuracy and color (grayscale shadows) [S1].
5. **Ground Contribution (Optional):** If the frame budget allows, enable Ground Contribution to sample shadowed lighting from the ground onto the bottom layer of the clouds, which adds tracing cost [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def evaluate_volumetric_cloud_sample(ray_sample, material_params, optimization_settings):
    """
    Evaluates a single sample in a volumetric cloud with optimizations.
    Args:
        ray_sample: Current position and accumulated transmittance along the ray.
        material_params: Struct containing density, color, scattering coefficients, and conservative_density (Vector3).
        optimization_settings: Dict with keys:
            - use_grayscale_material: bool
            - max_scattering_octaves: int (0-2)
            - use_csm_shadows: bool (True if Ray March Volume Shadow is unchecked)
            - enable_ground_contribution: bool
    Returns:
        color: vec3
        opacity: float
    """
    # Step 1: Conservative Density Culling
    # Evaluate material only if conservative density X component > 0 [S1]
    if material_params.conservative_density.x <= 0.0:
        return vec3(0.0), 0.0  # Skip evaluation, proceed to next sample

    # Step 2: Grayscale Material Evaluation
    if optimization_settings['use_grayscale_material']:
        # Only consider the R channel of material input parameters [S1]
        density = material_params.density_r
        color = vec3(material_params.color_r)
    else:
        density = material_params.density
        color = material_params.color

    # Step 3: Multi-Scattering Approximation
    # Default is single scattering (0 octaves), max two approximation octaves [S1]
    num_octaves = min(material_params.scattering_octaves, optimization_settings['max_scattering_octaves'])
    # Standard scattering approximation calculation [G]
    scattering_coefficient = density * (1.0 - exp(-num_octaves))

    # Step 4: Shadowing Strategy Selection
    if optimization_settings['use_csm_shadows']:
        # Uncheck Ray March Volume Shadow to enable Cascaded Shadow Maps [S1]
        # Provides performance boost, infinite length, less accurate, grayscale [S1]
        # Lookup shadow factor from cascaded shadow map textures [G]
        shadow_factor = texture_lookup_csm(ray_sample.position)
        shadow_color = vec3(shadow_factor)
    else:
        # Secondary ray marching through cloud volume [S1]
        # Perform secondary ray march through volume to compute shadow density [G]
        shadow_factor = ray_march_shadow_volume(ray_sample.position, material_params)
        shadow_color = shadow_factor

    # Step 5: Ground Contribution (Optional)
    ground_light = vec3(0.0)
    if optimization_settings['enable_ground_contribution']:
        # Sample shadowed lighting from ground to bottom layer of clouds [S1]
        # Sample lighting contribution from ground plane [G]
        ground_light = sample_ground_lighting(ray_sample.position)

    # Combine contributions
    final_color = color * scattering_coefficient * shadow_color + ground_light
    return final_color, density
```

**Parameters:**
- **Conservative Density Threshold:** 0.0. Material evaluated if value > 0, otherwise skips to next sample [S1].
- **Max Scattering Octaves:** Default 0. Maximum 2 [S1].
- **Use Grayscale Material:** Boolean. If true, restricts material input evaluation to the Red (R) channel [S1].
- **Use CSM Shadows:** Boolean. True enables cascaded shadow maps (uncheck Ray March Volume Shadow); False enables secondary ray marching [S1].
- **Enable Ground Contribution:** Boolean. Adds tracing cost for ground lighting [S1].

**Edge Cases and Pitfalls:**
- **Precision:** Grayscale shadows with CSM lose color accuracy in shadowed regions [S1].
- **Boundary:** Conservative density culling threshold must be tuned to avoid artifacts in low-density edges [G].
- **Degenerate Input:** Zero material parameters should be handled to prevent division by zero in scattering calculations [G].

## Subtopics

### [Octree-based Address Translation](15384-octree-based-address-translation.md)
Octree-based address translation manages multi-resolution volume data during ray-guided rendering by intrinsically coupling the down-sampling ratio between resolution levels with the spatial subdivision of an octree structure. Each octree node corresponds to exactly one brick in the data set. Unlike page-table hierarchies, this approach requires keeping the entire subtree from the root node to the target node in memory to reach a high-resolution brick, which can lead to sub-optimal GPU memory utilization due to the tight coupling of the multi-resolution hierarchy and spatial subdivision.

## Operating characteristics
- **Time Complexity:** The optimization reduces the constant factor of the per-sample cost. Conservative density culling provides an O(1) early-out. Grayscale material evaluation reduces texture fetches. Limiting scattering octaves caps complexity to O(1) (max 2). CSM replaces secondary ray marching with a texture lookup [S1].
- **Memory Complexity:** No additional memory structures are introduced by the shader optimizations themselves. CSM requires shadow map textures, but memory impact is not quantified in the sources [S1].
- **Measured Performance:** Enabling cascaded shadow maps (by unchecking the Ray March Volume Shadow box) provides a performance boost [S1]. Limiting scattering octaves saves performance [S1]. Enabling ground contribution adds cost [S1]. Exact FPS gains are not given in the sources.
- **Strengths:** Significant performance gains in complex cloud scenes. Simplifies material evaluation. Infinite shadow length with CSM.
- **Limitations:** Grayscale shadows are less accurate. Ground contribution adds cost. Conservative density culling may cause artifacts if not tuned.
- **When to Use:** When frame budget is tight and visual fidelity of shadows/scattering can be compromised. When clouds are dense and material evaluation is the bottleneck.
- **When Not to Use:** When high-fidelity color shadows are required. When ground lighting is critical for realism.

## Background
This method originates from the implementation of the Volumetric Cloud Component in Unreal Engine 5.8, as documented by Epic Games [S1]. It is part of the engine's toolkit for optimizing real-time volumetric rendering, specific to the cloud material system [S1].

## Sources
[S1] Volumetric Cloud Component in Unreal Engine  | Unreal Engine 5.8 Documentation | Epic Developer Community, https://dev.epicgames.com/documentation/en-us/unreal-engine/volumetric-cloud-component-in-unreal-engine, fetched 2026-10-30
