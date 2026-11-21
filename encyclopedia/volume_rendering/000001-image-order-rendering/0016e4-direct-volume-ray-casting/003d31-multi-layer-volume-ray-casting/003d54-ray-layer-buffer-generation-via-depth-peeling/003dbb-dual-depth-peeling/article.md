# Dual Depth Peeling

Also called: Bidirectional Depth Peeling, Depth Peeling Rendering Pipeline, Depth peeling loop, On-the-fly front-to-back peeling

## Overview
Dual Depth Peeling is an order-independent transparency (OIT) technique that decomposes a scene into depth layers by rendering primitives multiple times, discarding fragments closer than the previous layer, and blending the resulting layers to assemble the final image [S1]. It improves upon standard Depth Peeling by performing simultaneous peeling in both directions—front-to-back and back-to-front—thereby reducing the total number of rendering passes required for $N$ layers by approximately $N/2$ [S2]. Unlike standard Depth Peeling, which processes layers sequentially, this method resolves depth ordering per pixel, making it robust against self-intersecting surfaces that often complicate depth-sorting approaches [S1].

## How it works
The method executes a depth peeling loop for each frame, consisting of iterative peel and blend stages [S1].

**Inputs and Outputs:**
*   **Input:** Discrete representation of integral surfaces (e.g., triangles or quadrilaterals) with associated texture coordinates and normal vectors [S1].
*   **Output:** Final composited image with correct transparency and silhouette shading [S1].

**Data Structures:**
1.  **Color Buffer:** A floating-point color buffer is used for blending to prevent banding artifacts when many surface layers are present [S1].
2.  **Normal Buffer:** A secondary floating-point RGBA buffer stores surface normals required for the blend stage [S1].
3.  **Mask Buffer:** The alpha component of the normal buffer acts as a mask to distinguish integral surface pixels from context/background pixels [S1].
4.  **Depth Texture:** Stores the depth information of the current surface layer, required for the depth peeling algorithm's occlusion tests [S1].

**Parameters:**
*   **Silhouette Color:** User-specified color for fading surface edges [S1].
*   **Lighting Model:** Phong or Gooch models for diffuse color evaluation [S1].
*   **Modulation Type:** Multiplicative or additive transparency modulation [S1].

**Numbered Steps:**

1.  **Initialize:** Set up the floating-point color buffer, normal buffer, and depth texture. Initialize the depth peeling loop.
2.  **Peel Stage (Iterative):**
    a.  **Depth Test:** Render the primitives using a depth test that discards fragments closer than the depth stored in the depth texture from the previous iteration. For the first iteration, this is typically the near plane or a standard depth buffer [S1, S2].
    b.  **Shading:** Perform full surface shading for the visible fragments:
        i.  **Transparency:** Compute the view-dependent transparency term $\alpha_{view}$ from the fragment's normal vector. If normal-variation transparency is used, evaluate partial derivatives using `dFdx` and `dFdy` [S1].
        ii. **Pattern/Texture:** If adaptive patterns are used, compute $l_s$ and $l_t$ using `dFdx` and `dFdy`, and evaluate the pattern equation (Equation 5 in [S1]) to obtain texture color $c_{tex}$ and texture alpha $\alpha_{tex}$ [S1].
        iii. **Lighting:** Evaluate diffuse color $c_{diffuse}$ using the specified lighting model (Phong or Gooch). Compute specular highlight terms $(c, \alpha)_{spec}$ if specified [S1].
        iv. **Combination:** Compute the final RGBA output $(c, \alpha)_{peel}$ for the peel pass.
            *   For multiplicative modulation:
                $$ \alpha_{final} = \alpha_{view} \cdot \alpha_{tex} + \alpha_{specular} $$
                $$ c_{final} = c_{diffuse} \cdot c_{tex} + c_{specular} $$
            *   For additive modulation:
                $$ \alpha_{final} = \alpha_{view} + \alpha_{tex} + \alpha_{specular} $$
                (Note: $c_{final}$ remains $c_{diffuse} \cdot c_{tex} + c_{specular}$) [S1].
    c.  **Storage:** Write the final RGBA values to the color buffer. Store surface normals in the normal buffer, setting the alpha component to a mask flag indicating the pixel belongs to the integral surface [S1].
    d.  **Depth Update:** Update the depth texture with the depth values of the current layer for the next iteration [S1].
3.  **Blend Stage:**
    a.  **Silhouette Detection:** Apply a Sobel edge detection filter to both the normal buffer and the depth buffer to determine silhouette strength [S1].
    b.  **Masking:** Use the mask flag in the normal buffer to exclude pixels that have the flag set for one of the pixels contributing to the filter, avoiding silhouette generation across surface-context and context-background boundaries [S1].
    c.  **Silhouette Blending:** Smoothly fade the surface color into the user-specified silhouette color based on the computed silhouette strength [S1].
    d.  **Accumulation:** Blend the result into the final floating-point color buffer [S1].
4.  **Termination:** Adaptively terminate the peeling loop if no pixels are rendered during the peel stage, determined using an occlusion query for each iteration [S1]. The total number of iterations is one greater than the number of surface layers for the current viewpoint [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def dual_depth_peeling(primitives, silhouette_color, lighting_model, modulation_type):
    # Initialize buffers
    color_buffer = create_floating_point_buffer()
    normal_buffer = create_floating_point_buffer() # RGBA, A is mask
    depth_texture = create_depth_texture()
    
    # Initial depth for first pass (e.g., near plane)
    current_depth = NEAR_PLANE
    
    # Occlusion query for adaptive termination
    query = create_occlusion_query()
    
    while True:
        # Start Occlusion Query
        begin_occlusion_query(query)
        
        # PEEL STAGE
        # Render primitives with depth test against current_depth
        # Discard fragments with depth < current_depth
        render_primitives_with_depth_test(primitives, current_depth)
        
        # Shading in Vertex/Fragment Shader
        # 1. Transparency
        alpha_view = compute_alpha_view(normal)
        if normal_variation_transparency:
            alpha_view = evaluate_normal_variation(alpha_view, dFdx, dFdy)
            
        # 2. Pattern/Texture
        if adaptive_pattern:
            ls, lt = compute_pattern_coords(dFdx, dFdy)
            c_tex, alpha_tex = evaluate_pattern(ls, lt)
        else:
            c_tex, alpha_tex = get_texture_color_alpha()
            
        # 3. Lighting
        c_diffuse = evaluate_lighting(normal, position, lighting_model)
        c_spec, alpha_spec = evaluate_specular(normal, position, view_dir)
        
        # 4. Combination
        if modulation_type == ADDITIVE:
            alpha_final = alpha_view + alpha_tex + alpha_spec
        else: # Multiplicative
            alpha_final = alpha_view * alpha_tex + alpha_spec
            
        c_final = c_diffuse * c_tex + c_spec
        
        # Write to buffers
        write_to_color_buffer(c_final, alpha_final)
        write_to_normal_buffer(normal, mask_flag=1)
        
        # Update depth texture for next iteration
        update_depth_texture(current_depth)
        
        # End Occlusion Query
        end_occlusion_query(query)
        
        # Check if any pixels were rendered
        if not query_has_results(query) or not query_is_positive(query):
            break
            
        # Update current_depth for next pass
        # In Dual Depth Peeling, this involves managing front-to-back and back-to-front
        # The specific logic for updating depth_texture to support bidirectional peeling
        # is implementation-specific but generally involves maintaining two depth textures
        # or a combined depth buffer that supports both directions.
        # For the purpose of this algorithm description, we assume the depth texture
        # is updated to reflect the newly peeled layer's depth boundary.
        current_depth = get_next_depth_boundary(depth_texture)
        
    # BLEND STAGE
    # Apply silhouette detection and blending
    apply_sobel_filter(normal_buffer, depth_buffer)
    compute_silhouette_strength(sobel_results, normal_buffer.mask)
    blend_silhouette(color_buffer, silhouette_color, silhouette_strength)
    
    return color_buffer
```

**Edge Cases and Pitfalls:**
*   **Silhouette Width:** Silhouettes are extracted twice (once per pair of successive depth layers), which may result in inner silhouettes of increased width. This is mitigated by using a sharp and symmetric edge detection filter [S1].
*   **Banding Artifacts:** Slight banding artifacts can appear if blending directly to the output framebuffer in the presence of many surface layers. Using a floating-point color buffer improves blending accuracy [S1].
*   **Depth Sorting Precision:** Elements at different depths within the same layer may be stored if no intermediate geometry occludes them, reducing depth-sorting precision [S2].

## Subtopics

### [Effective Depth Peeling](17010-effective-depth-peeling.md)
Integrates Potentially Visible Hidden-Volume (PVHV) analysis into the depth peeling process to cull redundant fragments, reducing the number of effective layers required for high-quality multi-view warping.

### [Volume-peeling](17165-volume-peeling.md)
Extends Dual Depth Peeling to handle volumetric data intermixed with translucent polygonal geometry by using geometry peel depths as entry/exit points for volume ray-casting.

## Operating characteristics
*   **Time Complexity:** Rendering speed is largely a function of surface geometry size and is dominated by the requirement to submit the surface for rendering multiple times during depth peeling [S1].
*   **Memory Complexity:** Requires storage for multiple buffers: a floating-point color buffer, a normal buffer (with mask), and a depth texture [S1].
*   **Measured Performance:** Achieves interactive frame rates exceeding 15fps on NVIDIA GTX280 for small to medium meshes of below ≈500K triangles with medium depth complexity of 10 layers or less [S1].
*   **Strengths:** Resolves depth ordering per pixel, making it robust against self-intersecting surfaces which complicate depth sorting [S1]. Reduces the number of passes by $N/2$ for $N$ layers compared to standard depth peeling [S2].
*   **Limitations:** Performance degrades with larger meshes and higher depth complexity [S1]. May still store elements at different depths within the same layer if no intermediate geometry occludes them, reducing depth-sorting precision [S2].
*   **When to Use:** Suitable for interactive visualization of integral surfaces (e.g., stream surfaces, path surfaces) that may self-intersect, where correct transparency and silhouette shading are required [S1].
*   **When Not to Use:** Not suitable for scenes with very high depth complexity or extremely large meshes where the multiple render passes become prohibitive [S1].

## Background
Dual Depth Peeling was introduced as an improvement to the standard Depth Peeling algorithm to reduce the number of rendering passes required for order-independent transparency [S2]. It allows simultaneous peeling in both directions: from the camera to the background and from the background to the camera [S2]. The technique is implemented in OpenGL using GLSL shader programs, but the concepts can be ported to DirectX [S1]. It is particularly useful in illustrative rendering for integral surfaces, such as streamlines on stream surfaces, where visual clarity and correct transparency are critical [S1].

## Sources
[S1] IRIS: Illustrative Rendering for Integral Surfaces, https://doi.org/10.1109/tvcg.2010.173, fetched 2026-10-27
[S2] Hybrid visualisation of medical image: Surface and volume rendering ..., https://www.sciencedirect.com/science/article/pii/S0097849325002948, fetched 2026-10-28
