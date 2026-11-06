# SDF-Accelerated Volumetric Cloud Rendering

Also called: Ray marching with Signed Distance Fields

## Overview
A technique to accelerate real-time volumetric cloud rendering by using a Signed Distance Field (SDF) to skip empty space and determine step sizes, allowing for high sample counts without performance loss [S1, unofficial]. It solves the problem of wasting GPU cycles calculating zero-value samples in sparse cloud volumes, which previously led to 'soft, undefined volumes' or 'horrible banding/noisy output' if sample counts were low [S1, unofficial]. Unlike uniform sampling, this method spends samples only where clouds exist, avoiding expensive calculations for empty space [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A static cloud volume represented by an SDF texture and a density/coverage texture. A ray origin and direction.
*   **Output:** The final color and alpha of the ray, integrating scattering and transmittance.

**Data Structures:**
1.  **SDF Texture:** A signed, half-precision floating point 3D texture.
    *   **Resolution:** $128 \times 32 \times 128$ [S1, unofficial].
    *   **Layout:** Each entry contains the distance to the nearest point of the cloud volume shell. Inside the volume, values are negative [S1, unofficial].
    *   **Scaling:** The texture is stretched by a factor of 6 horizontally to cover a larger area and must tile horizontally for an infinitely extending cloud plane [S1, unofficial].
2.  **Cloud Volume Texture:** An RG16 3D texture of the same resolution as the SDF.
    *   **Red Channel:** Coverage.
    *   **Green Channel:** Density [S1, unofficial].

**Pre-computation:**
The SDF is pre-computed using a layered system where a compute shader runs for each cloud layer, adding to the RG16 3D texture. The method for generating cloud shapes is similar to the method described by Sébastien Hillaire [S1, unofficial]. Recalculating the distance field texture is expensive (takes about 200ms using a 3-pass compute shader on a GTX1070), so the system is limited to a static cloud volume [S1, unofficial]. Movement is achieved by scrolling the volume itself [S1, unofficial].

**Ray Marching Algorithm:**
The ray marcher operates in two modes:
1.  **SDF Mode:** Used when outside the cloud volume. The step size is determined by the SDF value.
2.  **Regular Mode:** Used when inside the cloud volume. The step size is fixed.

Areas outside of the cloud volume are skipped entirely [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Constants
SDF_RESOLUTION = (128, 32, 128)
HORIZONTAL_SCALE = 6.0
FIXED_STEP_SIZE = 0.1 # Example value, source uses _StepSize uniform

# Helper to map world position to SDF texture coordinates
def world_to_sdf_coords(world_pos):
    # Apply horizontal stretch and tiling
    x = (world_pos.x * HORIZONTAL_SCALE) % SDF_RESOLUTION[0]
    y = world_pos.y % SDF_RESOLUTION[1]
    z = world_pos.z % SDF_RESOLUTION[2]
    return (x / SDF_RESOLUTION[0], y / SDF_RESOLUTION[1], z / SDF_RESOLUTION[2])

# Helper to get density from SDF value
def density_function(sdf_value):
    # Inside volume (negative SDF), density is positive distance to shell
    # Outside volume (positive SDF), density is 0
    return max(-sdf_value, 0.0)

# Main Ray Marching Loop
def ray_march(ray_origin, ray_direction):
    # Initialize integration variables
    accumulated_color = (0.0, 0.0, 0.0)
    accumulated_alpha = 0.0
    
    current_pos = ray_origin
    
    # Determine if we are initially inside or outside
    # Note: In a real engine, you might check bounding boxes first
    sdf_val = sample_sdf(current_pos)
    is_inside = sdf_val < 0
    
    # Step through the ray
    while current_pos is within bounds and accumulated_alpha < 1.0:
        
        if is_inside:
            # Regular Mode: Fixed step size inside the cloud
            step_size = FIXED_STEP_SIZE
        else:
            # SDF Mode: Step size determined by distance to surface
            # SDF value is positive outside, representing distance to nearest surface
            step_size = sdf_val
            
            # If step size is too small or zero, break to avoid infinite loops
            if step_size < 1e-6:
                break
        
        # Advance ray
        next_pos = current_pos + ray_direction * step_size
        
        # Sample SDF at next position to determine mode for next iteration
        next_sdf_val = sample_sdf(next_pos)
        next_is_inside = next_sdf_val < 0
        
        # If we are currently inside, we need density for extinction
        if is_inside:
            density = density_function(sdf_val)
            
            # Transmittance calculation: T = e^(-extinction * step_size)
            # Extinction is derived from density
            extinction = density
            transmittance = exp(-extinction * step_size)
            
            # Get luminance for the current ray position
            luminance = get_luminance(current_pos)
            
            # Integrate scattering
            # Scattered light = Luminance * (1 - Transmittance)
            # Note: Source uses 'integScatt = luminance - luminance * transmittance'
            # which is equivalent to luminance * (1 - transmittance)
            scattered_light = luminance * (1.0 - transmittance)
            
            # Accumulate color and alpha
            # Source: intScattTrans.rgb += integScatt * intScattTrans.a
            # Source: intScattTrans.a *= transmittance
            accumulated_color += scattered_light * (1.0 - accumulated_alpha)
            accumulated_alpha += (1.0 - accumulated_alpha) * (1.0 - transmittance)
            
            # Update state
            current_pos = next_pos
            sdf_val = next_sdf_val
            is_inside = next_is_inside
        else:
            # Outside mode: Just advance, no scattering calculation needed
            # Skip areas outside of the cloud volume entirely
            current_pos = next_pos
            sdf_val = next_sdf_val
            is_inside = next_is_inside
            
    return accumulated_color, accumulated_alpha

def sample_sdf(pos):
    coords = world_to_sdf_coords(pos)
    # Sample the signed distance field texture
    return texture_3d(sdf_texture, coords)

def get_luminance(pos):
    # Sample the cloud volume texture for density/coverage and compute luminance
    # This depends on the specific cloud shader implementation
    pass
```

**Parameters:**
*   **SDF Texture Resolution:** $128 \times 32 \times 128$ [S1, unofficial].
*   **Horizontal Stretch Factor:** 6 [S1, unofficial].
*   **SDF Texture Format:** Signed, half-precision floating point 3D texture [S1, unofficial].
*   **Fixed Step Size:** Uniform variable `_StepSize` [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** The SDF texture uses half-precision floats, which may introduce quantization artifacts in distance calculations [S1, unofficial].
*   **Static Volume:** The SDF pre-computation is expensive (~200ms on GTX1070), so the cloud volume must be static [S1, unofficial]. Dynamic clouds are not supported without significant performance cost or alternative techniques.
*   **Tiling:** The SDF texture must tile horizontally to support an infinitely extending cloud plane [S1, unofficial].
*   **Inside/Outside Transition:** The switch between SDF and Regular modes must be handled carefully to avoid gaps or overlaps in the ray march [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Pre-computation is $O(N)$ where $N$ is the number of voxels in the SDF texture, but is done offline. Ray marching is $O(M)$ where $M$ is the number of samples taken. The SDF allows skipping large empty spaces, reducing $M$ significantly compared to uniform sampling.
*   **Memory Complexity:** $O(R^3)$ for the SDF texture, where $R$ is the resolution. For $128 \times 32 \times 128$, this is $524,288$ half-floats, which is relatively small.
*   **Measured Performance:** Pre-computation takes ~200ms on a GTX1070 [S1, unofficial]. Real-time rendering is possible with changing time of day and directional shadows [S1, unofficial].
*   **Strengths:** High visual fidelity with well-defined cloud shapes. Efficient use of sample budget by skipping empty space.
*   **Limitations:** Static cloud volume only. Pre-computation cost. Requires careful tuning of step sizes and texture resolution.
*   **When to Use:** Real-time rendering of static volumetric clouds where visual quality is prioritized over dynamic cloud formation.
*   **When Not to Use:** Dynamic cloud simulations, mobile platforms with very limited memory, or scenes requiring frequent updates to the cloud volume.

## Background
This technique was developed by the author of the blog post "Rendering volumetric clouds using signed distance fields" [S1, unofficial]. It was implemented in the author's game engine 'UHawk' and demonstrated in a video of the game [S1, unofficial]. The author was inspired by observing sparse but dense cumulus cloud formations and the frustration of achieving well-defined shapes with uniform sampling [S1, unofficial]. The concept of using SDFs for ray marching is based on the work of Jamie Wong, who provided a full overview of signed distance fields [S1, unofficial]. The cloud shape generation method is similar to the method described by Sébastien Hillaire [S1, unofficial].

## Sources
[S1] Rendering volumetric clouds using signed distance fields - MYBOOAM, https://www.mybooam.tech/news/11576/2020/05/06/rendering-volumetric-clouds-using-signed-distance-fields/, fetched 2026-10-26 (unofficial)
