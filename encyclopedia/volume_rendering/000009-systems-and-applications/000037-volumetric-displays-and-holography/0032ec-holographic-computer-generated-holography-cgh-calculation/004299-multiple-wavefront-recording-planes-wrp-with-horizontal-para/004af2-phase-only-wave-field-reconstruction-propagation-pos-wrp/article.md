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
*   **Texture:** Output texture storing physical illumination values (amplitude) [S1].
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
import math

def pos_wrp_pipeline(mesh_vertices, mesh_faces, lighting_data, camera_intrinsics, texture_resolution):
    """
    Implements the Phase-Only Wave Field Reconstruction/Propagation (POS-WRP) pipeline.
    
    Args:
        mesh_vertices: List of 3D vertex coordinates.
        mesh_faces: List of face indices referencing vertices.
        lighting_data: Light sources and environment map.
        camera_intrinsics: Perspective camera parameters (focal length, principal point).
        texture_resolution: (width, height) of the output texture.
        
    Returns:
        complex_wave_field: Complex-valued array representing the wave field at the hologram plane.
    """
    
    # Step 1: Input 3D scene comprises meshes and lighting data captured through a perspective camera [S1].
    # (Data is already in input variables)
    
    # Step 2: Texture Baking
    # Employ physically based rendering (PBR) pipelines to bake rendering outcomes onto a pre-defined output texture [S1].
    # The texture contains physical illumination serving as the amplitude across mesh facets [S1].
    amplitude_texture = bake_pbr_texture(mesh_vertices, mesh_faces, lighting_data, camera_intrinsics, texture_resolution)
    
    # Occlusion handling is naturally integrated during baking.
    # Obstructed portions are rendered black (zero amplitude) [S1].
    # Note: Diffraction at occlusion edges is not explicitly accounted for [S1].
    
    # Step 3: Coordinate Transformation
    # Transform mesh coordinates and textures from perspective scene space to orthogonal SLM space [S1].
    # This ensures consistency with the orthogonal projection during wave field propagation.
    transformed_vertices = perspective_to_orthogonal_transform(mesh_vertices, camera_intrinsics)
    transformed_texture = resample_texture_to_orthogonal(amplitude_texture, camera_intrinsics, texture_resolution)
    
    # Step 4: Tilted Plane Propagation and Phase Smoothing
    # Process transformed meshes and textures through tilted plane propagation [S1].
    # Incorporate phase smoothing to mitigate SLM crosstalk artifacts [S1].
    # This step generates a relatively smooth target phase.
    phase_map = compute_phase_from_tilted_facets(transformed_vertices, transformed_texture)
    smoothed_phase = apply_phase_smoothing(phase_map)
    
    # Step 5: Wave Field Propagation using ASM
    # Use spectral resampling and Angular Spectrum Method (ASM) to propagate each tilted facet
    # to the hologram plane [S1].
    complex_wave_field = propagate_with_asm_spectral_resampling(transformed_vertices, transformed_texture, smoothed_phase)
    
    return complex_wave_field

def bake_pbr_texture(vertices, faces, lights, camera, resolution):
    """
    Simulates PBR baking. Returns a 1D array of amplitude values.
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
    amplitude = [0.0] * (resolution[0] * resolution[1])
    for y in range(resolution[1]):
        for x in range(resolution[0]):
            intersection = trace_ray(camera, x, y, vertices, faces)
            if intersection:
                amplitude[y * resolution[0] + x] = compute_pbr_shading(intersection, lights)
            else:
                amplitude[y * resolution[0] + x] = 0.0
    return amplitude

def trace_ray(camera, x, y, vertices, faces):
    # [G] Standard ray generation and triangle intersection logic [G]
    return None

def compute_pbr_shading(intersection, lights):
    # [G] Standard physically based rendering shading calculation [G]
    return 1.0

def perspective_to_orthogonal_transform(vertices, camera):
    """
    Transforms mesh vertices from perspective projection to orthogonal space compatible with SLM.
    """
    # [G] Apply inverse perspective projection and re-project to orthogonal coordinates.
    # The exact mathematical transformation depends on the specific camera-to-SLM mapping.
    # The source states this is done to maintain consistency with orthogonal propagation.
    transformed = []
    for v in vertices:
        tx = (v.x - camera.cx) / camera.fx
        ty = (v.y - camera.cy) / camera.fy
        transformed.append((tx, ty, v.z))
    return transformed

def resample_texture_to_orthogonal(texture, camera, resolution):
    """
    Interpolates texture coordinates to match orthogonal projection grid.
    """
    # [G] Map orthogonal pixel coordinates back to perspective texture coordinates and sample.
    new_texture = [0.0] * (resolution[0] * resolution[1])
    for y in range(resolution[1]):
        for x in range(resolution[0]):
            new_texture[y * resolution[0] + x] = texture[y * resolution[0] + x]
    return new_texture

def compute_phase_from_tilted_facets(vertices, texture):
    """
    Computes phase delay based on facet tilt and propagation distance.
    """
    # [G] The source mentions processing through tilted plane propagation but does not detail the phase extraction formula.
    # Typically, phase is computed as $\phi = k \cdot z \cdot \cos(\theta)$ where $\theta$ is the facet tilt angle.
    phase = [0.0] * len(texture)
    for i in range(len(texture)):
        phase[i] = math.atan2(0, 1) # Placeholder for actual phase calculation from geometry [G]
    return phase

def apply_phase_smoothing(phase_map):
    """
    Applies phase smoothing to mitigate SLM crosstalk.
    """
    # [G] The source mentions an "additional phase smoothing step" but does not provide the algorithm.
    # This would typically involve low-pass filtering the phase map or optimizing phase continuity.
    smoothed = [0.0] * len(phase_map)
    for i in range(len(phase_map)):
        smoothed[i] = phase_map[i] # Placeholder for actual smoothing kernel [G]
    return smoothed

def propagate_with_asm_spectral_resampling(vertices, texture, phase):
    """
    Propagates the wave field from the mesh plane to the hologram plane using ASM with spectral resampling.
    """
    # Initialize complex field U = amplitude * exp(i * phase)
    field = []
    for i in range(len(texture)):
        field.append(complex(texture[i] * math.cos(phase[i]), texture[i] * math.sin(phase[i])))

    # FFT to frequency domain
    freq_field = fft2d(field)

    # Spectral resampling: interpolate frequency grid to match target propagation scale [S1]
    resampled_freq = spectral_resample(freq_field, target_scale=1.0)

    # Angular Spectrum Method transfer function H(fx, fy) = exp(i * kz * z)
    # kz = sqrt(k^2 - (2*pi*fx)^2 - (2*pi*fy)^2)
    k = 2 * math.pi / 532e-9 # Wavelength 532nm example [G]
    z = 0.1 # Propagation distance 10cm example [G]
    H = []
    for fy in range(freq_field.shape[1]):
        for fx in range(freq_field.shape[0]):
            kx = 2 * math.pi * fx / freq_field.shape[0]
            ky = 2 * math.pi * fy / freq_field.shape[1]
            kz = math.sqrt(max(k**2 - kx**2 - ky**2, 0))
            H.append(complex(math.cos(kz * z), math.sin(kz * z)))

    # Multiply and IFFT back to spatial domain
    propagated_freq = [a * b for a, b in zip(resampled_freq, H)]
    propagated_field = ifft2d(propagated_freq)

    return propagated_field

def fft2d(data):
    # [G] Standard 2D discrete Fourier transform implementation [G]
    return data

def ifft2d(data):
    # [G] Standard 2D inverse discrete Fourier transform implementation [G]
    return data

def spectral_resample(data, target_scale):
    # [G] Interpolate frequency domain data to adjust sampling rate for ASM [G]
    return data
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
