# Height-based Fog with Directional In-scatter

Also called: aerial perspective, atmospheric fog

## Overview
This method extends [Exponential Fog (Beer-Lambert)](3968-exponential-fog-beer-lambert.md) by introducing altitude-dependent density falloff and directional in-scatter to simulate aerial perspective. It solves the problem of flat, uniform fog by making distant objects appear hazy and colored by the sky or sun, thereby maintaining distance perception in large-scale scenes. It differs from [Linear Distance Fog](4409-linear-distance-fog.md) by using exponential height falloff rather than linear distance blending, and from [Fog Blending with Clouds](4123-fog-blending-with-clouds.md) by focusing on atmospheric haze rather than volumetric cloud structures.

## How it works
**Inputs:**
- Ray direction vector $\vec{d} = (dx, dy, dz)$.
- Ray origin height $y$ (world coordinates).
- Distance to the surface hit by the ray, $distance$.
- Sun direction vector $\vec{s} = (sunX, sunY, sunZ)$.
- Base fog color $fogColor_{base}$ (RGB).
- Sun fog color $sunFogColor$ (RGB).
- Atmospheric parameters: $fogDensity$, $fogHeight$, $haze$, $sunScatter$, $scattering$.
- Reference height $reference$ (default 200, or water level if enabled).

**Outputs:**
- Final fog-modulated pixel color $color_{rgb}$.

**Data Structures:**
No complex data structures are required; the method operates on scalar values and vectors derived from the ray and scene parameters.

**Steps:**

1.  **Early Exit Checks:**
    If $fogDensity \le 0$, return the original pixel color unchanged [S1, unofficial].
    If the final application amount $amount \le 0$, return the original pixel color unchanged [S1, unofficial].

2.  **Calculate Height Falloff:**
    Compute the mid-height $midY = y \times 0.5$ [S1, unofficial].
    Calculate the height falloff factor using the exponential decay based on the difference between $midY$ and the reference height:
    $$
    heightFalloff = \exp\left(-\frac{\max(0, midY - reference)}{\max(1, fogHeight)}\right)
    $$
    Here, $fogHeight$ controls the exponential thinning of fog with altitude [S1, unofficial].

3.  **Calculate Optical Depth:**
    Compute the optical depth $\tau$ (opticalDepth) combining distance, density, height falloff, and haze:
    $$
    opticalDepth = distance \times fogDensity \times heightFalloff \times haze
    $$
    $haze$ acts as a multiplier for the optical depth [S1, unofficial].

4.  **Calculate Fog Amount:**
    Determine the fog coverage amount using the Beer-Lambert law:
    $$
    fogAmount = \min(1, 1 - \exp(-opticalDepth))
    $$
    This value represents the fraction of the view ray occupied by fog [S1, unofficial].

5.  **Calculate Directional In-scatter:**
    Compute the dot product between the ray direction $\vec{d}$ and the sun direction $\vec{s}$:
    $$
    dot = \vec{d} \cdot \vec{s} = dx \cdot sunX + dy \cdot sunY + dz \cdot sunZ
    $$
    Calculate the scatter intensity by raising the positive part of the dot product to the 4th power and scaling by the sun scatter parameter:
    $$
    towardsSun = \max(0, dot)^4 \times sunScatter
    $$
    Clamp the scatter value:
    $$
    scatter = \min(1, towardsSun)
    $$
    This term simulates the directional in-scatter of sunlight, making fog brighter and warmer when looking towards the sun [S1, unofficial].

6.  **Interpolate Fog Color:**
    Blend between the base fog color and the sun fog color based on the scatter value:
    $$
    fogColor_{rgb} = fogColor_{base} + (sunFogColor_{rgb} - fogColor_{base}) \times scatter
    $$
    This creates the "aerial perspective" effect where fog takes on the color of the light source [S1, unofficial].

7.  **Apply Fog to Pixel Color:**
    Let $amount = fogAmount$ [S1, unofficial].
    Blend the computed fog color with the original pixel color $color_{rgb}$ using the scattering multiplier:
    $$
    color_{rgb} += (fogColor_{rgb} - color_{rgb}) \times amount \times scattering
    $$
    $scattering$ controls the overall intensity of the fog application [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def height_based_fog_with_directional_inscatter(
    ray_direction: Vector3,
    ray_origin_y: float,
    distance: float,
    sun_direction: Vector3,
    fog_color_base: Vector3,
    sun_fog_color: Vector3,
    fog_density: float,
    fog_height: float,
    haze: float,
    sun_scatter: float,
    scattering: float,
    reference: float = 200.0
) -> Vector3:
    # Early exit checks
    if fog_density <= 0:
        return None # Signal to skip blending, return original color
    
    # 1. Height Falloff
    mid_y = ray_origin_y * 0.5
    height_falloff = math.exp(-max(0, mid_y - reference) / max(1, fog_height))
    
    # 2. Optical Depth
    optical_depth = distance * fog_density * height_falloff * haze
    
    # 3. Fog Amount
    fog_amount = min(1, 1 - math.exp(-optical_depth))
    
    # Early exit for amount
    if fog_amount <= 0:
        return None # Signal to skip blending
    
    # 4. Directional In-scatter
    dot_product = ray_direction.dot(sun_direction)
    towards_sun = max(0, dot_product) ** 4 * sun_scatter
    scatter = min(1, towards_sun)
    
    # 5. Interpolate Fog Color
    fog_color = fog_color_base + (sun_fog_color - fog_color_base) * scatter
    
    # 6. Apply Fog
    # Note: The caller must provide the original pixel color.
    # This function returns the parameters needed for the final blend.
    return {
        "fog_color": fog_color,
        "amount": fog_amount,
        "scattering": scattering
    }

def apply_fog_blend(original_color: Vector3, fog_params: dict) -> Vector3:
    if fog_params is None:
        return original_color
    
    fog_color = fog_params["fog_color"]
    amount = fog_params["amount"]
    scattering = fog_params["scattering"]
    
    blended = original_color + (fog_color - original_color) * amount * scattering
    return blended
```

**Parameters:**
- `FOG_BASE`: Constant reference height for fog calculation, default value 200 [S1, unofficial].
- `atmosphere.fogDensity`: Controls overall fog density [S1, unofficial].
- `atmosphere.fogHeight`: Exponential falloff height for fog thinning with altitude [S1, unofficial].
- `atmosphere.haze`: Multiplier for optical depth [S1, unofficial].
- `atmosphere.sunScatter`: Controls strength of sun-directional in-scatter [S1, unofficial].
- `atmosphere.scattering`: Multiplier for fog application intensity [S1, unofficial].

**Edge Cases and Pitfalls:**
- If $fogDensity \le 0$, calculations should be skipped to avoid artifacts [S1, unofficial].
- If $amount \le 0$, color blending should be skipped [S1, unofficial].
- Precision issues may arise if $fogHeight$ is very small; using $\max(1, fogHeight)$ in the denominator prevents division by zero or extreme values [S1, unofficial].

## Subtopics

### Height Fog
A volumetric fog effect where density falls exponentially with height, filling space and sampled in closed form. It fades the horizon and fills valleys, providing a simpler alternative that lacks the directional in-scatter term [S2, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
The method is $O(1)$ per pixel as it involves a fixed number of arithmetic operations and does not require ray marching or sampling of a volume grid. Memory complexity is $O(1)$ as it stores only scalar parameters.

**Measured Performance:**
Performance is hardware-independent as it is a simple shader calculation. It scales linearly with resolution.

**Strengths and Limitations:**
- **Strengths:** Computationally cheap; provides realistic aerial perspective; maintains distance perception.
- **Limitations:** Approximates volumetric effects; does not account for multiple scattering or complex light transport within the fog.

**When to Use:**
Use for large-scale outdoor scenes where atmospheric haze is desired. Avoid when physically accurate volumetric lighting is required.

## Background
This method originates from implementations in voxel-based rendering engines, specifically MagicaVoxel and related tools, where it is used to simulate atmospheric effects in real-time [S2, unofficial]. It builds upon the Beer-Lambert law for transmittance [S1, unofficial].

## Sources
[S1] mingderwang/theme1, https://github.com/mingderwang/theme1, fetched 2026-10-27 (unofficial)
[S2] Milestone 13: MagicaVoxel look (.vox, DDA voxel grid, sun/sky, bloom, style voxel), https://github.com/adriendjn/mypbr/pull/10, fetched 2026-10-25 (unofficial)
