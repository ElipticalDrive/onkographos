# Phase-Only Wave Field Reconstruction/Propagation (POS-WRP)
Also called: POS-WRP, Perspective-to-Orthogonal Smooth Wave field Rendering Pipeline (POS-WRP)

## Overview
POS-WRP is a rendering pipeline that encodes mesh-based 3D scenes into complex-valued wave fields with fine-grained textures, photo-realistic shadings, and perspective depth cues [S1]. It solves the problem of maintaining continuous depth cues and realistic illumination when projecting from perspective scene space to the orthogonal space of a Spatial Light Modulator (SLM) [S1]. Unlike standard wave propagation which assumes paraxial orthogonal propagation, POS-WRP explicitly transforms mesh coordinates and textures to correct for projection discrepancies, delivering high-fidelity data with reduced speckle artifacts [S1].

## How it works
**Inputs:**
*   3D scene data: Mesh geometry (vertices, faces) and lighting data captured through a perspective camera [S1].
*   Pre-defined output texture dimensions [S1].

**Outputs:**
*   Complex-valued wave field representation of the scene at the hologram plane [S1].
*   Specifically, the amplitude is derived from baked textures, and the phase is processed for SLM compatibility [S1].

**Data Structures:**
*   **Mesh:** Standard polygonal mesh representation [S1].
*   **Texture:** 2D array storing physical illumination values (amplitude) [S1].
*   **Wave Field:** Complex-valued array $U(x,y)$ representing the field at the SLM plane [S1].

**Parameters:**
*   Processing time for POS-WRP component: 24.96 seconds per hologram [S1].
*   Spatial sampling rate: Assumed sufficient such that visual loss from ignoring diffraction at occlusion edges is negligible [S1].

**Steps:**

1.  **Input Acquisition:** Load the 3D scene comprising meshes and lighting data from a perspective camera view [S1].
2.  **Texture Baking (Amplitude Generation):**
    *   Employ a Physically Based Rendering (PBR) pipeline to bake rendering outcomes onto a pre-defined output texture [S1].
    *   The texture stores physical illumination, which serves as the amplitude $A(x,y)$ of the wave field across mesh facets [S1].
    *   **Occlusion Handling:** During baking, obstructed portions of a facet are rendered black (zero amplitude) [S1]. This adheres to geometric optics principles [S1]. Diffraction at occlusion edges is not explicitly accounted for, relying on facet size and sampling rate to render visual loss negligible [S1].
3.  **Coordinate Transformation:** Transform mesh coordinates and textures from perspective scene space to orthogonal SLM space to maintain consistency with the orthogonal projection required for wave field propagation from the SLM [S1].
4.  **Tilted Plane Propagation & Phase Smoothing:**
    *   Process the transformed meshes and textures through tilted plane propagation [S1].
    *   Incorporate an additional phase smoothing step to generate a relatively smooth target phase [S1]. This mitigates artifacts caused by SLM crosstalk [S1].
5.  **Wave Field Propagation:**
    *   Use spectral resampling and the Angular Spectrum Method (ASM) to propagate each tilted facet of the mesh scene to the hologram plane [S1].
    *   This creates the final complex-valued wave field representation of the entire scene [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def pos_wrp_pipeline(mesh_scene, lighting_data, camera_params, texture_resolution):
    """
    Implements the Phase-Only Wave Field Reconstruction/Propagation (POS-WRP) pipeline.
    
    Args:
        mesh_scene: List of mesh objects with geometry and material properties.
        lighting_data: Light sources and environment map.
        camera_params: Perspective camera intrinsics and extrinsics.
        texture_resolution: (width, height) of the output texture.
        
    Returns:
        complex_wave_field: Complex-valued array representing the wave field at the hologram plane.
    """
    
    # Step 1: Input 3D scene comprises meshes and lighting data captured through a perspective camera.
    # (Data is already in input variables)
    
    # Step 2: Texture Baking
    # Employ physically based rendering (PBR) pipelines to bake rendering outcomes onto a pre-defined output texture.
    # The texture contains physical illumination serving as the amplitude.
    amplitude_texture = bake_pbr_texture(
        meshes=mesh_scene,
        lights=lighting_data,
        camera=camera_params,
        resolution=texture_resolution
    )
    
    # Occlusion handling is naturally integrated during baking.
    # Obstructed portions are rendered black (zero amplitude).
    # Note: Diffraction at occlusion edges is not explicitly accounted for [S1].
    
    # Step 3: Coordinate Transformation
    # Transform mesh coordinates and textures from perspective scene space to orthogonal SLM space.
    # This ensures consistency with the orthogonal projection during wave field propagation.
    transformed_meshes = transform_to_orthogonal_space(
        meshes=mesh_scene,
        camera_params=camera_params
    )
    transformed_texture = transform_texture_coordinates(
        texture=amplitude_texture,
        camera_params=camera_params
    )
    
    # Step 4: Tilted Plane Propagation and Phase Smoothing
    # Process transformed meshes and textures through tilted plane propagation.
    # Incorporate phase smoothing to mitigate SLM crosstalk artifacts.
    # This step generates a relatively smooth target phase.
    # Implementation detail: The specific smoothing algorithm is not detailed in the source,
    # but it is applied to the phase component derived from the tilted propagation.
    smoothed_phase = apply_phase_smoothing(
        meshes=transformed_meshes,
        texture=transformed_texture
    )
    
    # Step 5: Wave Field Propagation using ASM
    # Use spectral resampling and Angular Spectrum Method (ASM) to propagate each tilted facet
    # to the hologram plane.
    complex_wave_field = propagate_with_asm(
        meshes=transformed_meshes,
        amplitude=transformed_texture,
        phase=smoothed_phase,
        method="spectral_resampling_asm"
    )
    
    return complex_wave_field

def bake_pbr_texture(meshes, lights, camera, resolution):
    """
    Simulates PBR baking. Returns a 2D array of amplitude values.
    Obstructed areas are 0.0.
    """
    # [G] Standard rasterization and shading calculation.
    # For each pixel in the output texture:
    #   1. Ray trace from camera through pixel into scene.
    #   2. If intersection with mesh:
    #      a. Calculate shading using PBR model (lights, material).
    #      b. Store result as amplitude.
    #   3. Else (occluded or no intersection):
    #      a. Store 0.0 (black).
    return np.zeros(resolution) # Placeholder for actual PBR engine call

def transform_to_orthogonal_space(meshes, camera_params):
    """
    Transforms mesh vertices from perspective projection to orthogonal space compatible with SLM.
    """
    # [G] Apply inverse perspective projection and re-project to orthogonal coordinates.
    # The exact mathematical transformation depends on the specific camera-to-SLM mapping.
    # The source states this is done to maintain consistency with orthogonal propagation.
    return meshes # Placeholder for actual transformation logic

def apply_phase_smoothing(meshes, texture):
    """
    Applies phase smoothing to mitigate SLM crosstalk.
    """
    # [G] The source mentions an "additional phase smoothing step" but does not provide the algorithm.
    # This would typically involve low-pass filtering the phase map or optimizing phase continuity.
    # Since the specific method is not given, we assume a generic smoothing operation.
    return np.zeros_like(texture) # Placeholder

def propagate_with_asm(meshes, amplitude, phase, method):
    """
    Propagates the wave field from the mesh plane to the hologram plane using ASM.
    """
    # [G] Angular Spectrum Method implementation.
    # 1. Initialize complex field U = amplitude * exp(i * phase).
    # 2. Compute FFT of U.
    # 3. Multiply by transfer function H(fx, fy) = exp(i * k_z * z).
    # 4. Compute IFFT to get propagated field.
    # The source specifies "spectral resampling" is used, implying interpolation in frequency domain.
    return np.zeros_like(amplitude) # Placeholder
```

## Operating characteristics
*   **Time Complexity:** The POS-WRP component takes 24.96 seconds per hologram [S1]. The total time for generating one phase-only hologram (including the subsequent PAGD optimization) is 54.96 seconds [S1].
*   **Memory Complexity:** Not given in the sources.
*   **Strengths:**
    *   Delivers high-fidelity wave field data with continuous depth cues [S1].
    *   Reduces speckle artifacts [S1].
    *   Maintains perspective depth cues and realistic shadings which are typically lost in orthogonal propagation [S1].
    *   Computationally efficient occlusion handling by using geometric optics (PBR) rather than wave-based masking [S1].
*   **Limitations:**
    *   Does not explicitly account for diffraction at occlusion edges, assuming visual loss is negligible due to sampling rate [S1].
    *   Real-time hologram generation remains a goal for future exploration; current performance is not real-time [S1].
*   **When to use:** When high-fidelity, photo-realistic holographic reconstruction from mesh data is required, and continuous depth cues are critical.
*   **When not to use:** When real-time performance is strictly required, or when diffraction effects at occlusion edges are visually significant for the specific resolution and scene scale.

## Background
*   **Origin:** The method was introduced in the paper "Mesh-represented and learning-empowered hologram synthesis for full 3D holographic displays" [S1].
*   **Context:** It is part of a framework that also includes the Phase-Aware Gradient Descent (PAGD) algorithm for encoding the complex wave field into phase-only holograms [S1].
*   **Development:** The method addresses challenges in end-to-end holographic display frameworks where tightly coupled approaches may lack robustness [S1]. It aims to bridge the disparity in wavefront propagation between simulation and physical display systems [S1].

## Sources
[S1] Mesh-represented and learning-empowered hologram synthesis for full 3D holographic displays, https://www.nature.com/articles/s41467-026-74557-0, fetched 2026-10-30
