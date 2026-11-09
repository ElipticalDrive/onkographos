# Order Independent Transparency for Point-Based Models

Also called: Depth-peeling for splats, Multi-pass splatting transparency

## Overview
This method renders transparent point-based models on GPUs in an order-independent manner by extending depth-peeling techniques to deferred-shaded multi-pass splatting [S1]. It solves the problem of correctly compositing translucent fragments without explicit depth sorting, which is computationally expensive or impossible to perform efficiently on the GPU for arbitrary fragment counts [S1]. Unlike standard alpha blending, which is order-dependent and produces incorrect results when fragments overlap in arbitrary orders, this method approximates the correct compositing result in a single or multi-pass pipeline [S1]. It distinguishes itself from siblings like [Stop-the-pop](1472-stop-the-pop.md) by using iterative layer reconstruction with deferred shading, and from [Weighted Blended Order-Independent Transparency](8931-weighted-blended-order-independent-transparency.md) by explicitly separating layers by depth rather than blending weights [S1].

## How it works
The algorithm consists of three main phases: opaque rendering, iterative transparent layer reconstruction, and final compositing.

**Inputs and Outputs:**
*   **Input:** Point-based model (set of splats with position, density, color, normal), camera parameters, number of transparent layers $n$ (typically 3 or 4) [S1].
*   **Output:** Final composited image [S1].

**Data Structures:**
*   **Depth Buffers:** A stack of depth textures. The first pass uses the standard depth buffer. Subsequent passes bind the previous layer's depth buffer as a read-only texture [S1].
*   **Attribute Textures:** Textures storing accumulated attributes (color, normal) for each layer [S1].
*   **Layer Textures:** Textures storing the final shaded color for each reconstructed layer [S1].

**Algorithm Steps:**

1.  **Opaque Rendering:**
    Render opaque objects using the standard three-pass splatting approach: visibility, attribute accumulation, and deferred shading [S1]. This establishes the background depth and attributes.

2.  **Iterative Transparent Layer Reconstruction (for $i = 1$ to $n$):**
    For each layer $i$, reconstruct the visible transparent fragments behind the previously reconstructed layers.

    *   **Initialization (Layer 1):**
        *   Bind the opaque objects' depth buffer as the reference depth texture [S1].
        *   Clear destination buffers (attributes, depth) [S1].

    *   **Initialization (Layer $i > 1$):**
        *   Bind the depth buffer of layer $i-1$ as a read-only depth texture [S1].
        *   Clear destination buffers (attributes, depth) [S1].

    *   **Visibility Splatting (Pass 1 of Layer $i$):**
        *   Rasterize all transparent splats.
        *   Perform an additional depth comparison: only fragments with depth greater than the depth of the previous layer (or opaque objects for $i=1$) pass the test [S1].
        *   This pass identifies which splats are visible at this depth layer [S1].

    *   **Attribute Accumulation (Pass 2 of Layer $i$):**
        *   Rasterize all transparent splats again.
        *   In the fragment shader, perform two additional depth comparisons:
            1.  Compare with the depth of the previous layer.
            2.  Compare with the depth of opaque objects.
        *   Only visible splats behind both the previous transparent layers and opaque objects pass these tests [S1].
        *   Accumulate attributes (e.g., color, normal) for passing splats [S1].

    *   **Normalization and Deferred Shading (Pass 3 of Layer $i$):**
        *   Normalize the accumulated attributes [S1].
        *   Apply deferred shading to compute the final color for this layer [S1].
        *   Store the result in a layer texture [S1].

    *   **Special Handling for Last Layer ($i = n$):**
        *   **Skip Visibility Splatting:** To reduce rendering time, skip the visibility splatting pass for the last layer [S1].
        *   **Modulate Weights:** Use transparency coefficients to modulate splatting weights, giving more importance to more opaque surfaces to diminish artifacts [S1].
        *   **Switch Lighting:** Switch from deferred shading to per-splat lighting for the last layer. Blending surface attributes from different surfaces in deferred shading makes no sense for the farthest layer, which contains small splats where per-pixel lighting quality is equivalent [S1].

3.  **Final Compositing:**
    *   Combine all reconstructed layers with the final image by drawing them with alpha-blending in the inverse order of their construction (from farthest layer $n$ to closest layer 1) [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_point_based_transparency(splats, camera, num_layers=3):
    # 1. Opaque Rendering
    # Render opaque objects using standard 3-pass splatting (visibility, attribute accumulation, deferred shading)
    opaque_depth_buffer = render_opaque_passes(splats, camera)
    
    # Initialize layer storage
    layer_textures = []
    layer_depth_buffers = []
    
    # 2. Iterative Transparent Layer Reconstruction
    for i in range(1, num_layers + 1):
        # Determine reference depth
        if i == 1:
            ref_depth_texture = opaque_depth_buffer
        else:
            ref_depth_texture = layer_depth_buffers[i-2] # Depth of previous layer
            
        # Clear destination buffers
        clear_buffers()
        
        # Visibility Splatting (Skip for last layer)
        if i < num_layers:
            render_visibility_splatting(splats, camera, ref_depth_texture)
            
        # Attribute Accumulation
        # Perform depth comparisons against ref_depth_texture and opaque objects
        accumulated_attributes = render_attribute_accumulation(splats, camera, ref_depth_texture, opaque_depth_buffer)
        
        # Normalization and Deferred Shading
        if i == num_layers:
            # Last layer: switch to per-splat lighting and modulate weights
            layer_color = render_last_layer_lighting(splats, camera, accumulated_attributes)
        else:
            # Normal layers: deferred shading
            normalized_attributes = normalize(accumulated_attributes)
            layer_color = deferred_shade(normalized_attributes)
            
        layer_textures.append(layer_color)
        layer_depth_buffers.append(get_current_depth_buffer())
        
    # 3. Final Compositing
    final_image = render_opaque_passes(splats, camera) # Start with opaque background
    for i in range(num_layers - 1, -1, -1): # From farthest to closest
        final_image = alpha_blend(final_image, layer_textures[i])
        
    return final_image
```

**Parameters:**
*   $n$: Number of transparent layers to reconstruct. Typically bounded to 3 or 4 for most scenes to balance efficiency and memory usage [S1].

**Edge Cases and Pitfalls:**
*   If the viewpoint contains more than $n$ visible transparent layers, color information of unreconstructed layers is lost [S1].
*   The heuristic of accumulating all visible transparent splats behind layer $n-1$ into layer $n$ can introduce artifacts, mitigated by modulating splatting weights with transparency coefficients [S1].

## Subtopics

### [Order Independent Transparency Approaches](3611-order-independent-transparency-approaches.md)
Provides the theoretical framework and specific algorithmic variations (such as weight-based approximation or closest-sample separation) used to handle non-commutative transparency accumulation. This method is a specific implementation of this family using depth-peeling for splats.

## Operating characteristics
*   **Time Complexity:** $O(n \cdot N)$, where $N$ is the number of splats and $n$ is the number of layers. Each layer requires multiple passes (visibility, attribute accumulation, shading) [S1].
*   **Memory Complexity:** $O(n \cdot W \cdot H)$, where $W$ and $H$ are image dimensions. Requires storing depth and color buffers for each layer [S1].
*   **Measured Performance:** On a GeForce 7800, rendering 27 transparent point-sampled spheres with 6 layers achieved 21 fps. With 3 layers, performance improved to 45 fps without heuristic and 49 fps with heuristic [S1].
*   **Strengths:** Order-independent transparency for point-based models; leverages GPU rasterization; deferred shading allows efficient lighting [S1].
*   **Limitations:** Limited by the number of layers $n$; artifacts if more than $n$ layers are visible; requires multiple passes [S1].
*   **When to use:** Real-time rendering of transparent point-based models where order-independent transparency is required and the number of overlapping transparent layers is limited [S1].
*   **When not to use:** Scenes with many overlapping transparent layers ($>n$); non-point-based models [S1].

## Background
This method was introduced in the paper "Splat/Mesh Blending, Perspective Rasterization and Transparency for Point-Based Rendering" [S1]. It extends depth-peeling techniques to deferred-shaded multi-pass splatting, addressing the limitations of standard alpha blending for point-based rendering [S1].

## Sources
[S1] Splat/Mesh Blending, Perspective Rasterization and Transparency for Point-Based Rendering, https://doi.org/10.2312/spbg/spbg06/049-057, fetched 2026-10-26
