# Splat/Mesh Blending with Transition Smoothing

Also called: Hybrid Rendering Transition Smoothing, Hybrid rendering with transition smoothing, Splat-Polygon Blending

## Overview
This method solves the visual discontinuities that appear when rendering a surface using both polygonal meshes and point-based splats simultaneously [S1]. It achieves smooth transitions by reusing the natural weight falloff of the splat representation to linearly blend between the polygonal and splat rendering results [S1]. Unlike alternatives that rasterize additional geometry at the transition boundary, this approach integrates blending into the standard multi-pass splatting pipeline, avoiding increased rendering cost and complexity [S1].

## How it works
The method operates on a hybrid scene containing both polygonal meshes and splats [S1]. It requires preprocessing to identify transition edges and uses a four-pass rendering pipeline to ensure correct occlusion and smooth color blending [S1].

### Inputs and Outputs
*   **Inputs:** A set of splats (defined by position, orientation, scale, and attributes) and a set of polygons (vertices, indices, and attributes) [S1]. A small depth bias value $\varepsilon$ [S1].
*   **Outputs:** A final image with smooth transitions between splat-only, polygon-only, and blended regions [S1].

### Data Structures
*   **Transition List:** A list of shared splats. Each element stores the list of splat indices shared by two different leaves (in multiresolution contexts) or simply the splats located on transition edges [S1].
*   **Octree Node (Multiresolution variant):** Stores a list of splat indices [S1].
*   **Octree Leaf (Multiresolution variant):** Stores a list of polygon indices [S1].
*   **Render Targets:**
    *   Depth Buffer: For visibility testing [G].
    *   Polygon Texture: Stores the rasterized polygonal result (color and alpha) [S1].
    *   Attribute Buffers: Accumulate splat attributes (color, normal, etc.) [S1].

### Preprocessing
1.  **Transition Edge Sampling:** Identify edges shared by polygons held by different leaves (in multiresolution) or simply transition boundaries. Uniformly sample these edges such that splat centers coincide with the edge points [S1].
2.  **Splat Removal and Indexing:** Remove splats that are too close to these transition edges to avoid oversampling. Add the splats centered on the edges to the **Transition List** [S1].

### Rendering Pipeline
The rendering consists of four passes. Let $W_{splat}(x,y)$ be the sum of weights of all splats covering pixel $(x,y)$ [S1]. Let $C_{splat}(x,y)$ be the accumulated color from splats [S1]. Let $C_{poly}(x,y)$ and $\alpha_{poly}(x,y)$ be the color and alpha from the polygon rasterization [S1].

**Pass 1: Visibility Splatting**
*   Draw all splats as opaque ellipses into the depth buffer [S1].
*   Shift the depth values of splats by $\varepsilon$ along the view direction [S1].

**Pass 2: Polygon Rasterization**
*   Render polygons onto the depth buffer established in Pass 1 [S1].
*   Polygonal regions hidden by splats are removed [S1].
*   Update the depth buffer so that splat parts hidden by polygons are removed in subsequent passes [S1].
*   Render polygons into a texture to store the result for later blending [S1].

**Pass 3: Attribute Accumulation**
*   Accumulate splat attributes (color, normals, etc.) into attribute buffers [S1].
*   Compare the depth of splats and polygons with a tolerance to allow overlapping [S1].
*   Shift the depth values of splats by $\varepsilon$ toward the viewer [S1].

**Pass 4: Finalization (Blending and Shading)**
*   Normalize the accumulated splat attributes [S1].
*   Clamp the sum of splat weights $W_{splat}(x,y)$ to the range $[0, 1]$ [S1].
*   Because it is difficult to ensure that the sum of weights saturates for pixels only covered by splats, blending must be done if and only if the pixel is covered by a polygon [S1].
*   Check the alpha component $\alpha_{poly}(x,y)$ of the polygon rendered image to verify polygon coverage [S1].
*   If $\alpha_{poly}(x,y) > 0$, linearly blend the splat color $C_{splat}(x,y)$ and polygon color $C_{poly}(x,y)$ using $W_{splat}(x,y)$ as the factor:
    $$ C_{final}(x,y) = W_{splat}(x,y) \cdot C_{splat}(x,y) + (1 - W_{splat}(x,y)) \cdot C_{poly}(x,y) $$
*   If $\alpha_{poly}(x,y) = 0$, use $C_{splat}(x,y)$ directly [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_hybrid(splats, polygons, epsilon):
    # Initialize buffers [G]
    depth_buffer = [[float('inf')] for _ in range(height)]
    polygon_texture = [[None for _ in range(width)] for _ in range(height)]
    attr_color = [[0.0, 0.0, 0.0] for _ in range(height)]
    attr_weight = [[0.0] for _ in range(height)]

    # Pass 1: Visibility Splatting
    for splat in splats:
        # Project splat to screen footprint [G]
        for y, x in splat.screen_footprint():
            w = splat.weight_at(x, y)
            if w > 0:
                shifted_depth = splat.depth + epsilon
                if shifted_depth < depth_buffer[y][x]:
                    depth_buffer[y][x] = shifted_depth

    # Pass 2: Polygon Rasterization
    for poly in polygons:
        # Rasterize polygon, respecting depth buffer [G]
        for y, x in poly.screen_projection():
            if poly.depth >= depth_buffer[y][x]:
                continue
            polygon_texture[y][x] = (poly.color, 1.0)
            depth_buffer[y][x] = poly.depth

    # Pass 3: Attribute Accumulation
    for splat in splats:
        # Project splat to screen footprint [G]
        for y, x in splat.screen_footprint():
            w = splat.weight_at(x, y)
            if w > 0:
                shifted_depth = splat.depth - epsilon
                # Allow overlapping by checking depth difference [G]
                if abs(shifted_depth - depth_buffer[y][x]) <= epsilon:
                    attr_color[y][x] += splat.color * w
                    attr_weight[y][x] += w

    # Pass 4: Finalization
    final_image = [[None for _ in range(width)] for _ in range(height)]
    for y in range(height):
        for x in range(width):
            if polygon_texture[y][x] is None:
                final_image[y][x] = attr_color[y][x]
                continue
            w = min(max(attr_weight[y][x], 0.0), 1.0)
            c_splat = attr_color[y][x] / max(attr_weight[y][x], 1e-6)
            c_poly = polygon_texture[y][x][0]
            final_image[y][x] = w * c_splat + (1.0 - w) * c_poly
    return final_image
```

### Parameters
*   $\varepsilon$: A small positive value used to shift splat depth values. In Pass 1, it is added along the view direction. In Pass 3, it is subtracted (shifted toward the viewer) to handle depth comparisons and occlusion correctly [S1].

### Edge Cases and Pitfalls
*   **Non-saturating Weights:** If the sum of weights does not saturate to 1 for pixels covered only by splats, incorrect blending may occur. This is mitigated by checking the alpha component of the polygon image to ensure blending only happens where polygons are present [S1].
*   **Multiresolution Consistency:** In multiresolution rendering, splats located exactly on an edge must be drawn only once if at least one of the polygons sharing the edge is drawn as a set of splats. The Transition List ensures this consistency [S1].
*   **Sampling Quality:** The quality of smoothing depends on point sampling at the transition. Uniform sampling such that splat centers coincide with edges is required for the weight sum to saturate correctly [S1].

## Subtopics

### Multiresolution Hybrid Rendering with Octree
A hybrid multi-resolution rendering system based on an octree that switches between splat and polygon representations based on sampling density [S1].
*   **What it is:** An extension of the blending method to hierarchical data structures. Nodes store splat indices; leaves store polygon indices [S1].
*   **What sets it apart:** It explicitly manages transition splats across different levels of detail. Edges shared by polygons in different leaves are explicitly sampled, and splats too close to these edges are removed to avoid oversampling [S1].
*   **When to choose it:** When rendering large datasets with varying density where some regions are better represented by polygons (under-sampled) and others by splats [S1].
*   **Link:** [Splat/Mesh Blending with Transition Smoothing](1181-splat-mesh-blending-with-transition-smoothing.md)

## Operating characteristics
*   **Time Complexity:** The method adds minimal overhead to standard splatting. It requires four passes, but the polygon rasterization pass is standard. The blending step is a simple per-pixel operation [S1].
*   **Memory Complexity:** Requires storage for the polygon texture, attribute buffers, and the transition list. The transition list size depends on the number of transition edges [S1].
*   **Strengths:** Avoids rasterizing additional geometries at the transition level, keeping rendering cost and complexity low. Provides high-quality visual transitions [S1].
*   **Limitations:** Requires preprocessing to identify and sample transition edges. Blending quality depends on the sampling density at the transition [S1].
*   **When to use:** When a hybrid representation (polygons for sparse regions, splats for dense regions) is necessary and visual continuity is critical [S1].
*   **When not to use:** When the scene is purely polygonal or purely splatted, as the overhead is unnecessary [S1].

## Background
This method was introduced in the context of point-based rendering to address the visual artifacts caused by mixing polygonal and point-based representations [S1]. It builds on the principles of splatting, specifically the property that the sum of splat weights varies from 1 to 0 at the boundary of the splat representation [S1]. The technique was developed to enable efficient hybrid rendering systems, particularly in multiresolution contexts where different parts of the scene are represented at different levels of detail [S1].

## Sources
[S1] Splat/Mesh Blending, Perspective Rasterization and Transparency for Point-Based Rendering, https://doi.org/10.2312/spbg/spbg06/049-057, fetched 2026-10-26
