# Pre-integrated Clipping Boundary Handling


## Overview
A method to handle artifacts at clip boundaries when using pre-integrated volume rendering with low sampling rates, by mapping an opaque texture onto the clip geometry [S1, unofficial]. It sets apart from [Clipping](1444-clipping.md) and [Clipped Cropping Region Planes](2931-clipped-cropping-region-planes.md) by explicitly computing the optical contribution of the gap between the clipping surface and the first volume slice, rather than relying solely on depth buffering or simple suppression [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: Clipping surface geometry (proxy polygons), volume texture, viewing ray, fragment shader program.
- Output: Correctly composited color and opacity at the clip boundary, avoiding artifacts caused by discrete sampling gaps.

**Data Structures:**
- Clipping Surface: Defined by proxy polygons. Each polygon has vertices with 3D coordinates.
- Volume Texture: 3D texture storing scalar values $v$.
- Fragment Shader State: Current fragment position $P$ on the clipping surface, and the intersection point $Q$ on the first proxy polygon behind $P$.

**Numbered Steps:**
1. Render the clipping surface before the clipped volume behind it.
2. For each fragment $P$ on the clipping surface, find the intersection $Q$ of the viewing ray with the first proxy polygon behind $P$ [S1, unofficial].
3. Look up data values $v_P$ at $P$ and $v_Q$ at $Q$ from the volume texture [S1, unofficial].
4. Compute the corresponding sampling distance $d_{\Delta}$ [S1, unofficial].
5. Compute the contribution of the ray segment between the surface and the first slice in the clipped volume in a fragment program [S1, unofficial].
6. Note: For consistency, do not compute the actual length of the ray segment between $P$ and $Q$ if it deviates from the pre-integrated model assumptions [S1, unofficial].

**Formulas:**
- Sampling distance: $d_{\Delta}$ is computed based on $v_P$ and $v_Q$. The exact formula for $d_{\Delta}$ is not given in the sources, but it is used to determine the contribution of the ray segment [S1, unofficial].
- Contribution: The contribution of the ray segment between $P$ and $Q$ is computed in the fragment program. The exact formula is not given in the sources [S1, unofficial].

**Parameters:**
- Typical values: Not given in the sources.

**Edge Cases and Pitfalls:**
- Precision: The method relies on accurate intersection detection between the viewing ray and the proxy polygons [S1, unofficial].
- Boundaries: The method assumes that the clipping surface is rendered before the volume [S1, unofficial].
- Degenerate Input: If the proxy polygon behind $P$ is not found, the method may fail [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def pre_integrated_clipping_boundary_handling(clipping_surface, volume_texture, viewing_ray):
    # Render clipping surface first
    for fragment in clipping_surface.fragments:
        P = fragment.position
        # Find intersection Q with first proxy polygon behind P
        Q = find_intersection(viewing_ray, P, clipping_surface.proxy_polygons)
        if Q is None:
            # Handle case where no proxy polygon is found
            continue
        # Look up data values
        v_P = volume_texture.sample(P)
        v_Q = volume_texture.sample(Q)
        # Compute sampling distance
        d_delta = compute_sampling_distance(v_P, v_Q)
        # Compute contribution of ray segment
        contribution = compute_contribution(d_delta)
        # Set fragment color and opacity
        fragment.color = contribution
        fragment.opacity = 1.0 # Opaque texture
```

## Subtopics

## Operating characteristics
- Time Complexity: Not given in the sources.
- Memory Complexity: Not given in the sources.
- Measured Performance: Not given in the sources.
- Scaling: Not given in the sources.
- Strengths: Handles artifacts at clip boundaries effectively [S1, unofficial].
- Limitations: Requires interleaved rendering of volume and clip surface, which is more complex [S1, unofficial].
- When to Use: When using pre-integrated volume rendering with low sampling rates [S1, unofficial].
- When Not to Use: When the goal is to show relationships between features, as opaque textures might obscure them if not handled carefully [S1, unofficial].

## Background
Origin: Not given in the sources.
History and Development: Not given in the sources.
Where it is Used: Not given in the sources.

## Sources
[S1] PDF A Focus and Context Interface for Interactive Volume Rendering, https://www.sci.utah.edu/~ikits/publications/Ikits_Unpublished_04.pdf, fetched 2026-10-27 (unofficial)
