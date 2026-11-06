# Volumetric Toon Shading

Also called: Fragment Shader Toon-Shading for Volumes, Interactive volume cartoon shading, Toon-Shading for Volumes, Toon-shading via 3D normal map

## Overview
Volumetric Toon Shading is a non-photorealistic rendering technique that simulates cel shading for volumetric datasets without requiring explicit surface vertices [S1, unofficial]. It differs from polygonal toon-shading by operating entirely on a per-fragment basis using a 3D normal map and hardware texture combiners, decoupling the illumination model from physical light transport to produce discrete, stylized tones [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** A 3D scalar field $f(x,y,z)$; light direction vector $L$; user-defined tone colors for illuminated and shadowed regions.
*   **Output:** Final color $c$ for each fragment.

**Data Structures**
*   **3D Normal Map:** A 3D texture storing the gray-scale gradients of the volume data in its RGB components. Each texel $(x,y,z)$ contains the normalized gradient vector $\nabla f_n$ at that position [S1, unofficial].

**Steps**
1.  **Generate Normal Map:** Compute the gradient $\nabla f$ for the volume data. Normalize it to obtain $\nabla f_n$. Store $\nabla f_n$ in the RGB channels of a 3D texture [S1, unofficial].
2.  **Lookup Normal:** For each fragment, sample the 3D normal map at the fragment's 3D position to retrieve the normal vector $N$ [S1, unofficial].
3.  **Compute Dot Product:** Calculate the dot product between the light direction $L$ and the normal $N$:
    $$ d = L \cdot N $$
    where $L$ is treated as a constant color vector in the texture combiner stage [S1, unofficial].
4.  **Thresholding:** Use the value $d$ to trigger a multiplexer in the texture combiner stage. The threshold is fixed at $0.5$ [S1, unofficial].
    *   If $d > 0.5$, select the "illuminated" tone register.
    *   If $d \leq 0.5$, select the "shadowed" tone register.
5.  **Output Color:** The selected register's color is output as the fragment's final color $c$ [S1, unofficial].

**Parameters**
*   **Threshold:** The multiplexer threshold is $0.5$ [S1, unofficial].
*   **Offset:** An offset can be added to the input $d$ before thresholding to shift the boundary between light and shadow, enabling arbitrary mapping of shades to tones [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode for Volumetric Toon Shading
# Assumes access to a 3D texture 'normal_map' storing normalized gradients
# and a constant light vector 'L'

def volumetric_toon_shading(fragment_position_3d, light_direction_L, tone_light, tone_shadow):
    # Step 1: Lookup the normal from the 3D normal map
    # The normal map stores the normalized gradient of the volume scalar field
    N = normal_map.sample(fragment_position_3d)
    
    # Step 2: Compute the dot product between light direction and normal
    # L is assumed to be normalized
    dot_product = dot(L, N)
    
    # Step 3: Apply threshold (0.5) to determine shading region
    # An offset can be added to dot_product here for arbitrary mapping
    threshold = 0.5
    
    if dot_product > threshold:
        final_color = tone_light
    else:
        final_color = tone_shadow
        
    return final_color
```

**Edge Cases and Pitfalls**
*   **Precision:** The binary nature of the thresholding ($> 0.5$) means small variations in the dot product near the threshold do not affect the output, creating hard edges. This is intentional for the toon effect but may appear aliased if the normal map resolution is low [S1, unofficial].
*   **Boundary Conditions:** The method relies on the existence of gradients. In regions where the volume scalar field is constant, the gradient is zero, resulting in a normal of $(0,0,0)$. The dot product will be $0$, which is $\leq 0.5$, causing these regions to always render as shadowed unless an offset is used [S1, unofficial].

## Subtopics

### Silhouette Extraction
*   **What it is:** A related technique using the same fragment-shader-based approach to render object silhouettes.
*   **Difference:** While toon shading modulates color based on light direction, silhouette extraction identifies edges where the view direction changes relative to the surface normal.
*   **When to choose:** When the goal is to emphasize object boundaries rather than surface lighting.
*   **Link:** [Silhouette Extraction](silhouette-extraction.md)

## Operating characteristics
*   **Time Complexity:** $O(1)$ per fragment, as it involves a single texture lookup and a dot product.
*   **Memory Complexity:** $O(V)$ for the 3D normal map, where $V$ is the volume resolution.
*   **Strengths:** No need for explicit surface extraction or mesh generation; efficient on hardware supporting multitexturing and fragment shaders; interactive frame rates achievable [S1, unofficial].
*   **Limitations:** The binary thresholding can lead to aliasing artifacts; the method is specific to the hardware architecture described (texture combiners) and may require adaptation for modern shader pipelines [S1, unofficial].
*   **When to use:** Interactive visualization of volumetric data where stylized, non-photorealistic appearance is desired and performance is critical [S1, unofficial].

## Background
The method was introduced in the context of interactive volume illustration, aiming to create toon-shaded images from volumes without special markup [S1, unofficial]. It adapts the general idea of toon-shading polygonal objects, where the dot product $L \cdot N$ is used as a texture coordinate, to the volume rendering domain by utilizing a 3D normal map [S1, unofficial]. The approach was developed to allow efficient rendering using fragment shader hardware and multitextures [S1, unofficial].

## Sources
[S1] Interactive volume illustration, https://openalex.org/W2147354080, fetched 2026-10-27 (unofficial)
