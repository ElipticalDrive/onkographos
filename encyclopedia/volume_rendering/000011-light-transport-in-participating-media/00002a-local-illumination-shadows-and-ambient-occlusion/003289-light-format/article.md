# Light Format

Also called: Light data structure, Light object fields

## Overview
Light Format defines the data structure and memory layout for light sources within a volumetric rendering pipeline, specifically for GPU-based ray marching. It solves the problem of efficiently transferring and accessing light parameters (position, color, attenuation, shadowing) from host to device memory. It is distinguished from general lighting models by its specific handling of homogeneous coordinates for light type differentiation and its integration with a fixed-size light vector where the first two entries serve special roles for sky and fog calculations.

## How it works
**Inputs and Outputs:**
- Input: A list of light parameters defined on the host CPU.
- Output: A contiguous block of light data in device (GPU) memory, typically accessed as an array or buffer during ray marching.

**Data Structures:**
The core structure is the `Light` object. Lights are stored in a vector (array) on the host and copied once per frame to device memory. The first two lights in this vector are reserved for special "sky" and "fog" calculations [S1].

**Light Object Fields:**
1.  **Position:** A homogeneous 3D vector ($\mathbf{p} = [x, y, z, w]$).
    -   If $w = 1$, the light is a **Point Light**. The vector represents the spatial position.
    -   If $w = 0$, the light is a **Directional Light**. The vector $[x, y, z]$ represents the direction of the light rays.
2.  **Colour:** A homogeneous 3D vector ($\mathbf{c} = [r, g, b, i]$).
    -   The RGB components represent the light's color.
    -   The 4th component $i$ represents the **inverse intensity** ($1/I$). This allows multiplication by intensity to be replaced by division or scaling during shading calculations.
3.  **Attenuation:** A 3-component vector $\mathbf{a} = [a_c, a_l, a_q]$ representing constant, linear, and quadratic attenuation factors.
    -   *Pitfall:* If the light is a "sky light" (one of the first two in the vector), these values are ignored and treated as $[1, 0, 0]$ because sky lights are considered infinitely distant [S1].
4.  **Hardness:** A linear float representing shadow softness.
    -   Range: Minimum 3 (maximum softness without artifacts) to Maximum 128 (perfectly sharp shadows) [S1].
5.  **Radius:** A linear float representing the maximum distance the light can illuminate [S1].
6.  **ShadowRadius:** A linear float representing the maximum distance the light can cast shadows [S1].
7.  **Shadows:** A boolean flag enabling or disabling shadow casting for this light [S1].

**Numbered Steps:**
1.  **Host Initialization:** Create a vector of `Light` objects on the CPU. Populate fields based on scene requirements.
2.  **Special Light Assignment:** Assign the first two entries in the vector to sky and fog lights. Ensure their attenuation is effectively $[1, 0, 0]$ [S1].
3.  **Memory Transfer:** Copy the entire vector of lights to device memory once per frame. This is done in the same manner as material data [S1].
4.  **Device Access:** During ray marching, the shader accesses the light at index $i$ from the device buffer to compute illumination at the current sample point.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Definition of Light structure for implementation
class Light:
    position: Vector4  # xyz = pos/dir, w = 1 (point) or 0 (directional)
    color: Vector4     # rgb = color, w = inverse intensity
    attenuation: Vector3 # constant, linear, quadratic
    hardness: float    # 3.0 to 128.0
    radius: float      # max illumination distance
    shadow_radius: float # max shadow casting distance
    shadows: bool      # enable/disable shadows

def setup_lights(host_lights: List[Light]):
    """
    Prepare lights for GPU transfer.
    """
    # Ensure at least two lights exist for sky/fog
    if len(host_lights) < 2:
        raise ValueError("At least two lights required for sky and fog")

    # Special handling for sky/fog lights (indices 0 and 1)
    # Sky lights are infinitely distant, so attenuation is ignored
    for i in range(2):
        host_lights[i].attenuation = Vector3(1.0, 0.0, 0.0)

    # Copy to device memory (pseudo-operation)
    device_light_buffer = upload_to_gpu(host_lights)
    return device_light_buffer

def evaluate_light(light: Light, point: Vector3) -> float:
    """
    Calculate attenuation factor for a given point.
    Note: This is a helper for shading, not the full format definition.
    """
    # Determine if point light or directional
    if light.position.w == 1.0:
        # Point light: calculate distance
        dist = length(light.position.xyz - point)
        # Apply attenuation
        if dist > 0:
            attenuation_factor = light.attenuation.x + \
                                 light.attenuation.y * dist + \
                                 light.attenuation.z * dist * dist
            return 1.0 / attenuation_factor
        else:
            return 1.0
    else:
        # Directional light: no attenuation based on distance
        return 1.0
```

**Parameters with Typical Values:**
-   **Hardness:** 3 (soft) to 128 (sharp) [S1].
-   **Sky Light Attenuation:** $[1, 0, 0]$ [S1].
-   **Default Point Light 1 (from variant):** Position $(50, 10, 30)$, Diffuse $(1,1,1)$, Specular $(0,0,1)$, Attenuation 50, Falloff 2 [S2, unofficial].
-   **Default Point Light 2 (from variant):** Position $(-10, 10, -10)$, Diffuse $(1,0,0)$, Specular $(1,0,0)$, Attenuation 20, Falloff 20 [S2, unofficial].

**Edge Cases and Pitfalls:**
-   **Sky Light Attenuation:** Sky lights must not use standard attenuation formulas. The source explicitly states that attenuation values are ignored and $[1, 0, 0]$ is used because they are infinitely distant [S1].
-   **Light Type Ambiguity:** The type of light is determined solely by the $w$ component of the position vector. Implementers must check $w$ before interpreting $xyz$ as position or direction.
-   **Indexing:** The first two lights have special semantic meaning (sky/fog). General lighting logic should skip or handle these indices differently if they are not intended for direct object illumination.

## Subtopics

### Point Light Definition
A specific variant of the light format used in voxel game engines (e.g., MonoGame-based implementations). It differs from the general CUDA raymarching format by explicitly separating diffuse and specular colors and using a scalar attenuation/falloff model rather than a 3-component vector. It is chosen when a simpler, per-light scalar falloff is preferred over the quadratic attenuation model.
-   **Fields:** `DiffuseColour` (Vector3), `Position` (Vector3), `SpecularColour` (Vector3), `Attenuation` (float), `Falloff` (float) [S2, unofficial].
-   **Usage:** Used in [TerryLyons16/MonoGame_VoxelGameEngine](https://github.com/TerryLyons16/MonoGame_VoxelGameEngine) [S2, unofficial].

## Operating characteristics
-   **Memory:** The light data is small and fixed-size per light. The entire vector is transferred once per frame, making the overhead negligible compared to volume data transfer.
-   **Performance:** Accessing light data in the shader is a simple memory load. The complexity of lighting evaluation depends on the shading model (e.g., shadow ray casting), not the format itself.
-   **Strengths:** Simple, compact representation. Homogeneous coordinates allow unified handling of point and directional lights.
-   **Limitations:** The fixed vector size and special indexing for sky/fog lights may require adaptation for scenes with many dynamic lights or different sky models.

## Background
The Light Format described here originates from CUDA-based raymarching distance field rendering [S1]. It was designed to support real-time volumetric rendering with shadows and ambient effects. The variant with scalar attenuation and separate specular colors comes from a voxel game engine implementation [S2, unofficial].

## Sources
[S1] Raymarching Distance Fields with CUDA, https://doi.org/10.3390/electronics10222730, fetched 2026-10-26
[S2] TerryLyons16/MonoGame_VoxelGameEngine, https://github.com/TerryLyons16/MonoGame_VoxelGameEngine, fetched 2026-10-27 (unofficial)
