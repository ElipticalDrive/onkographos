# Photon Splatting

Also called: Capturing Direct Illumination (Photon Splatting)

## Overview
Photon Splatting is a method for capturing direct illumination by projecting photons from light sources onto scene geometry and storing them in a sparse voxel octree. It solves the problem of efficiently accumulating incoming radiance for subsequent indirect illumination calculations (such as voxel cone tracing) by treating light-view map pixels as photons. It differs from [Per-Splat Relighting](3371-per-splat-relighting.md) by focusing on the initial capture and storage of direct light energy into the voxel structure rather than per-surface lighting updates.

## How it works
**Inputs:**
*   Light-view map: A depth/position map rendered from the light's perspective. Each pixel represents a photon.
*   Sparse Voxel Octree: A hierarchical data structure representing the scene, where leaf nodes store photon data.

**Outputs:**
*   Updated Octree: Leaf nodes contain direction distributions and energy values proportional to the subtended solid angle of the light-view map pixel.

**Data Structures:**
*   **Light-View Map:** Stores world positions and depth. Resolution is typically higher than the lowest level of the voxel grid.
*   **Octree Leaf Node:** Stores photon data. Due to hardware constraints, this is often implemented using 16-bit normalized texture formats to support atomic operations, denormalized upon access.
*   **Brick Structure:** Voxels are repeated at brick boundaries to enable hardware filtering. This redundancy causes thread collisions if photons are copied directly to all required adjacent locations.

**Steps:**
1.  **Render Light-View Map:** Render the scene from the light's view using standard rasterization to generate a world position map. Each pixel corresponds to a photon [S1].
2.  **Splat Photons:** Launch a fragment shader with one thread per light-view-map pixel. For each pixel:
    *   Determine the corresponding leaf node in the octree based on the world position.
    *   Calculate the photon's energy, which is proportional to the subtended solid angle of the pixel as seen from the light [S1].
    *   Store the photon as a direction distribution and energy in the leaf node.
3.  **Handle Atomic Operations:** Since atomic add operations are only available for integer textures, use a 16-bit normalized texture format for storage. Denormalize the values when accessing them later [S1].
4.  **Resolve Brick Boundary Inconsistencies:** Because voxels are repeated for adjacent bricks, direct copying causes thread collisions and bandwidth issues. Use the "Value Transfer to Neighboring Bricks" method to ensure coherence [S1, S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Step 1: Render Light-View Map
light_view_map = rasterize_scene_from_light_view(scene, light)

# Step 2: Splat Photons
# One thread per pixel in light_view_map
for each pixel in light_view_map:
    world_pos = pixel.world_position
    solid_angle = pixel.subtended_solid_angle
    
    # Find leaf node in octree
    leaf_node = octree.find_leaf(world_pos)
    
    # Photon data: direction and energy
    photon_energy = solid_angle * light_intensity
    photon_direction = normalize(light_position - world_pos)
    
    # Store using atomic add on 16-bit normalized texture
    # Note: Atomic add is performed on integer representation of 16-bit float
    atomic_add(leaf_node.texture_buffer, photon_energy, photon_direction)

# Step 3: Value Transfer to Neighboring Bricks
# To resolve brick boundary inconsistencies without atomic operations
# Assume octree is complete for simplicity, one thread per leaf node

# X-axis passes
for each leaf_node:
    # Add current node's data to the right neighbor
    right_neighbor = leaf_node.get_neighbor(axis='x', direction=+1)
    right_neighbor.texture_buffer += leaf_node.texture_buffer

for each leaf_node:
    # Copy sum from right neighbor back to current node
    right_neighbor = leaf_node.get_neighbor(axis='x', direction=+1)
    leaf_node.texture_buffer = right_neighbor.texture_buffer

# Y-axis passes
for each leaf_node:
    top_neighbor = leaf_node.get_neighbor(axis='y', direction=+1)
    top_neighbor.texture_buffer += leaf_node.texture_buffer

for each leaf_node:
    top_neighbor = leaf_node.get_neighbor(axis='y', direction=+1)
    leaf_node.texture_buffer = top_neighbor.texture_buffer

# Z-axis passes
for each leaf_node:
    front_neighbor = leaf_node.get_neighbor(axis='z', direction=+1)
    front_neighbor.texture_buffer += leaf_node.texture_buffer

for each leaf_node:
    front_neighbor = leaf_node.get_neighbor(axis='z', direction=+1)
    leaf_node.texture_buffer = front_neighbor.texture_buffer
```

**Parameters:**
*   **Light-View Map Resolution:** Typically higher than the lowest level of the voxel grid to avoid gaps [S1].
*   **Texture Format:** 16-bit normalized (e.g., RG16F) to support atomic adds [S1].

**Edge Cases and Pitfalls:**
*   **Atomic Operations:** Hardware atomic adds are limited to integer textures. Using 16-bit normalized formats requires careful denormalization [S1].
*   **Thread Collisions:** Copying photons directly to all required locations in adjacent bricks leads to significant performance degradation due to parallel random scattering and bandwidth issues [S1].
*   **Empty Voxels:** Photons are stored at the surface, and empty voxels are collapsed. Photons are always placed at the finest level of the voxel structure [S1].

## Subtopics

### Value Transfer to Neighboring Bricks
This variant addresses the inconsistency of duplicated voxels at brick boundaries. It uses six passes (two per axis: x, y, z) to propagate data. In the first pass of an axis, each thread adds its voxel data to the neighbor in the positive direction. In the second pass, it copies the summed data from the neighbor back. This ensures coherence along each axis without atomic operations or thread collisions, leveraging neighbor pointers for efficient access [S2].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of light-view map pixels for splatting. The value transfer step is linear with respect to the number of leaf nodes, with a constant factor of 6 passes.
*   **Memory Complexity:** Proportional to the number of leaf nodes in the octree. The use of 16-bit normalized textures reduces memory footprint compared to 32-bit floats.
*   **Performance:** Efficient for scenes with high-resolution light-view maps. The value transfer method avoids the performance hit of atomic operations and thread collisions associated with direct copying [S1, S2].
*   **Strengths:** Simple implementation, efficient use of hardware filtering, avoids atomic operations in the transfer step.
*   **Limitations:** Requires a complete octree assumption for the simplified explanation of value transfer. The 16-bit format may introduce precision issues for very high dynamic range lighting.

## Background
Photon Splatting was introduced in the context of Interactive Indirect Illumination Using Voxel Cone Tracing by Nishita et al. (2011). It builds on the idea of using light-view maps to capture direct illumination for subsequent indirect illumination calculations. The method was developed to address the challenges of storing photons in a sparse voxel octree while maintaining performance and accuracy [S1, S2].

## Sources
[S1] PDF Interactive Indirect Illumination Using Voxel Cone Tracing, https://research.nvidia.com/sites/default/files/publications/GIVoxels-pg2011-authors.pdf, fetched 2026-10-27
[S2] Interactive Indirect Illumination Using Voxel Cone Tracing, https://doi.org/10.1111/j.1467-8659.2011.02063.x, fetched 2026-10-26
