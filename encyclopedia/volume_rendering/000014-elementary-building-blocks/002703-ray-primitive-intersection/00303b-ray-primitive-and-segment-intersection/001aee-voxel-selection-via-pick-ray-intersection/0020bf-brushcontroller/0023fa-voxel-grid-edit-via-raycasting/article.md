# Voxel Grid Edit via Raycasting
Also called: Raycasting Sphere Brush
## Overview
This method enables interactive voxel painting and erasing by casting a ray from the camera through the mouse cursor to determine the brush position and surface normal on the voxel grid [S2, unofficial]. It differs from global noise generation by allowing localized, immediate edits via mouse input, bridging the gap between static voxel generation and dynamic user interaction [S1, unofficial]. It builds on [BrushController](2362-brushcontroller.md) by implementing the specific raycasting logic for voxel selection [S2, unofficial].
## How it works
**Inputs:** Ray origin ($O$), ray direction ($D$), brush mode (Add or Remove), and the voxel grid data structure (typically an Octree) [S2, unofficial].
**Outputs:** Modified voxel grid state and updated hover information [S2, unofficial].

**Data Structures:**
The method relies on a `HoverInfo` structure to track the current interaction state [S2, unofficial]:
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```cpp
struct HoverInfo {
    bool hasHit;       // True if the ray intersected a voxel boundary
    OctreeNode* node;  // Pointer to the intersected node (nullptr if none)
    glm::vec3 position; // World-space hit position
    glm::vec3 normal;   // Surface normal at the hit point
    int depth;          // Depth of the octree node
};
```
Default constructor initializes `hasHit=false`, `node=nullptr`, and `depth=0` [S2, unofficial].

**Steps:**
1. **Raycast for Hit:** Call the voxel grid's raycast function with the ray origin, direction, and the inverse of the brush mode: `voxelGrid->raycastVoxel(rayOrigin, rayDir, hitPosition, hitNormal, !add)` [S2, unofficial]. The `!add` parameter indicates whether to look for an existing voxel to remove (if `add` is false) or an empty space to add to (if `add` is true) [S2, unofficial].
2. **Adjust Hit Position (Add Mode Only):** If the mode is Add and a hit is found, offset the hit position along the surface normal to prevent self-intersection or z-fighting in subsequent frames: `hitPosition += hitNormal * 0.01f` [S2, unofficial].
3. **Modify Voxel:**
   * **If Add:** Insert a voxel at the adjusted position using `voxelGrid->addVoxel(hitPosition)` [S2, unofficial]. Alternatively, a spherical brush mask can be applied to modify voxels within a local radius around the hit point [S1, unofficial].
   * **If Remove:** Retrieve the `hoverInfo` node. If `hoverInfo.node` exists and is active, remove the voxel at its position using `voxelGrid->removeVoxel(hoverInfo.node->position)` [S2, unofficial].
4. **Update Hover State:** Immediately refresh the hover information to reflect the new grid state by calling `renderer.updateHoveredVoxel(rayOrigin, rayDir)` [S2, unofficial]. Internally, this resets `hoverInfo.hasHit` to false and calls `raycastOctreeForHover(root, rayOrigin, rayDir, 1)` to re-evaluate the intersection [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def edit_voxel_grid(ray_origin, ray_dir, is_adding, voxel_grid, renderer):
    # Step 1: Raycast
    # !add logic: if is_adding is True, pass False to look for empty space.
    hit_found, hit_pos, hit_normal = voxel_grid.raycast_voxel(ray_origin, ray_dir, not is_adding)
    if not hit_found:
        return False

    # Step 2: Adjust position for adding
    if is_adding:
        hit_pos = hit_pos + hit_normal * 0.01

    # Step 3: Modify voxel
    if is_adding:
        voxel_grid.add_voxel(hit_pos)
    else:
        hover_info = voxel_grid.get_hover_info()
        if hover_info.node is not None and hover_info.node.is_active():
            voxel_grid.remove_voxel(hover_info.node.position)

    # Step 4: Update hover state
    renderer.update_hovered_voxel(ray_origin, ray_dir)
    return True
```

**Parameters:**
* `brush_size`: radius of the brush [S1, unofficial].
* `brush_voxel`: color/index to place (0 for break) [S1, unofficial].
* Normal offset factor: $0.01$ [S2, unofficial].

**Edge Cases:**
* **No Hit:** If the ray does not intersect any voxel boundary, the function returns early and no state is modified [S2, unofficial].
* **Inactive Nodes:** When removing, the method checks if the node is active. If not, no removal occurs [S2, unofficial].
* **Precision:** The small normal offset ($0.01$) is used to prevent visual artifacts when adding voxels adjacent to existing ones [S2, unofficial].
## Subtopics

### [Raycast-based Voxel Editing](5211-raycast-based-voxel-editing.md)
Modifies density values within a local radius around the hit point to sculpt terrain, triggering chunk regeneration. Chose this when interactive, localized sculpting is required rather than single-voxel placement.

## Operating characteristics
* Time complexity: Not given in the sources.
* Memory complexity: Not given in the sources.
* Strengths: Immediate visual feedback; efficient for sparse voxel grids [S2, unofficial].
* Limitations: Single-voxel edits may be slow for large-scale terrain sculpting compared to radius-based methods [S2, unofficial].
* When to use: Interactive modeling, precise placement [S2, unofficial].
* When not to use: Large-scale terrain generation where noise functions are more efficient [S2, unofficial].
## Background
Originated in the `model_voxelizer` project by saurlax [S2, unofficial]. It is used in 3D modeling tools that require interactive voxel manipulation [S2, unofficial].
## Sources
[S1] kosshi-net/voxplat, https://github.com/kosshi-net/voxplat, fetched 2026-10-27 (unofficial)
[S2] GitHub - saurlax/model_voxelizer: A 3D model voxelization tool that ..., https://github.com/saurlax/model_voxelizer, fetched 2026-10-30 (unofficial)
