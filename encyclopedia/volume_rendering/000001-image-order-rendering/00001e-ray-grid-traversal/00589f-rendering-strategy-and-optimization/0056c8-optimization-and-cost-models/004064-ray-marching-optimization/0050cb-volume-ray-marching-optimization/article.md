# Volume Ray Marching Optimization

Also called: Volumetric Cloud Optimization, Cloud Shader Optimization

## Overview
Volume Ray Marching Optimization refers to a set of shader-level and rendering pipeline techniques designed to reduce the computational cost of evaluating participating media, specifically in the context of volumetric clouds. It addresses the bottleneck of expensive material evaluations and multiple light scattering approximations during ray traversal. Unlike general traversal optimizations that skip empty space, this method optimizes the *cost* of the density and lighting calculations performed at each sample point. It distinguishes itself by focusing on material simplification (grayscale inputs), scattering approximation limits, and alternative shadowing strategies (cascaded shadow maps vs. secondary ray marching) rather than spatial acceleration structures.

## How it works
The method operates within the fragment shader of a volumetric cloud material, typically in a real-time engine context. It modifies the evaluation loop of the ray marcher to reduce arithmetic intensity and memory bandwidth.

**Inputs and Outputs:**
- **Input:** Ray origin, ray direction, step size, material parameters (density, color, scattering coefficients), camera position, light positions.
- **Output:** Final color and opacity for the pixel.

**Data Structures:**
- **Material Parameters:** Typically a vector or struct containing density, color, and scattering coefficients.
- **Conservative Density:** A scalar or vector component used for early-out culling.

**Numbered Steps:**

1. **Conservative Density Culling:**
   Before evaluating the full material model, check the conservative density value. If the conservative density is less than or equal to 0, skip the expensive material evaluation and proceed to the next sample directly [S1]. This acts as a fast-path filter for low-density regions.

2. **Grayscale Material Evaluation:**
   If the "Gray Scale Material" optimization is enabled, restrict the material's input parameter evaluation to only the Red (R) channel. The resulting material response (density/opacity) will be grayscale, but the lighting calculation remains colored [S1]. This reduces the number of texture lookups or procedural noise evaluations required per sample.

3. **Multi-Scattering Approximation:**
   Limit the number of octaves used for the Multi-Scattering Approximation. The default is single scattering (0 octaves). Increasing this to a maximum of two approximation octaves simulates multiple scattering effects but increases computational cost [S1]. The shader should cap this value to balance quality and performance.

4. **Shadowing Strategy Selection:**
   Determine the shadowing method based on the "Ray March Volume Shadow" setting:
   - **If Enabled:** Perform secondary ray marching through the cloud volume to compute shadows. This is accurate but expensive.
   - **If Disabled:** Use Cascaded Shadow Maps (CSM). This provides a performance boost and infinite shadow length but results in less accurate, grayscale shadows [S1].

5. **Ground Contribution (Optional):**
   If frame budget allows, enable Ground Contribution. This adds a sampling pass to calculate the shadowed lighting contribution from the ground onto the bottom layer of the clouds [S1]. This is an additive cost for visual fidelity.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def evaluate_volumetric_cloud_sample(ray_sample, material_params, optimization_settings):
    """
    Evaluates a single sample in a volumetric cloud with optimizations.
    
    Args:
        ray_sample: Current position and accumulated transmittance along the ray.
        material_params: Struct containing density, color, scattering coefficients.
        optimization_settings: Dict with keys:
            - use_grayscale_material: bool
            - max_scattering_octaves: int (0-2)
            - use_csm_shadows: bool (True if Ray March Volume Shadow is unchecked)
            - enable_ground_contribution: bool
            - conservative_density_threshold: float (default 0.0)
            
    Returns:
        color: vec3
        opacity: float
    """
    
    # Step 1: Conservative Density Culling
    # Check if the conservative density (X component of Vector3) is <= 0
    # If so, skip expensive material evaluation
    conservative_density = material_params.conservative_density.x
    if conservative_density <= optimization_settings['conservative_density_threshold']:
        # Skip material evaluation, return zero contribution or pass-through
        return vec3(0.0), 0.0

    # Step 2: Grayscale Material Evaluation
    if optimization_settings['use_grayscale_material']:
        # Only consider the R channel of material input parameters
        # This simplifies density/color calculation
        density = material_params.density_r
        color = vec3(material_params.color_r)
    else:
        # Full material evaluation
        density = material_params.density
        color = material_params.color

    # Step 3: Multi-Scattering Approximation
    # Limit octaves to save performance. Default 0 (single scattering), max 2.
    num_octaves = min(material_params.scattering_octaves, optimization_settings['max_scattering_octaves'])
    scattering_coefficient = compute_multi_scattering_approximation(density, num_octaves)

    # Step 4: Shadowing
    if optimization_settings['use_csm_shadows']:
        # Use Cascaded Shadow Maps
        # Shadows are grayscale and less accurate
        shadow_factor = get_csm_shadow_factor(ray_sample.position)
        shadow_color = vec3(shadow_factor) # Grayscale shadow
    else:
        # Secondary Ray Marching
        # Accurate but expensive
        shadow_factor = ray_march_volume_shadow(ray_sample.position, material_params)
        shadow_color = shadow_factor # Colored shadow

    # Step 5: Ground Contribution (Optional)
    ground_light = vec3(0.0)
    if optimization_settings['enable_ground_contribution']:
        # Sample shadowed lighting from ground to bottom layer of clouds
        ground_light = sample_ground_contribution(ray_sample.position)

    # Combine contributions
    # Note: Exact blending formula depends on the specific renderer's integration
    # This is a simplified representation
    final_color = color * scattering_coefficient * shadow_color + ground_light
    
    return final_color, density
```

**Parameters:**
- **Conservative Density Threshold:** Default 0.0. If conservative density <= threshold, skip evaluation [S1].
- **Max Scattering Octaves:** Default 0. Maximum 2 [S1].
- **Use Grayscale Material:** Boolean. If true, only R channel is used [S1].
- **Use CSM Shadows:** Boolean. If true, uses cascaded shadow maps; if false, uses secondary ray marching [S1].
- **Enable Ground Contribution:** Boolean. Adds cost for ground lighting [S1].

**Edge Cases and Pitfalls:**
- **Precision:** When using grayscale shadows with CSM, color accuracy is lost in shadowed regions [S1].
- **Boundary:** Conservative density culling must be carefully tuned to avoid artifacts in low-density edges.
- **Degenerate Input:** If material parameters are zero, the optimization should handle division by zero or invalid scattering coefficients gracefully [G].

## Subtopics

### [Octree-based Address Translation](15384-octree-based-address-translation.md)
Octree-based address translation is a method for managing multi-resolution volume data during ray-guided rendering by tightly coupling the spatial subdivision of an octree with the resolution levels of the volume data. It solves the problem of efficiently accessing high-resolution bricks while minimizing memory bandwidth by ensuring that each octree node corresponds to exactly one brick in the data set. Unlike page-table hierarchies which decouple spatial and resolution indices, this approach simplifies address calculation but requires keeping the entire path from root to leaf in cache, leading to potential memory overhead.

## Operating characteristics
- **Time Complexity:** The optimization reduces the constant factor of the per-sample cost. Conservative density culling provides an O(1) early-out. Grayscale material evaluation reduces texture fetches. Limiting scattering octaves reduces the complexity of the scattering function from O(N) to O(min(N, 2)). CSM shadows replace O(M) ray marches (where M is the number of shadow samples) with a single texture lookup, significantly reducing time complexity for shadowing.
- **Memory Complexity:** No additional memory structures are introduced by the shader optimizations themselves. However, CSM requires storing shadow map textures, which increases memory usage [S1].
- **Measured Performance:** Enabling CSM shadows provides a "performance boost" [S1]. Limiting scattering octaves saves "some performance" [S1]. Enabling ground contribution "adds some costs" [S1]. Exact FPS gains are not given in the sources.
- **Strengths:** Significant performance gains in complex cloud scenes. Simplifies material evaluation. Infinite shadow length with CSM.
- **Limitations:** Grayscale shadows are less accurate. Ground contribution adds cost. Conservative density culling may cause artifacts if not tuned.
- **When to Use:** When frame budget is tight and visual fidelity of shadows/scattering can be compromised. When clouds are dense and material evaluation is the bottleneck.
- **When Not to Use:** When high-fidelity color shadows are required. When ground lighting is critical for realism.

## Background
This method originates from the implementation of the Volumetric Cloud Component in Unreal Engine 5.8, as documented by Epic Games. It is part of the engine's toolkit for optimizing real-time volumetric rendering. The techniques are specific to the cloud material system and are not general-purpose volume rendering algorithms.

## Sources
[S1] Volumetric Cloud Component in Unreal Engine  | Unreal Engine 5.8 Documentation | Epic Developer Community, https://dev.epicgames.com/documentation/en-us/unreal-engine/volumetric-cloud-component-in-unreal-engine, fetched 2026-10-30
