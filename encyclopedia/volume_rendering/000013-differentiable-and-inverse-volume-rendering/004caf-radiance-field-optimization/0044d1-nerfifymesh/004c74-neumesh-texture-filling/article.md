# NeuMesh Texture Filling
Also called: Sliding-window texture filling, Texture Filling

## Overview
NeuMesh Texture Filling is a method for editing the appearance of a specific region on a textured 3D mesh within a Neural Radiance Field (NeRF) framework by transferring texture characteristics from a pre-trained template. It solves the problem of localized artistic editing without retraining the entire radiance field or requiring multi-view image supervision. Unlike [NeRFifyMesh](11365-nerfifymesh.md), which generates a ground-truth radiance field from scratch, this method modifies an existing model by binding a new radiance decoder to the edited area and fusing it with the original model using inverse distance weighting to ensure smooth transitions.

## How it works
**Inputs:**
1. A pre-trained NeuMesh model (geometry, original texture codes, and original radiance decoder).
2. A target UV-map of the selected area to be edited, obtained by unwrapping the target vertices (e.g., using Blender).
3. A texture template from a pre-trained NeuMesh model (e.g., a small squared patch with ~10 vertices representing a specific material like fur or metal).

**Outputs:**
1. An edited NeuMesh model where the target area exhibits the appearance of the texture template, smoothly blended with the original model.

**Data Structures:**
- **Texture Codes:** A latent vector associated with each vertex or query point that encodes local geometric and textural information.
- **Radiance Decoder:** A neural network (MLP) that maps texture codes and viewing directions to color values.
- **UV Coordinates:** 2D coordinates mapping 3D mesh vertices to the 2D texture space.

**Steps:**
1. **UV Unwrapping:** Obtain the target UV-map for the selected vertices of the mesh using a standard 3D modeling tool (e.g., Blender).
2. **Template Selection:** Select a texture template from a pre-trained NeuMesh model. The template should be a small patch (e.g., ~10 vertices) representing the desired appearance.
3. **Sliding-Window Filling:** Repeatedly fill the target UV-map with the texture template in a sliding-window manner. This involves mapping the template's texture codes to the target vertices.
4. **Code Assignment:** Assign texture codes to the target vertices by interpolating the codes from the selected template.
5. **Decoder Binding:** Bind a new radiance decoder to the target area. This decoder is initialized or copied from the pre-trained model because the target texture codes and template texture codes do not share the same latent radiance space as the original model's global decoder.
6. **Color Fusion:** For query points near the boundary of the edited area that have texture codes or decoders from different sources (original vs. template), fuse the color contributions from the different decoders using inverse distance weighting. This ensures a smooth transition between the edited appearance and the original appearance.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def neu_mesh_texture_filling(mesh, target_uv_map, template_patch, original_decoder):
    """
    Perform texture filling on a NeuMesh model.
    
    Args:
        mesh: The original 3D mesh with vertices and faces.
        target_uv_map: UV coordinates for the vertices to be edited.
        template_patch: A small patch (~10 vertices) from a pre-trained NeuMesh model.
        original_decoder: The radiance decoder of the original NeuMesh model.
        
    Returns:
        edited_mesh: The mesh with updated texture codes and a fused decoder.
    """
    # Step 1: Get target vertices and their UVs
    target_vertices = get_vertices_from_uv_map(mesh, target_uv_map)
    
    # Step 2: Interpolate template texture codes to target vertices
    # Assuming template has texture codes C_template and target has initial codes C_target
    # Interpolation is done in UV space or based on vertex proximity in UV space
    interpolated_codes = interpolate_texture_codes(template_patch.texture_codes, 
                                                   template_patch.uv_coords, 
                                                   target_uv_map)
    
    # Step 3: Assign interpolated codes to target vertices
    for v in target_vertices:
        v.texture_code = interpolated_codes[v.id]
        
    # Step 4: Create/Bind new decoder for the target area
    # The new decoder is bound to the template's decoder structure
    # because the latent spaces differ
    new_decoder = copy_decoder(template_patch.decoder)
    
    # Step 5: Define fusion function for query points
    def fused_color_query(point, direction):
        # Determine which decoder(s) to use based on point's texture code source
        # If point is in target area, use new_decoder
        # If point is outside, use original_decoder
        # If point is near boundary, use both
        
        is_in_target = is_point_in_target_area(point, target_vertices)
        
        if is_in_target:
            color_new = new_decoder(point.texture_code, direction)
        else:
            color_new = None
            
        color_orig = original_decoder(point.texture_code, direction)
        
        if is_in_target:
            # Calculate distance to boundary for weighting
            dist_to_boundary = distance_to_boundary(point, target_vertices)
            # Inverse distance weighting
            weight = 1.0 / (dist_to_boundary + epsilon)
            # Normalize weight or use a sigmoid falloff
            # For simplicity, assume a smooth transition function w(d)
            w = smooth_transition_weight(dist_to_boundary)
            return w * color_new + (1 - w) * color_orig
        else:
            return color_orig

    # Step 6: Update mesh with new codes and decoder binding
    edited_mesh = mesh
    edited_mesh.target_vertices = target_vertices
    edited_mesh.decoder = fused_color_query
    
    return edited_mesh
```

**Parameters:**
- **Template Size:** Typically a small squared patch with ~10 vertices.
- **Learning Rate:** Fixed learning rate for optimization (if optimization is used), as per NeuMesh original paper.
- **Optimizer:** Adam optimizer.

**Edge Cases and Pitfalls:**
- **Latent Space Mismatch:** The target texture code and template texture code do not share the same latent radiance space, necessitating the binding of a new decoder.
- **Boundary Artifacts:** Without inverse distance weighting, sharp discontinuities may appear at the boundary of the edited area.
- **UV Unwrapping Quality:** Poor UV unwrapping can lead to distorted texture transfer.

## Subtopics

### NeRF-Tex
NeRF-Tex is a method that combines NeRF with a 2D neural texture on an underlying 3D mesh, conditioned on user-defined parameters for artistic editability. It allows artists to control the appearance of the scene through these parameters. Unlike NeuMesh Texture Filling, which uses a sliding-window approach with a pre-trained template, NeRF-Tex uses continuous neural textures and explicit user parameters. It is chosen when fine-grained, parameter-driven artistic control is required rather than template-based transfer.

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Measured Performance:** Not given in the sources.
- **Strengths:** Allows localized editing without retraining the entire model; smooth transitions via inverse distance weighting.
- **Limitations:** Requires a pre-trained NeuMesh model and a suitable texture template; dependent on UV unwrapping quality.
- **When to use:** When editing specific regions of a textured mesh in a NeRF with a desired appearance from a template.
- **When not to use:** When global editing or parameter-driven control is needed (use NeRF-Tex instead).

## Background
NeuMesh Texture Filling was introduced in the paper "NeuMesh: Learning Disentangled Neural Mesh-Based Implicit Field for Geometry and Texture Editing" [S1]. It builds on the NeuMesh framework, which learns disentangled neural mesh-based implicit fields for geometry and texture. The method addresses the need for precise 3D texture editing by leveraging a sliding-window technique and decoder fusion.

## Sources
[S1] NeuMesh: Learning Disentangled Neural Mesh-Based Implicit Field for Geometry and Texture Editing, https://doi.org/10.1007/978-3-031-19787-1_34, fetched 2026-10-26
[S2] Advances in Neural Rendering, https://doi.org/10.48550/arxiv.2111.05849, fetched 2026-10-26 (unofficial)
