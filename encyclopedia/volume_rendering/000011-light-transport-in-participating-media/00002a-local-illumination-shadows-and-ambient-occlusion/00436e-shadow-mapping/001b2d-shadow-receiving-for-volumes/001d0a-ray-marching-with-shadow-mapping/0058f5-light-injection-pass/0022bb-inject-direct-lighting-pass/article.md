# Inject Direct Lighting Pass
Also called: Direct Light Injection, Voxel Direct Lighting Injection

## Overview
The Inject Direct Lighting Pass populates a voxelized scene's data structure with direct illumination data, enabling subsequent global illumination calculations (such as voxel cone tracing). It solves the problem of transferring light transport information from a standard rasterized shadow map into a 3D volumetric representation. It differs from its sibling, the [Light Injection Pass](2759-light-injection-pass.md), which operates in the GPU fragment stage using camera-space ray traversal and A-Buffer visibility; this pass operates in world space by traversing an octree from shadow map texels.

## How it works
**Inputs:**
- A voxelized scene represented as an octree data structure, stored in 3D texture bricks.
- A shadow map rendered from the light's point of view.
- Light direction and position.

**Outputs:**
- Updated 3D texture bricks containing reflected radiance values at leaf nodes.

**Data Structures:**
- **Octree:** Hierarchical structure where leaf nodes correspond to voxels. Each leaf stores radiance data.
- **Shadow Map:** 2D texture storing depth information from the light's perspective.

**Steps:**
1. **Render Shadow Map:** Render the scene from the light's point of view to generate a shadow map.
2. **Iterate Shadow Map Texels:** For each pixel (texel) in the shadow map:
   a. **Reconstruct World Position:** Convert the shadow map texel coordinates and depth value back into world-space coordinates.
   b. **Traverse Octree:** Traverse the octree data structure to locate the voxel(s) corresponding to the reconstructed world position.
   c. **Calculate Reflected Radiance:** Compute the direct lighting contribution. The method assumes the surface reflects diffusely.
   d. **Write to Bricks:** Write the calculated radiance value into the 3D texture bricks associated with the leaf nodes of the octree.

**Parameters:**
- **Cascade Shadow Map:** Uses a cascade shadow map with 4 cascades. The last cascade is specifically used for the light injection pass [S1, unofficial].
- **Slope Scale Depth Bias:** Disabled. This ensures that the reconstructed world position is exactly inside the voxel, preventing misalignment artifacts [S1, unofficial].

**Pitfalls and Edge Cases:**
- **Shadow Map Instability:** If the shadow map moves relative to the scene (e.g., due to camera movement), it causes flickering during the subsequent cone tracing step. Stabilization is required, such as computing the directional light position from the view camera [S1, unofficial].
- **Unfilled Voxels:** Stabilizing the shadow map may result in some voxels remaining unfilled with lighting data if the shadow map resolution is low or not fully utilized [S1, unofficial].
- **Resolution Artifacts:** Low shadow map resolution leads to incomplete light injection. Launching one thread per shadow map texel means only voxels aligned with texel centers receive light. For example, in a vertical column of 5 voxels, a low-resolution shadow map might only inject light into 3 of them, leaving gaps [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def inject_direct_lighting(octree, shadow_map, light_direction, light_position):
    # Step 1: Shadow map is assumed to be already rendered from light's POV
    
    # Step 2: Iterate over shadow map texels
    for y in range(shadow_map.height):
        for x in range(shadow_map.width):
            # Get depth from shadow map
            depth = shadow_map.sample(x, y)
            
            # Reconstruct world position from shadow map texel
            # Requires inverse projection and view matrices of the light
            world_pos = reconstruct_world_position(x, y, depth, light_view_matrix, light_proj_matrix)
            
            # Traverse octree to find leaf node(s) at world_pos
            leaf_nodes = octree.traverse(world_pos)
            
            for leaf in leaf_nodes:
                # Calculate reflected radiance assuming diffuse reflection
                # Note: Specific shading model details (e.g., albedo, NdotL) are not provided in sources
                # [G] Assuming standard diffuse direct lighting calculation
                radiance = calculate_diffuse_radiance(light_direction, leaf.normal, leaf.albedo)
                
                # Write radiance to 3D texture brick
                leaf.write_radiance(radiance)
```

## Subtopics

### [Light Injection Pass](16315-light-injection-pass.md)
A GPU fragment-stage algorithm that renders volume illumination by traversing light rays, computing visibility via binary search on the A-Buffer, and compositing light samples into camera space.

## Operating characteristics
- **Time Complexity:** Proportional to the number of shadow map texels. Each texel triggers an octree traversal.
- **Memory Complexity:** Requires storage for the shadow map and the 3D texture bricks.
- **Strengths:** Integrates well with existing shadow mapping infrastructure.
- **Limitations:** Highly sensitive to shadow map resolution. Low resolution causes visible gaps in lighting (unfilled voxels). Requires careful stabilization to avoid flickering.
- **When to use:** When a voxelized scene is already available and direct lighting needs to be baked into the voxel structure for subsequent global illumination passes like cone tracing.
- **When not to use:** When high-resolution shadow maps are not feasible, or when camera-relative lighting accuracy is critical without stabilization overhead.

## Background
This method is part of the Voxel Cone Tracing pipeline, specifically the step that injects direct lighting into the voxel data structure. It was described in the context of implementing voxel cone tracing in a custom engine, utilizing cascade shadow maps and octree traversal.

## Sources
[S1] Implementing Voxel Cone Tracing, http://simonstechblog.blogspot.com/2013/01/implementing-voxel-cone-tracing.html, fetched 2026-10-27 (unofficial)
