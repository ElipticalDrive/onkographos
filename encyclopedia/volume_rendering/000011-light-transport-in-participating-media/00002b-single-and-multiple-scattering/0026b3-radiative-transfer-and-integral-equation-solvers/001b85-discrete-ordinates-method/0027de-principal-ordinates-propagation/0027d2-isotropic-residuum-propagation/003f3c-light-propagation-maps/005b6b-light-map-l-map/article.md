# Light Map (L Map)

Also called: L map

## Overview
The Light Map (L Map) is a perceptual culling map used in the rendering of participating media to identify regions around light sources where scattering effects are visually salient [S1]. It addresses the artifact of false undersampling caused by the Extinction-Saliency (XS) map, which incorrectly directs the rendering algorithm to skip detailed sampling in areas near light sources where multiple scattering is actually significant [S1]. Unlike the XS map, which combines extinction and saliency, the L map specifically isolates the spatial extent of light source influence to guide adaptive refinement [S1].

## How it works
The L Map is generated through a three-step process designed to be computationally inexpensive while accurately marking areas requiring high-fidelity scattering calculations [S1].

**Inputs:**
*   Scene geometry and light source positions.
*   Media optical thickness distribution.

**Outputs:**
*   A scalar map (the L Map) where non-zero values indicate areas around light sources where scattering is a salient feature.

**Data Structures:**
*   The L Map is typically stored as a 2D texture or grid aligned with the screen or voxel space, matching the resolution of the XS map or the primary rendering buffer. Values represent the intensity of the "influence" from light sources.

**Steps:**
1.  **Identify Light Sources:** Determine the positions and extents of all light sources in the scene [S1].
2.  **Generate 0-LightMap ($L_0$):** Render a scan-line image where light sources are rendered as constant white (value 1.0) and the rest of the scene is black (value 0.0) [S1]. This image is termed the 0-LightMap ($L_0$) [S1].
3.  **Apply Point Spread Function (PSF):** Convolve the white pixels of $L_0$ with a Point Spread Function to define the area around light sources where scattering becomes important [S1]. This step effectively blurs the sharp light source boundaries into a soft region of influence [S1].

**Formula:**
The final L Map value $L(x)$ at position $x$ is computed as:
$$ L(x) = (L_0 * \text{PSF})(x) $$
Where $*$ denotes convolution.

**Parameters:**
*   **PSF Shape:** The shape of the point spread function depends only on the optical thickness of the media [S1]. This simplification keeps the method computationally simple while producing good results [S1]. More accurate shapes could be derived from work such as Narasimhan and Nayar [2003], but are not required for the basic L Map definition [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_light_map(scene, media_optical_thickness):
    # Step 1: Identify light sources (implicit in rendering pass)
    
    # Step 2: Produce 0-LightMap (L0)
    # Render scene where light sources are white (1.0) and everything else is black (0.0)
    L0 = render_light_sources_as_white(scene)
    
    # Step 3: Apply Point Spread Function
    # The PSF shape depends only on the optical thickness of the media
    psf = create_psf_from_optical_thickness(media_optical_thickness)
    
    # Convolve L0 with the PSF to define the area where scattering is important
    L_map = convolve(L0, psf)
    
    return L_map
```

**Edge Cases and Pitfalls:**
*   **Complex PSF Shapes:** While the method assumes a PSF shape based solely on optical thickness for simplicity, ignoring more complex scattering profiles (e.g., those from Narasimhan and Nayar [2003]) may lead to suboptimal culling in highly anisotropic media [S1].
*   **Resolution Mismatch:** If the L Map resolution does not match the rendering resolution, artifacts may occur during the culling phase [G].

## Subtopics

### [XSL Map Selective Rendering](16896-xsl-map-selective-rendering.md)
A perceptual strategy that combines the L map with an Extinction map (X map) and a Saliency map (S map) to guide selective rendering, providing more accurate estimates in perceptually important areas while reducing rendering times.

## Operating characteristics
*   **Time Complexity:** The generation of the L Map is fast, involving a simple render pass and a convolution. The complexity is dominated by the PSF convolution, which is typically $O(N)$ for separable filters or $O(N \cdot k^2)$ for non-separable ones, where $N$ is the number of pixels and $k$ is the kernel size [S1].
*   **Memory Complexity:** Requires storage for one additional map (the L Map) of the same resolution as the XS map or the primary rendering buffer [S1].
*   **Strengths:** Effectively fixes artifacts caused by the XS map wrongly indicating undersampling near light sources [S1]. Computationally simple to generate [S1].
*   **Limitations:** The PSF shape is simplified to depend only on optical thickness, which may not capture all scattering nuances [S1].
*   **When to use:** Use when rendering participating media with significant multiple scattering near light sources and when the XS map alone produces visible artifacts around light sources [S1].
*   **When not to use:** Avoid if computational resources are extremely limited and the PSF convolution overhead is prohibitive, or if the media is optically thin enough that scattering near light sources is negligible [S1].

## Background
The Light Map was introduced in the context of perceptual rendering of participating media to address specific artifacts in adaptive rendering techniques [S1]. It was proposed as a novel solution to overcome the problem of the XS map incorrectly directing the rendering algorithm to undersample areas around light sources [S1]. The work is part of the broader effort to balance rendering quality and performance in physically based rendering of participating media [S1].

## Sources
[S1] Perceptual rendering of participating media, https://doi.org/10.1145/1278387.1278389, fetched 2026-10-30
