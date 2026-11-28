# Splat/Mesh Blending with Transition Smoothing

Also called: Hybrid Rendering Transition Smoothing, Hybrid rendering with transition smoothing, Splat-Polygon Blending

## Overview
Splat/Mesh Blending with Transition Smoothing is a hybrid rendering technique that eliminates visual discontinuities at the boundary between polygonal mesh representations and point-based splat representations [S1]. It solves the problem of jagged or missing geometry when switching between these two paradigms by linearly interpolating between the two rendering results using the splat weight sum as a blending factor [S1]. Unlike naive hybrid approaches that require rasterizing additional transition geometries, this method reuses the natural weight falloff of the splat representation to achieve smooth blending with minimal complexity [S1].

## How it works
The method operates as a multi-pass pipeline that integrates polygon rasterization into a standard splatting framework [S1]. It requires careful handling of depth buffers and attribute accumulation to ensure correct occlusion and smooth color blending [S1].

### Inputs and Outputs
*   **Inputs:** A set of splats (point-based primitives with position, normal, radius, and attributes) and a set of polygons (mesh geometry with vertices, normals, and attributes) [S1]. A small depth shift parameter $\epsilon$ [S1].
*   **Outputs:** A final image with smooth transitions between splat and polygon regions, handling both opaque and transparent surfaces [S1].

### Data Structures
*   **Depth Buffer:** Standard depth buffer storing depth values, modified by shifting splat depths by $\epsilon$ [S1].
*   **Attribute Buffers:** Buffers storing accumulated splat attributes (color, normal, etc.) [S1].
*   **Polygon Texture:** A texture storing the rasterized result of the polygon pass to be reused in the finalization step [S1].
*   **Transition List (Multiresolution only):** A list where each element stores the indices of splats shared by two different octree leaves [S1].

### Preprocessing (Sampling)
To ensure the sum of splat weights saturates at transition edges, the transition edges must be uniformly sampled such that splats share their center with the edge [S1]. This is done during the sampling step or in a preprocessing step [S1]. In multiresolution rendering, edges shared by two polygons held by two different leaves are explicitly sampled [S1]. Splats too close to such edges are removed to avoid oversampling [S1]. Shared splats are indexed in a separate list [S1].

### Rendering Steps
The rendering process consists of four passes:

1.  **Visibility Splatting:** Splats are drawn as opaque ellipses into the depth buffer. The depth values of splats are shifted by $\epsilon$ along the view direction [S1].
2.  **Polygon Rasterization:** Polygons are rendered upon the previous depth buffer. Polygonal regions hidden by splats are removed. The depth buffer is updated so that splat parts hidden by polygons are removed in the next pass. Polygons are rendered into a texture to reuse the result [S1].
3.  **Attribute Accumulation:** Splat attributes are accumulated into attribute buffers. Depth of splats and polygons is compared with a tolerance to allow overlapping. Depth values of splats are shifted by $\epsilon$ toward the viewer [S1].
4.  **Finalization:** During normalization and deferred shading, the sum of weights of the splatting is clamped to $[0,1]$ and used to linearly blend the colors from splatting and polygon rasterization. Blending occurs if and only if the pixel is covered by a polygon, checked via the alpha component of the polygon rendered image [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
epsilon = 1e-6  # Small value for depth shifting [S1]
tolerance = 1e-4  # Depth comparison tolerance for overlapping [G]
height, width = 1024, 1024  # Image dimensions [G]
output_image = [[None] * width for _ in range(height)]
depth_buffer = [[float('inf')] * width for _ in range(height)]
attribute_buffers = [[{'color': (0.0, 0.0, 0.0), 'weight_sum': 0.0} for _ in range(width)] for _ in range(height)]
polygon_texture = [[{'rgb': (0.0, 0.0, 0.0), 'a': 0.0} for _ in range(width)] for _ in range(height)]

def render_hybrid_splat_mesh(splats, polygons):
    # Pass 1: Visibility Splatting
    for splat in splats:
        shifted_depth = splat.depth + epsilon
        for pixel in splat.covered_pixels:
            if shifted_depth < depth_buffer[pixel[0]][pixel[1]]:
                depth_buffer[pixel[0]][pixel[1]] = shifted_depth

    # Pass 2: Polygon Rasterization
    for polygon in polygons:
        for pixel in polygon.covered_pixels:
            if polygon.depth < depth_buffer[pixel[0]][pixel[1]]:
                depth_buffer[pixel[0]][pixel[1]] = polygon.depth
                polygon_texture[pixel[0]][pixel[1]] = {'rgb': polygon.color, 'a': 1.0}

    # Pass 3: Attribute Accumulation
    for splat in splats:
        shifted_depth = splat.depth - epsilon
        for pixel in splat.covered_pixels:
            if abs(depth_buffer[pixel[0]][pixel[1]] - shifted_depth) < tolerance:
                attr = attribute_buffers[pixel[0]][pixel[1]]
                attr['color'] = splat.attributes
                attr['weight_sum'] += splat.weight

    # Pass 4: Finalization
    for y in range(height):
        for x in range(width):
            splat_color = attribute_buffers[y][x]['color']
            polygon_color = polygon_texture[y][x]['rgb']
            splat_weight_sum = attribute_buffers[y][x]['weight_sum']
            polygon_alpha = polygon_texture[y][x]['a']

            w = max(0.0, min(1.0, splat_weight_sum))

            if polygon_alpha > 0.0:
                final_color = tuple(p + w * (s - p) for p, s in zip(polygon_color, splat_color))
            else:
                final_color = splat_color

            output_image[y][x] = final_color
```

### Parameters
*   $\epsilon$: A small value used to shift splat depth values along the view direction in Pass 1 and toward the viewer in Pass 3 to handle depth comparisons and occlusion correctly [S1].

### Edge Cases and Pitfalls
*   **Incorrect Blending for Splat-Only Pixels:** If the sum of weights does not saturate for pixels only covered by splats, incorrect blending occurs. This is handled by checking the alpha component of the polygon rendered image to ensure blending only happens where polygons are present [S1].
*   **Multiresolution Edge Handling:** In multiresolution rendering, splats located exactly on an edge must be drawn once if at least one of the polygons sharing the edge is drawn as a set of splats [S1].
*   **Weight Saturation:** The quality of smoothing depends on point sampling at the transition. If the sum of weights does not saturate at transition edges, the blending factor will be incorrect [S1].

## Subtopics

### Multiresolution Hybrid Rendering with Octree
A hybrid multi-resolution rendering system based on an octree that switches between splat and polygon representations based on sampling density and manages transition splats explicitly [S1]. It is chosen when rendering large datasets with varying density where some regions are better represented by polygons (under-sampled) and others by splats [S1]. The octree node stores a list of splat indices, while leaves store a list of polygon indices [S1]. During preprocessing, edges shared by two polygons held by two different leaves are explicitly sampled, and splats too close to such edges are removed [S1]. Shared splats are indexed in a separate list [S1]. At rendering time, the octree is traversed with per-node visibility culling and level-of-details estimation [S1]. If a leaf is under-sampled, the polygonal representation is selected [S1]. At the end of the tree traversal, each element of the transition list is checked: if at least one of the leaves shared by the transition is drawn as a set of splats, then the list of splats held by the transition are also drawn [S1].

## Operating characteristics
*   **Time Complexity:** The method adds minimal overhead to standard splatting, as it avoids rasterizing additional geometries at the transition level [S1].
*   **Memory Complexity:** Requires additional storage for the polygon texture and attribute buffers [S1]. In multiresolution rendering, additional storage for the transition list is needed [S1].
*   **Measured Performance:** Not given in the sources.
*   **Strengths:** Avoids the increased rendering cost and complexity of rasterizing additional transition geometries [S1]. Provides high-quality smooth transitions between splat and polygon representations [S1].
*   **Limitations:** Blending quality depends on the sampling of transition edges [S1]. The method blends resulting shaded colors, which may be less accurate than blending attributes before shading [S1].
*   **When to Use:** When rendering point-based models with hybrid polygonal representations, especially in multiresolution contexts where sampling density varies [S1].
*   **When Not to Use:** When high-precision attribute blending is required before shading, as this method blends shaded colors [S1].

## Background
The method was introduced in the context of point-based rendering, specifically to address the visual discontinuities that arise when mixing splat and polygon representations [S1]. It is part of a broader effort to enhance splatting pipelines through hybridization, leveraging the efficiency of splatting and the accuracy of polygonal rasterization [S1]. The technique was developed as part of a hybrid multi-resolution rendering system based on an octree [S1].

## Sources
[S1] Splat/Mesh Blending, Perspective Rasterization and Transparency for Point-Based Rendering, https://doi.org/10.2312/spbg/spbg06/049-057, fetched 2026-10-26
