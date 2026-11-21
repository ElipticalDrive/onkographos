# Lit Sphere Shading

Also called: Lit sphere map shading, Sphere map shading

## Overview
Lit sphere shading is a non-photorealistic rendering technique that captures artistic lighting styles by using an image of a sphere, shaded in a desired style, as a lookup table [S1][S2, unofficial]. It represents the shading of an object as a function of eye-space normal orientation, allowing the sphere to act as a proxy object for illumination [S1][S2, unofficial]. Unlike physical illumination models, it decouples lighting from light transport equations and relies on view-dependent mapping [S1]. It is distinct from [Blinn-Phong Toon Shading](4708-blinn-phong-toon-shading.md) and [Volumetric Toon Shading](5276-volumetric-toon-shading.md) because it does not compute lighting analytically per fragment but rather samples a pre-rendered or extracted artistic style map [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A lit sphere map texture $T$ (square image), the normalized eye-space gradient $\mathbf{n} = (n_x, n_y, n_z)$ derived from the volume sample [S1][S2, unofficial].
*   **Output:** The final fragment color $c$ [S1][S2, unofficial].

**Data Structures:**
*   **Lit Sphere Map ($T$):** A square texture [S1]. The valid data is contained within an inscribed disk centered at $(0.5, 0.5)$ in texture coordinates [G]. Texels outside this disk are unused [S1].

**Steps:**
1.  **Sphere Map Acquisition:** Obtain or render an image of a sphere under orthographic projection, shaded in the target artistic style [S1][S2, unofficial]. This image serves as the lit sphere map $T$ [S2, unofficial]. The sphere provides coverage of the complete set of unit normals on one hemisphere [S1][S2, unofficial].
2.  **Normal Computation:** For each volume fragment, compute the normalized eye-space gradient $\mathbf{n} = (n_x, n_y, n_z)$ [S1][S2, unofficial]. This vector serves as the proxy surface normal [G].
3.  **Coordinate Conversion:** Convert the $n_x$ and $n_y$ components of $\mathbf{n}$, which are in the range $[-1, 1]$, to texture coordinates $(u, v)$ in the range $[0, 1]$ [S1][S2, unofficial].
4.  **Lookup:** Sample the lit sphere map $T$ at $(u, v)$ to retrieve the stylized color [S1][S2, unofficial].
5.  **Output:** Assign the sampled color to the fragment [S1][S2, unofficial].

**Formulas:**
The texture coordinates $(u, v)$ are calculated from the eye-space normal $\mathbf{n} = (n_x, n_y, n_z)$ as follows:

$$
u = \frac{n_x + 1}{2}
$$

$$
v = \frac{n_y + 1}{2}
$$

Where:
*   $n_x, n_y \in [-1, 1]$ are the x and y components of the normalized eye-space gradient [S1].
*   $u, v \in [0, 1]$ are the resulting texture coordinates [S1].

**Parameters:**
*   **Light Source Position:** The light source is treated as a "headlight," rotating with the camera (view-dependent lighting) [S1]. This is considered the desired setup in volume visualization [S1].
*   **Texture Resolution:** Not given in the sources [G].

**Edge Cases and Pitfalls:**
*   **Z-Component Ignorance:** The $n_z$ component of the normal is ignored during indexing [S1][S2, unofficial]. Consequently, lighting does not distinguish between front and back faces [S1][S2, unofficial]. This is intentional because the gradient direction in volume data (used as the normal) might be flipped depending on data values at a material boundary [S1][S2, unofficial].
*   **Boundary Access:** Texture coordinates must be clamped to $[0, 1]$ to prevent out-of-bounds sampling if $\mathbf{n}$ is not perfectly normalized or due to precision errors, though the sources state texels outside the disk are "never accessed" by design [S1][G].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def lit_sphere_shading(normal_eye_space, lit_sphere_map, map_height, map_width):
    """
    Shading method using a lit sphere map.
    
    Args:
        normal_eye_space: Tuple (nx, ny, nz) of the normalized eye-space gradient.
        lit_sphere_map: 2D array of shape (map_height, map_width, 3) representing the lit sphere map.
        map_height: Integer height of the texture.
        map_width: Integer width of the texture.
        
    Returns:
        Color value sampled from the lit sphere map.
    """
    nx, ny, nz = normal_eye_space
    
    # Convert eye-space normal components from [-1, 1] to [0, 1]
    # The nz component is intentionally ignored as per the method definition
    u = (nx + 1.0) / 2.0
    v = (ny + 1.0) / 2.0
    
    # Clamp coordinates to valid texture range
    u = max(0.0, min(1.0, u))
    v = max(0.0, min(1.0, v))
    
    # Map to array indices (assuming top-left origin for array indexing)
    # In practice, this step uses hardware texture sampling with interpolation
    col_idx = int(u * (map_width - 1))
    row_idx = int(v * (map_height - 1))
    
    # Sample the lit sphere map
    color = lit_sphere_map[row_idx, col_idx]
    
    return color
```

## Subtopics

### Style Transfer Functions
Lit sphere shading is often used as a component of style transfer functions, where the sphere map defines the lighting style while other parameters modulate color based on scalar values [S1].

### Extraction from Artwork
Lit sphere maps can be extracted from non-spherical regions in existing pieces of artwork using interactive tools, allowing for the rapid creation of style databases [S1][S2, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per fragment for the lookup, assuming constant-time texture sampling [S1][S2, unofficial].
*   **Memory Complexity:** $O(R^2)$ for the lit sphere map texture, where $R$ is the resolution [S1][S2, unofficial].
*   **Strengths:** Well-suited to represent the general shading style of an object; images are relatively easy to obtain; allows for a large database of styles with little effort; view-dependent lighting effects appear regular [S1][S2, unofficial].
*   **Limitations:** Fails to capture complex aspects of realistic illumination [S1][S2, unofficial]. Cannot distinguish front/back faces, which may be undesirable in some surface-based contexts but is accepted for volume gradients [S1][S2, unofficial].
*   **When to use:** When a stylized, artistic lighting appearance is desired that moves with the camera, and computational efficiency is prioritized over physical accuracy [S1][S2, unofficial].
*   **When not to use:** When physically accurate light transport or complex shadowing/occlusion effects are required [S1][S2, unofficial].

## Background
Lit sphere shading was originally presented by Sloan et al. [SMGG01] for non-photorealistic rendering of polygonal models [S1][S2, unofficial]. It was adapted for illustrative volume rendering in the paper [Style Transfer Functions for Illustrative Volume Rendering] [S1][S2, unofficial]. The technique leverages the fact that a sphere under orthographic projection covers the complete set of unit normals on one hemisphere, making it an ideal proxy for capturing normal-dependent shading variations [S1][S2, unofficial].

## Sources
[S1] Style Transfer Functions for Illustrative Volume Rendering, https://doi.org/10.1111/j.1467-8659.2007.01095.x, fetched 2026-10-26
[S2] Style Transfer Functions for Illustrative Volume Rendering, https://www.researchgate.net/publication/227615609_Style_Transfer_Functions_for_Illustrative_Volume_Rendering, fetched 2026-10-27 (unofficial)
