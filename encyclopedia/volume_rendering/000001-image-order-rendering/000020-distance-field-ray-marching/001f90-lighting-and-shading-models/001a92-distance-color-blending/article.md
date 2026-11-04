# Distance Color Blending

Also called: Depth-Cued Colors, Intensity Depth-Cuing

## Overview
Distance Color Blending is a shading technique that enhances depth perception in volume rendering by blending the sampled voxel color with a background (sky) color based on the distance from the viewer [S1, S5, unofficial]. It solves the problem of foreground/background distinction in dense volumetric data by applying a transparency falloff that dims distant objects and shifts their hue, simulating atmospheric haze or aerial perspective [S1, S5, unofficial]. Unlike standard lighting models which compute surface reflectance, this method operates on the accumulated color along the ray, modulating it non-linearly with depth [S5, unofficial]. It is distinct from [Ray-marched Shadows](1087-ray-marched-shadows.md) (which occlude light) and [Ambient Occlusion via SDF](1631-ambient-occlusion-via-sdf.md) (which darkens crevices), as it specifically targets global depth cues rather than local geometry lighting [S5, unofficial].

## How it works
**Inputs:**
*   $c_v$: The color of the volume sample at the current hit point or integration step [S5, unofficial].
*   $c_b$: The background (sky) color [S5, unofficial].
*   $d_v$: The normalized distance of the sample from the viewer, typically the fraction of the total ray length or a normalized depth value in $[0, 1]$ [S5, unofficial].
*   $k_{ds}$: Depth scaling factor controlling the magnitude of the blending effect [S5, unofficial].
*   $k_{de}$: Depth exponent controlling the rate of application (non-linearity) [S5, unofficial].
*   *Alternative Inputs for Linear Variant:* $z$: fragment depth; $z_{\max}$: maximum depth in scene; $\text{depth}$: attenuation parameter [S7].

**Outputs:**
*   $c_d$: The final depth-cued color [S5, unofficial].

**Data Structures:**
*   No specific complex data structures are required beyond standard scalar variables for color ($vec3$ or $vec4$) and distance ($float$).

**Steps:**
1.  **Calculate Distance:** Determine the distance from the eye to the current sample point. In ray marching, this is the accumulated ray length $t$ or the normalized fraction $d_v$ [S1, S5, unofficial].
2.  **Compute Blending Factor:** Calculate the blending factor $B$ (or $s$) based on the distance and chosen falloff function (exponential or linear) [S5, unofficial][S7].
3.  **Blend Colors:** Interpolate between the volume color $c_v$ and the background color $c_b$ using the blending factor [S5, unofficial].
4.  **Output:** Return the resulting color $c_d$ [S5, unofficial].

**Formulas:**

**Exponential Depth Cueing (Standard):**
The blending factor $B$ is calculated as:
$$ B = k_{ds} d_v^{k_{de}} $$
The final color is:
$$ c_d = (1 - B) c_v + B c_b $$
Where:
*   $c_d$ is the depth-cued color [S5, unofficial].
*   $c_v$ is the original voxel color [S5, unofficial].
*   $c_b$ is the background color [S5, unofficial].
*   $d_v$ is the fraction of distance through the volume [S5, unofficial].
*   $k_{ds}$ controls the size/magnitude of the blending effect [S5, unofficial].
*   $k_{de}$ controls the rate of application (exponent) [S5, unofficial].

*Note: Some implementations use a cubic exponential falloff for the fog factor $F$:*
$$ F = 1.0 - \exp(-k d^3) $$
$$ C_{final} = C_{obj} \cdot (1 - F) + C_{bg} \cdot F $$
Where $k = 10^{-7}$ and $d$ is the object distance [S3, unofficial].

*Note: Another exponential variant uses ray distance $t$ directly:*
$$ \text{fog} = 1.0 - \exp(-0.0035 \cdot t^2) $$
$$ \text{col} = \text{mix}(\text{lit\_color}, \text{sky\_color}, \text{fog}) $$
Clamped to $[0.0, 1.0]$ [S4, unofficial].

**Linear Depth Cueing:**
The blending factor $s$ is calculated as:
$$ s = \frac{1 + \text{depth} \times z}{1 + z_{\max}} $$
The final color intensity $I$ is modified to:
$$ I_{final} = s I + (1 - s) I_{bg} $$
Where:
*   $z$ is the z-value of the fragment [S7].
*   $z_{\max}$ is the maximal z-value in the scene [S7].
*   $\text{depth}$ is the attenuation parameter [S7].
*   $I$ is the original intensity [S7].
*   $I_{bg}$ is the background color intensity [S7].

**Parameters:**
*   $k_{ds}$: Typical value $1.0$ [S5, unofficial].
*   $k_{de}$: Typical value $0.5$ [S5, unofficial].
*   $c_b$: Background color. If grey $(a,a,a)$, only intensity dimming occurs. If blue $(a,b,c)$ with $c>a,b$, a cool shift is introduced to simulate aerial perspective [S5, unofficial].
*   Fog density coefficient: $0.0035$ (for the $t^2$ exponential variant) [S4, unofficial].
*   $k$: $10^{-7}$ (for the $d^3$ exponential variant) [S3, unofficial].
*   Linear depth parameter: Non-zero to enable attenuation [S7].

**Edge Cases and Pitfalls:**
*   **Precision:** Ensure $d_v$ is normalized correctly to avoid blending artifacts at the volume boundaries [S5, unofficial].
*   **Color Shift:** Using a non-grey background color $c_b$ introduces a hue shift (e.g., blue shift for distance). This should be intentional to simulate atmospheric scattering; otherwise, use a grey background for pure intensity dimming [S5, unofficial].
*   **Clamping:** The blending factor and final color must be clamped to $[0.0, 1.0]$ to prevent overflow or negative values [S4, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def distance_color_blending(c_v, c_b, d_v, k_ds, k_de):
    """
    Exponential Distance Color Blending.
    
    Args:
        c_v (vec3): Volume color at current sample.
        c_b (vec3): Background color.
        d_v (float): Normalized distance through volume [0, 1].
        k_ds (float): Depth scaling factor.
        k_de (float): Depth exponent.
        
    Returns:
        vec3: Depth-cued color.
    """
    # Calculate blending factor B
    # B = k_ds * d_v ^ k_de
    # Note: If d_v is 0, B is 0. If d_v is 1, B is k_ds.
    # Ensure k_ds is scaled if necessary to keep B in [0, 1] range for full depth.
    B = k_ds * (d_v ** k_de)
    
    # Clamp B to [0, 1]
    B = max(0.0, min(1.0, B))
    
    # Blend colors
    c_d = (1.0 - B) * c_v + B * c_b
    
    return c_d

def linear_depth_cueing(c_v, c_b, z, z_max, depth_param):
    """
    Linear Depth Cueing.
    
    Args:
        c_v (vec3): Volume color.
        c_b (vec3): Background color.
        z (float): Fragment depth.
        z_max (float): Maximal depth in scene.
        depth_param (float): Attenuation parameter.
        
    Returns:
        vec3: Depth-cued color.
    """
    # Calculate blending factor s
    s = (1.0 + depth_param * z) / (1.0 + z_max)
    
    # Clamp s to [0, 1]
    s = max(0.0, min(1.0, s))
    
    # Blend colors
    c_d = s * c_v + (1.0 - s) * c_b
    
    return c_d
```

## Subtopics

### [Exponential Depth Cueing](3708-exponential-depth-cueing.md)
Exponential Depth Cueing is a shading technique that simulates atmospheric fog to enhance depth perception in volume rendering by dimming distant voxels. It solves the problem of foreground/background distinction in dense volumetric data by applying a non-linear transparency falloff based on normalized depth. Unlike linear cueing, it uses an exponential function to control how rapidly objects recede into darkness, allowing for more natural aerial perspective effects. See [Exponential Depth Cueing](3708-exponential-depth-cueing.md).

### Linear Depth Cueing
Linear Depth Cueing is a technique to simulate atmospheric attenuation as depth cuing, blending colors of distant objects with the background color to indicate distance [S7]. It uses a linear interpolation factor based on the fragment's z-depth relative to the maximum scene depth [S7]. It is simpler to implement but may appear less natural than exponential variants for large depth ranges [S7].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per sample. The blending calculation involves a few arithmetic operations (multiplication, exponentiation, addition) and is negligible compared to the ray marching or volume integration cost [S5, unofficial].
*   **Memory Complexity:** $O(1)$. No additional buffers are required beyond the color and depth values [S5, unofficial].
*   **Performance:** The technique is computationally inexpensive and can be applied in the fragment shader or during the final color accumulation step without significant performance impact [S1, S5, unofficial].
*   **Strengths:** Enhances depth perception significantly in dense volumes; simple to implement; adds atmospheric realism [S1, S5, unofficial].
*   **Limitations:** Does not account for actual light scattering physics (it is a heuristic); the choice of parameters ($k_{ds}, k_{de}$) is artistic and may require tuning for different scenes [S5, unofficial].
*   **When to Use:** When depth perception is critical in dense volumetric data, or when simulating atmospheric effects like fog or haze [S1, S5, unofficial].
*   **When Not to Use:** When physical accuracy of light transport is required, or when the volume is sparse and depth cues are already provided by geometry [S5, unofficial].

## Background
Distance Color Blending, also known as Intensity Depth-Cuing, is a well-known technique for enhancing the perception of depth in a scene, originally described in computer graphics literature [Foley96, cited in S5]. It dims the color of objects far from the viewer, creating an effect similar to viewing the scene through haze [S5, unofficial]. The technique has been adapted for volume rendering to dim volume sample colors as they recede from the viewer [S5, unofficial]. It has also been augmented with color modulation to increase blue in distant samples, simulating aerial perspective used by painters for centuries [S5, unofficial]. The method is often extended to optical models in volume rendering to enhance depth perception [S6].

## Sources
[S1] Raymarching Distance Fields (2013), http://9bitscience.blogspot.com/2013/07/raymarching-distance-fields_14.html, fetched 2026-10-26 (unofficial)
[S2] evjapont-hash/CubeWorld-Canyon, https://github.com/evjapont-hash/CubeWorld-Canyon, fetched 2026-10-26 (unofficial)
[S3] kushchevd/RayMarching, https://github.com/kushchevd/RayMarching, fetched 2026-10-26 (unofficial)
[S4] MohammadHossinzehi/sdf-raymarcher, https://github.com/MohammadHossinzehi/sdf-raymarcher, fetched 2026-10-26 (unofficial)
[S5] Non-Photorealistic Rendering of Volume Models, https://userpages.cs.umbc.edu/ebert/npr/volillus.pdf, fetched 2026-10-27 (unofficial)
[S6] Hardware-based view-independent cell projection, https://www.semanticscholar.org/paper/Hardware-based-view-independent-cell-projection-Weiler-Kraus/36615d19709bce724437ec3257e43a1916f7de26, fetched 2026-10-29
[S7] Computing and Displaying Isosurfaces inR, https://doi.org/10.18637/jss.v028.i01, fetched 2026-10-27
