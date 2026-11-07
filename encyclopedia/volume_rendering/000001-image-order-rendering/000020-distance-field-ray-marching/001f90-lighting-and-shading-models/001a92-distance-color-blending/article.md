# Distance Color Blending

Also called: Depth-Cued Colors, Intensity Depth-Cuing, aerial perspective

## Overview
Distance Color Blending is a shading technique that enhances depth perception in volume rendering by blending sampled voxel colors with a background color based on distance from the viewer [S1, unofficial][S5]. It solves the foreground/background distinction problem in dense volumetric data by applying a non-linear transparency falloff that dims distant objects and shifts their hue, simulating atmospheric haze or aerial perspective [S1, unofficial][S5]. Unlike standard lighting models that compute surface reflectance, this method operates on the accumulated color along the ray, modulating it with depth [S5].

## How it works
**Inputs:**
*   $c_v$: The color of the volume sample at the current hit point or integration step [S5].
*   $c_b$: The background (sky) color [S5].
*   $d_v$: The normalized distance of the sample from the viewer, typically the fraction of the total ray length in $[0, 1]$ [S5].
*   $k_{ds}$: Depth scaling factor controlling the magnitude of the blending effect [S5].
*   $k_{de}$: Depth exponent controlling the rate of application (non-linearity) [S5].
*   *Alternative Inputs for Linear Variant:* $z$: fragment depth; $z_{\max}$: maximum depth in scene; $\text{depth}$: attenuation parameter [S9].

**Outputs:**
*   $c_d$: The final depth-cued color [S5].

**Data Structures:**
*   No specific complex data structures are required beyond standard scalar variables for color ($vec3$ or $vec4$) and distance ($float$) [G].

**Steps:**
1.  **Calculate Distance:** Determine the distance from the eye to the current sample point [S1, unofficial].
2.  **Compute Blending Factor:** Calculate the blending factor $B$ (or $s$) based on the distance and chosen falloff function (exponential or linear) [S5][S9].
3.  **Blend Colors:** Interpolate between the volume color $c_v$ and the background color $c_b$ using the blending factor [S5].
4.  **Output:** Return the resulting color $c_d$ [S5].

**Formulas:**

**Exponential Depth Cueing (Standard):**
The blending factor $B$ is calculated as:
$$ B = k_{ds} d_v^{k_{de}} $$
The final color is:
$$ c_d = (1 - B) c_v + B c_b $$
Where:
*   $c_d$ is the depth-cued color [S5].
*   $c_v$ is the original voxel color [S5].
*   $c_b$ is the background color [S5].
*   $d_v$ is the fraction of distance through the volume [S5].
*   $k_{ds}$ controls the size/magnitude of the blending effect [S5].
*   $k_{de}$ controls the rate of application (exponent) [S5].

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
*   $z$ is the z-value of the fragment [S9].
*   $z_{\max}$ is the maximal z-value in the scene [S9].
*   $\text{depth}$ is the attenuation parameter [S9].
*   $I$ is the original intensity [S9].
*   $I_{bg}$ is the background color intensity [S9].

**Parameters:**
*   $k_{ds}$: Typical value $1.0$ [S5].
*   $k_{de}$: Typical value $0.5$ [S5].
*   $c_b$: Background color. If grey $(a,a,a)$, only intensity dimming occurs [S7, unofficial]. If blue $(a,b,c)$ with $c>a,b$, a cool shift is introduced to simulate aerial perspective [S5][S6, unofficial].
*   Fog density coefficient: $0.0035$ (for the $t^2$ exponential variant) [S4, unofficial].
*   $k$: $10^{-7}$ (for the $d^3$ exponential variant) [S3, unofficial].
*   Linear depth parameter: Non-zero to enable attenuation [S9].

**Edge Cases and Pitfalls:**
*   **Precision:** Ensure $d_v$ is normalized correctly to avoid blending artifacts at the volume boundaries [S5].
*   **Color Shift:** Using a non-grey background color $c_b$ introduces a hue shift (e.g., blue shift for distance) [S6, unofficial]. This should be intentional to simulate atmospheric scattering; otherwise, use a grey background for pure intensity dimming [S5].
*   **Clamping:** The blending factor and final color must be clamped to $[0.0, 1.0]$ to prevent overflow or negative values [S4, unofficial].
*   **Hard Transfer Functions:** Hard transfer functions hide information in some regions; this technique avoids that by modulating existing colors rather than masking them [S5].

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
    # Calculate blending factor B = k_ds * d_v ^ k_de
    B = k_ds * (d_v ** k_de)
    
    # Clamp B to [0, 1]
    B = max(0.0, min(1.0, B))
    
    # Blend colors: c_d = (1 - B) * c_v + B * c_b
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
    # Calculate blending factor s = (1 + depth * z) / (1 + z_max)
    s = (1.0 + depth_param * z) / (1.0 + z_max)
    
    # Clamp s to [0, 1]
    s = max(0.0, min(1.0, s))
    
    # Blend colors: I_final = s * I + (1 - s) * I_bg
    c_d = s * c_v + (1.0 - s) * c_b
    
    return c_d
```

## Subtopics

### [Exponential Depth Cueing](3708-exponential-depth-cueing.md)
Simulates atmospheric fog by dimming distant voxels using a non-linear transparency falloff based on normalized depth. It sets itself apart from linear cueing by using an exponential function to control how rapidly objects recede, allowing for more natural aerial perspective effects. Choose it when realistic atmospheric depth perception is prioritized over computational simplicity.

### Linear Depth Cueing
Simulates atmospheric attenuation by blending distant object colors with the background color using a linear interpolation factor based on fragment z-depth relative to maximum scene depth [S9]. It is simpler to implement but may appear less natural than exponential variants for large depth ranges [S9]. Choose it for straightforward depth indication or when linear falloff matches the scene's scale. [G]

## Operating characteristics
*   **Time Complexity:** Not given in the sources. [G]
*   **Memory Complexity:** Not given in the sources. [G]
*   **Measured Performance:** Not given in the sources. [G]
*   **Scaling:** Not given in the sources. [G]
*   **Strengths:** Enhances depth perception in dense volumes; simple to implement; adds atmospheric realism [S1, unofficial][S5].
*   **Limitations:** Does not account for actual light scattering physics; parameter tuning ($k_{ds}, k_{de}$) is artistic [S5].
*   **When to Use:** When depth perception is critical in dense volumetric data, or when simulating atmospheric effects like fog or haze [S1, unofficial][S5].
*   **When Not to Use:** Not given in the sources. [G]

## Background
Originally described as intensity depth-cuing in computer graphics literature [S5]. It dims distant object colors to simulate viewing through haze [S5]. Adapted for volume rendering to dim sample colors as they recede, augmented with color modulation to increase blue in distant samples, simulating aerial perspective used by painters for centuries [S5][S6, unofficial]. Extended to optical models in volume rendering to enhance depth perception [S8].

## Sources
[S1] Raymarching Distance Fields (2013), http://9bitscience.blogspot.com/2013/07/raymarching-distance-fields_14.html, fetched 2026-10-26 (unofficial)
[S2] evjapont-hash/CubeWorld-Canyon, https://github.com/evjapont-hash/CubeWorld-Canyon, fetched 2026-10-26 (unofficial)
[S3] kushchevd/RayMarching, https://github.com/kushchevd/RayMarching, fetched 2026-10-26 (unofficial)
[S4] MohammadHossinzehi/sdf-raymarcher, https://github.com/MohammadHossinzehi/sdf-raymarcher, fetched 2026-10-26 (unofficial)
[S5] PDF Volume illustration: nonphotorealistic rendering of volume models ..., https://www.csee.umbc.edu/~rheingan/pubs/volillus.tvcg01.pdf, fetched 2026-10-27
[S6] Non-Photorealistic Rendering of Volume Models, https://www.cs.umbc.edu/~rheingan/pubs/volillus00.pdf, fetched 2026-10-27 (unofficial)
[S7] Non-Photorealistic Rendering of Volume Models, https://userpages.cs.umbc.edu/ebert/npr/volillus.pdf, fetched 2026-10-27 (unofficial)
[S8] Hardware-based view-independent cell projection, https://www.semanticscholar.org/paper/Hardware-based-view-independent-cell-projection-Weiler-Kraus/36615d19709bce724437ec3257e43a1916f7de26, fetched 2026-10-29
[S9] Computing and Displaying Isosurfaces inR, https://doi.org/10.18637/jss.v028.i01, fetched 2026-10-27
