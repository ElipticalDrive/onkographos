# Order Independent Transparency for Point-Based Models

Also called: Depth-peeling for splats, Multi-pass splatting transparency

## Overview
This method renders transparent point-based models on GPUs in an order-independent manner by extending depth-peeling techniques to deferred-shaded multi-pass splatting [S1]. It reconstructs transparent layers iteratively from closest to farthest, avoiding the need for explicit per-pixel fragment sorting [S1]. It distinguishes itself from siblings like [Stop-the-pop](1472-stop-the-pop.md) by using iterative layer reconstruction with deferred shading [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Point-based model (set of splats with position, density, color, normal), camera parameters, number of transparent layers $n$ (typically 3 or 4) [S1].
*   **Output:** Final composited image [S1].

**Data Structures:**
*   **Depth Buffers:** A stack of depth textures. The first pass uses the standard depth buffer. Subsequent passes bind the previous layer's depth buffer as a read-only texture [S1].
*   **Attribute Textures:** Textures storing accumulated attributes (color, normal) for each layer [S1].
*   **Layer Textures:** Textures storing the final shaded color for each reconstructed layer [S1].

**Algorithm Steps:**

1.  **Opaque Rendering:**
    Render opaque objects using the standard three-pass splatting approach: visibility, attribute accumulation, and deferred shading [S1].

2.  **Iterative Transparent Layer Reconstruction (for $i = 1$ to $n$):**
    For each layer $i$, reconstruct the visible transparent fragments behind the previously reconstructed layers [S1].
    *   **Initialization:** Bind the previous depth buffer as a read-only depth texture (except for the first pass) [S1]. Clear the destination buffers (attributes, depth) [S1].
    *   **Visibility Splatting:** Perform an additional depth comparison with the depth of the previous layer; only fragments behind the previous layer pass the test [S1].
    *   **Attribute Accumulation:** Add two additional depth comparisons in the fragment shader: one with the depth of the previous layer and one with the depth of opaque objects; only visible splats behind the previous layers pass these tests [S1].
    *   **Normalization and Deferred Shading:** Normalize the accumulated attributes and apply deferred shading to compute the final color for this layer. Store the result in a layer texture [S1].
    *   **Special Handling for Last Layer ($i = n$):** Skip the visibility splatting pass to reduce rendering time [S1]. Modulate splatting weights with transparency coefficients so that more opaque surfaces have more importance to diminish artifacts [S1]. Switch from deferred shading to per-splat lighting because blending surface attributes from different surfaces makes no sense for the farthest layer, which contains small splats where per-pixel lighting quality is equivalent [S1].

3.  **Final Compositing:**
    Combine all reconstructed layers with the final image by drawing them with alpha-blending in the inverse order of their construction (from farthest to closest) [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_point_based_transparency(splats, camera, num_layers=3):
    # 1. Opaque Rendering: visibility, attribute accumulation, deferred shading
    opaque_depth = rasterize_opaque_splats(splats, camera)
    
    layer_colors = []
    layer_depths = []
    
    for i in range(1, num_layers + 1):
        ref_depth = opaque_depth if i == 1 else layer_depths[i-2]
        clear_render_targets()
        
        if i < num_layers:
            # Visibility splatting: discard fragments with depth <= ref_depth
            rasterize_visibility_splats(splats, camera, ref_depth)
            
        # Attribute accumulation: discard fragments with depth <= ref_depth or depth <= opaque_depth
        attrs = rasterize_attribute_splats(splats, camera, ref_depth, opaque_depth)
        
        if i == num_layers:
            # Last layer: modulate weights with transparency coefficients, switch to per-splat lighting
            color = compute_last_layer_lighting(splats, camera, attrs)
        else:
            norm_attrs = normalize(attrs)
            color = compute_deferred_shading(norm_attrs)
            
        layer_colors.append(color)
        layer_depths.append(read_current_depth_buffer())
        
    final_image = rasterize_opaque_splats(splats, camera)
    for i in range(num_layers - 1, -1, -1):
        final_image = alpha_composite(final_image, layer_colors[i])
        
    return final_image
```

**Parameters:**
*   $n$: Number of transparent layers to reconstruct. Typically bounded to 3 or 4 for most scenes to balance efficiency and memory usage [S1].

**Edge Cases and Pitfalls:**
*   If the viewpoint contains more than $n$ visible transparent layers, color information of unreconstructed layers is lost [S1].
*   The heuristic of accumulating all visible transparent splats behind layer $n-1$ into layer $n$ can introduce artifacts, mitigated by modulating splatting weights with transparency coefficients [S1].

## Subtopics

### [Order Independent Transparency Approaches](3611-order-independent-transparency-approaches.md)
Provides the theoretical framework and specific algorithmic variations used to handle non-commutative transparency accumulation. This method is a specific implementation of this family using depth-peeling for splats.

## Operating characteristics
*   **Time and Memory Complexity:** Not given in the sources.
*   **Measured Performance:** On a GeForce 7800, rendering 27 transparent point-sampled spheres with 6 layers achieved 21 fps. With 3 layers, performance was 45 fps without heuristic and 49 fps with heuristic [S1].
*   **Strengths:** Order-independent transparency for point-based models; leverages GPU rasterization; deferred shading allows efficient lighting [S1].
*   **Limitations:** Limited by the number of layers $n$; artifacts if more than $n$ layers are visible; requires multiple passes [S1].
*   **When to use:** Real-time rendering of transparent point-based models where order-independent transparency is required and the number of overlapping transparent layers is limited [S1].
*   **When not to use:** Scenes with many overlapping transparent layers ($>n$); non-point-based models [S1].

## Background
This method was introduced in the paper "Splat/Mesh Blending, Perspective Rasterization and Transparency for Point-Based Rendering" [S1]. It extends depth-peeling techniques to deferred-shaded multi-pass splatting, addressing the limitations of standard alpha blending for point-based rendering [S1].

## Sources
[S1] Splat/Mesh Blending, Perspective Rasterization and Transparency for Point-Based Rendering, https://doi.org/10.2312/spbg/spbg06/049-057, fetched 2026-10-26
