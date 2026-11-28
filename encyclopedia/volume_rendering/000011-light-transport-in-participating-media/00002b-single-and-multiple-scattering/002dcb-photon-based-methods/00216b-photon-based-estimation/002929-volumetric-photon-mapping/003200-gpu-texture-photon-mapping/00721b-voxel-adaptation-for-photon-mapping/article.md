# Voxel Adaptation for Photon Mapping

Also called: Photon Mapping (Voxel Adaptation)

## Overview
Voxel Adaptation for Photon Mapping adapts photon mapping for real-time voxel raytracing by decoupling photon gathering from the rendering pass, storing indirect lighting data per-voxel to eliminate per-ray gathering overhead during trace [S1, unofficial]. It differs from standard photon mapping by replacing kd-tree photon storage with a uniform grid and moving the gathering step to an offline pass, trading off estimation accuracy for real-time performance [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A scene defined as a uniform voxel grid; photon tracing functions capable of simulating light transport.
*   **Output:** A precomputed per-voxel data buffer containing indirect lighting information (colors, normals, metadata) used during the main raytracing pass.

**Data Structures:**
The method uses a uniform grid layout rather than a spatial acceleration structure like a kd-tree [S1, unofficial].
*   **Per-Voxel Storage:** Each voxel in the scene stores a structure containing:
    *   Color (RGB)
    *   Surface normals (3D vector)
    *   Photon metadata
*   **Memory Footprint:** The complete per-voxel color, normal, and photon data buffer for the tested scene occupies 16 MB [S1, unofficial].

**Algorithm Steps:**
1.  **Photon Tracing:** Trace photons through the scene using existing raytracer functions [S1, unofficial].
2.  **Storage:** Store photon data (colors, normals, and photon metadata) for each independent voxel in the scene within the uniform grid [S1, unofficial].
3.  **Estimation:** Perform irradiance/radiance estimation once after every complete photon trace pass [S1, unofficial].
    *   *Note:* The sources indicate that the irradiance estimation is currently incorrect because photons are stored in a uniform grid instead of a kd-tree, requiring a robust radiance estimation algorithm compatible with uniform grid storage [S1, unofficial].
4.  **Buffer Update:** Store the estimated results directly into the per-voxel scene data [S1, unofficial].
5.  **Rendering:** During the main raytracing pass, skip photon gathering and use the precomputed per-voxel data [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def voxel_adaptation_photon_mapping(scene, voxel_grid, photon_tracer):
    """
    Implements Voxel Adaptation for Photon Mapping.
    
    Args:
        scene: The scene definition (e.g., Cornell box).
        voxel_grid: Uniform grid structure for per-voxel storage.
        photon_tracer: Function to trace photons through the scene.
        
    Returns:
        voxel_grid: Updated with precomputed lighting data.
    """
    # Step 1: Trace photons
    photons = photon_tracer(scene)
    
    # Step 2: Store photon data in uniform grid
    # Note: Sources state irradiance estimation is currently incorrect 
    # for this structure [S1, unofficial].
    for photon in photons:
        voxel = get_voxel_at(photon.position, voxel_grid)
        voxel.color = photon.color
        voxel.normal = photon.normal
        voxel.metadata = photon.metadata
        
    # Step 3: Perform irradiance/radiance estimation
    # TODO: Implement robust radiance estimation for uniform grid storage
    # as the current method is noted as incorrect [S1, unofficial].
    for voxel in voxel_grid:
        voxel.indirect_lighting = estimate_radiance_uniform_grid(voxel, voxel_grid)
        
    # Step 4: Store results (already in voxel structure)
    # Step 5: Rendering uses precomputed data (handled by main raytracer)
    return voxel_grid
```

**Parameters:**
*   **Scene Storage Size:** 16 MB for the complete per-voxel color, normal, and photon data buffer [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Estimation Accuracy:** The irradiance estimation is incorrect when using a uniform grid instead of a kd-tree [S1, unofficial]. A robust radiance estimation algorithm compatible with uniform grid storage is required [S1, unofficial].

## Operating characteristics
*   **Performance:** Maintains ~20 FPS on an NVIDIA GeForce 8800GTS (G80) GPU after photon tracing is complete [S1, unofficial].
*   **Overhead:** The photon tracing step is performed once and causes no performance drop during subsequent raytracing frames [S1, unofficial].
*   **Strengths:** Eliminates per-ray gathering overhead during rendering, enabling real-time performance [S1, unofficial].
*   **Limitations:** Estimation accuracy is compromised due to the use of a uniform grid instead of a kd-tree [S1, unofficial].

## Background
*   **Origin:** This method was discussed in a forum archive by Ken Silverman regarding real-time voxel raytracing in development [S1, unofficial].
*   **Context:** It was presented as an alternative to polygon-based indirect lighting (such as Geomerics' real-time radiosity) for potential future game products [S1, unofficial].
*   **Scene:** The method was demonstrated on a Cornell box scene (red left, green right, white everywhere else, light top-center) [S1, unofficial].

## Sources
[S1] Forum archive / Ken Silverman / Real-time voxel raytracing, in development.., https://www.jonof.id.au/forum/topic-1433.html, fetched 2026-10-26 (unofficial)
