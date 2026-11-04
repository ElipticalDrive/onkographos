# Lit Sphere Shading

Also called: Lit sphere map shading, Sphere map shading

## Overview
Lit sphere shading is a non-photorealistic illumination technique that captures artistic lighting styles by using an image of a sphere, shaded in a desired style, as a lookup table [S1][S2, unofficial]. It represents the shading of an object as a function of eye-space normal orientation, allowing the sphere to act as a proxy object for illumination [S1][S2, unofficial]. Unlike photometric models that calculate light transport, this method maps normal vectors directly to pre-computed color variations, providing view-dependent lighting that rotates with the camera [S1][S2, unofficial]. It is distinct from [Volume Illustration Tone Shading](805-volume-illustration-tone-shading.md) and [Vignette Shading](1830-vignette-shading.md) by its reliance on a geometric proxy (the sphere) rather than scalar field properties or screen-space effects, and from [Blinn-Phong Toon Shading](4708-blinn-phong-toon-shading.md) by its ability to encode complex, non-physical artistic styles rather than simple geometric light interactions [S1][S2, unofficial].

## How it works

### Inputs and Outputs
**Inputs:**
1.  **Lit Sphere Map:** A square texture containing an image of a sphere under orthographic projection, shaded in the target artistic style [S1][S2, unofficial].
2.  **Eye-space Normal:** The normal vector $\mathbf{n} = (n_x, n_y, n_z)$ of the fragment being shaded, expressed in the camera's coordinate system [S1][S2, unofficial].

**Outputs:**
1.  **Shaded Color:** The final color value for the fragment, sampled from the lit sphere map [S1][S2, unofficial].

### Data Structures
**Lit Sphere Map:**
*   **Format:** Square texture (e.g., $N \times N$ pixels).
*   **Content:** An orthographic projection of a sphere shaded with the desired artistic style [S1][S2, unofficial].
*   **Layout:** Texels outside an inscribed disk are never accessed [S1][S2, unofficial]. Normal vectors parallel to the viewing direction map to the center of the disk; normal vectors orthogonal to the viewing direction map to the circumference of the disk [S1][S2, unofficial].
*   **Indexing:** The map is indexed using only the $n_x$ and $n_y$ components of the eye-space normal [S1][S2, unofficial].

### Parameters
*   **Light Source Position:** Treated as a "headlight" that rotates with the camera (view-dependent lighting) [S1]. This is the standard setup for volume visualization [S1].
*   **Texture Coordinates:** The $n_x$ and $n_y$ components, originally in the range $[-1, 1]$, are converted to the texture coordinate range $[0, 1]$ [S1][S2, unofficial].

### Steps
1.  **Generate or Obtain Lit Sphere Map:** Render or extract an image of a sphere under orthographic projection shaded in the target artistic style [S1][S2, unofficial]. This image serves as the lit sphere map [S2, unofficial].
2.  **Compute Eye-space Normal:** For each fragment, compute the eye-space normal $\mathbf{n} = (n_x, n_y, n_z)$ [S1][S2, unofficial].
3.  **Convert to Texture Coordinates:** Convert the $n_x$ and $n_y$ components from the range $[-1, 1]$ to the texture coordinate range $[0, 1]$ [S1][S2, unofficial].
4.  **Sample Map:** Use the converted coordinates to index the lit sphere map and retrieve the shaded color [S1][S2, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def lit_sphere_shading(normal_eye_space, lit_sphere_map):
    """
    Shades a fragment using a lit sphere map.
    
    Args:
        normal_eye_space: Tuple (nx, ny, nz) representing the eye-space normal.
        lit_sphere_map: 2D array or texture representing the lit sphere map.
        
    Returns:
        Color value sampled from the lit sphere map.
    """
    nx, ny, nz = normal_eye_space
    
    # Convert nx and ny from [-1, 1] to [0, 1]
    # Note: nz is ignored as per the method definition [S1][S2, unofficial]
    u = (nx + 1.0) / 2.0
    v = (ny + 1.0) / 2.0
    
    # Sample the lit sphere map
    # The lit sphere map is a square texture where texels outside an inscribed disk 
    # are never accessed [S1][S2, unofficial].
    color = lit_sphere_map.sample(u, v)
    
    return color
```

### Edge Cases and Pitfalls
*   **Front/Back Face Ambiguity:** The $n_z$ component is ignored during indexing, meaning lighting does not distinguish between front and back faces [S1][S2, unofficial]. This is intentional because the gradient direction in the volume data (used as the normal) might be flipped depending on data values at a material boundary [S1][S2, unofficial].
*   **Texture Access:** Texels outside the inscribed disk of the square texture are never accessed [S1][S2, unofficial]. Implementers should ensure that sampling coordinates fall within the valid disk region or handle out-of-bounds access appropriately, though the method implies valid normals will always map within the disk.

## Subtopics

### Style Transfer Functions
Lit sphere shading is often used as a component of style transfer functions, which allow artists to define complex shading styles that can be applied to volume data [S1]. It enables the capture of artistic lighting variations as a function of normal direction, facilitating the creation of non-photorealistic volume renderings [S1].

## Operating characteristics

### Time and Memory Complexity
*   **Time Complexity:** $O(1)$ per fragment, as it involves a simple texture lookup after coordinate conversion [S1][S2, unofficial].
*   **Memory Complexity:** $O(N^2)$ for the lit sphere map, where $N$ is the texture resolution [S1][S2, unofficial].

### Measured Performance
Not given in the sources.

### Scaling
Not given in the sources.

### Strengths and Limitations
*   **Strengths:**
    *   Captures complex artistic lighting styles that are difficult to achieve with simple geometric light models [S1][S2, unofficial].
    *   View-dependent lighting effects appear regular and consistent with the camera movement [S1][S2, unofficial].
    *   Images of illuminated spheres are relatively easy to obtain or extract from existing artwork [S1][S2, unofficial].
    *   Allows for the rapid creation of a large database of shading styles [S1][S2, unofficial].
*   **Limitations:**
    *   Fails to capture complex aspects of realistic illumination, such as shadows, reflections, or indirect lighting [S1][S2, unofficial].
    *   Does not distinguish between front and back faces, which may be undesirable in some contexts (though intentional for volume data) [S1][S2, unofficial].

### When to Use
*   Use when the goal is to apply a specific artistic style to volume data that depends on surface orientation [S1][S2, unofficial].
*   Use when view-dependent lighting is desired without the computational cost of full ray tracing or complex light transport simulations [S1][S2, unofficial].

### When Not to Use
*   Do not use when realistic illumination is required [S1][S2, unofficial].
*   Do not use when front/back face distinction is critical for the visual interpretation of the data [S1][S2, unofficial].

## Background

### Origin
Lit sphere shading was originally presented by Sloan et al. [SMGG01] for non-photorealistic rendering of polygonal models [S1][S2, unofficial]. It was later adapted for illustrative volume rendering by [Style Transfer Functions for Illustrative Volume Rendering](https://doi.org/10.1111/j.1467-8659.2007.01095.x) [S1][S2, unofficial].

### History and Development
The technique leverages the property that a sphere under orthographic projection covers the complete set of unit normals on one hemisphere [S1][S2, unofficial]. This allows the sphere to act as a proxy for illumination, capturing color variations as a function of normal direction [S1][S2, unofficial]. Sloan et al. also described a method for extracting lit sphere maps from non-spherical regions in existing artwork, enabling the rapid creation of style databases [S1][S2, unofficial].

### Where it is Used
Lit sphere shading is used in illustrative volume rendering to enhance the visual clarity and artistic appeal of volume data [S1][S2, unofficial]. It is particularly useful in medical visualization and scientific visualization where clear distinction of structures and artistic presentation are important [S1][S2, unofficial].

## Sources

[S1] Style Transfer Functions for Illustrative Volume Rendering, https://doi.org/10.1111/j.1467-8659.2007.01095.x, fetched 2026-10-26
[S2] Style Transfer Functions for Illustrative Volume Rendering, https://www.researchgate.net/publication/227615609_Style_Transfer_Functions_for_Illustrative_Volume_Rendering, fetched 2026-10-27 (unofficial)
