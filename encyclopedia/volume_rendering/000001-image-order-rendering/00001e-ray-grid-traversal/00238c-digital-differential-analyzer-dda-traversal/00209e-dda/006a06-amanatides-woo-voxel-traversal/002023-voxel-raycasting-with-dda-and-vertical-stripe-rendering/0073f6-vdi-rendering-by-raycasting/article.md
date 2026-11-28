# VDI Rendering by Raycasting

Also called: NDC Space Raycasting, Voxel Stepping VDI Rendering

## Overview
This method renders Volumetric Depth Images (VDIs) by casting rays from the new viewpoint into the Normalized Device Coordinate (NDC) space of the original view, where supersegment lists form a regular 2D grid of cuboids. Unlike object-space approaches that generate explicit geometry for each supersegment, this technique uses Amanatides-Woo voxel traversal to step through the grid, enabling efficient early ray termination and better scaling for large VDIs.

## How it works
**Inputs:**
*   Original view $V_0$ with viewport resolution $w \times h$.
*   New view $V_N$ with camera parameters.
*   VDI data structure: a set of supersegment lists $\mathbb{L}$, where each list $\mathbb{L}_i$ contains a sorted list of supersegments $\mathbb{S}$.
*   Volume bounding box in world space.

**Outputs:**
*   Final pixel color for each pixel in $V_N$.

**Data Structures:**
*   **NDC Space:** The coordinate system of $V_0$ mapped to $[-1, 1]$ on all axes.
*   **Supersegment Lists ($\mathbb{L}$):** In NDC space, these form a regular 2D grid. Each cell in the grid corresponds to a list $\mathbb{L}_i$.
    *   Cell width: $2/w$.
    *   Cell height: $2/h$.
    *   Each $\mathbb{L}_i$ is a cuboid in NDC space.
*   **Supersegments ($\mathbb{S}$):** Within each list $\mathbb{L}_i$, supersegments are sorted by position. Each $\mathbb{S}_j^i$ is a cuboid in NDC space.

**Steps:**

1.  **Ray Generation:** For each pixel in $V_N$, construct a ray $R$ originating from the camera center of $V_N$.
2.  **Ray Clipping:** Determine the start point $t_{near}$ and end point $t_{far}$ of the ray segment within the volume.
    *   Intersect $R$ with the viewport plane of $V_0$ to find the entry point into the VDI space.
    *   Intersect $R$ with the volume bounding box to find the exit point.
    *   The valid ray segment is $R(t)$ for $t \in [t_{near}, t_{far}]$.
3.  **NDC Transformation:** Transform the ray $R$ and the grid structure into the NDC space of $V_0$. In this space, the supersegment lists $\mathbb{L}$ form a regular 2D grid of cuboids.
4.  **Voxel Traversal (List Level):** Traverse the 2D grid of lists $\mathbb{L}$ using the Amanatides-Woo voxel traversal algorithm [2].
    *   Initialize traversal at the grid cell containing the ray origin in NDC space.
    *   Iterate through the grid, determining the next intersected list $\mathbb{L}_i$ and the intersection points (entry and exit) with $\mathbb{L}_i$.
    *   Stop traversal when the ray exits the volume bounding box or $t_{far}$ is reached.
5.  **Supersegment Search:** For each intersected list $\mathbb{L}_i$:
    *   Identify the supersegments $\mathbb{S}_j^i$ within $\mathbb{L}_i$ that overlap with the ray segment between the entry and exit points of $\mathbb{L}_i$.
    *   Use spatial smoothness: once the first intersecting supersegment is found, subsequent supersegments are found via adjacency search (checking the next sorted element), as they are sorted by position.
6.  **Intersection Length Calculation:** For each intersected supersegment $\mathbb{S}_j^i$:
    *   Calculate the intersection length $l_j$ of the ray with the cuboid $\mathbb{S}_j^i$ in NDC space.
    *   Note: The sources specify calculating the intersection length in NDC space to adjust color contribution. The exact mapping from NDC length to physical depth/color weight is implied to be handled by the VDI data (e.g., depth values stored in supersegments).
7.  **Color Accumulation:**
    *   Compute the color contribution $C_j$ from each supersegment $\mathbb{S}_j^i$, weighted by the intersection length $l_j$.
    *   Accumulate contributions using alpha compositing (front-to-back or back-to-front, typically front-to-back for early termination).
    *   Final pixel color $C_{final} = \sum C_j \alpha_j \prod (1-\alpha_k)$ for previous segments $k$.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_vdi_ray_casting(view_V0, view_VN, vdi_lists, volume_bbox):
    """
    view_V0: Original view info (resolution w, h, projection matrix)
    view_VN: New view info (camera position, orientation, resolution)
    vdi_lists: List of lists L_i, where each L_i is a list of supersegments S_j^i
    volume_bbox: Bounding box of the volume in world space
    """
    w, h = view_V0.resolution
    cell_width = 2.0 / w
    cell_height = 2.0 / h
    
    # Precompute grid dimensions in NDC space
    grid_cols = w
    grid_rows = h
    
    # Initialize output image
    image = zeros(view_VN.resolution)
    
    for y in range(view_VN.resolution[1]):
        for x in range(view_VN.resolution[0]):
            # 1. Generate ray in world space
            ray_origin, ray_direction = view_VN.get_ray(x, y)
            
            # 2. Clip ray to volume and VDI viewport
            t_near, t_far = clip_ray_to_volume_and_viewport(
                ray_origin, ray_direction, volume_bbox, view_V0.viewport_plane
            )
            
            if t_near > t_far:
                continue
                
            # 3. Transform ray to NDC space of V0
            # This involves transforming the ray origin and direction using V0's inverse projection/view
            ndc_ray_origin, ndc_ray_direction = transform_to_ndc_v0(
                ray_origin, ray_direction, view_V0
            )
            
            # 4. Amanatides-Woo traversal in NDC grid
            # Initialize traversal state
            current_cell_x, current_cell_y = ndc_to_grid_cell(ndc_ray_origin, cell_width, cell_height)
            
            # Calculate step directions and t_max values for DDA
            step_x = 1 if ndc_ray_direction.x > 0 else -1
            step_y = 1 if ndc_ray_direction.y > 0 else -1
            step_z = 1 if ndc_ray_direction.z > 0 else -1 # Assuming 3D traversal if needed, but L is 2D grid
            
            # Note: Amanatides-Woo is typically 3D. Here, L is a 2D grid of lists.
            # The traversal is over the 2D grid of lists. The ray's z-component in NDC determines depth.
            # However, the source says "traverses this grid using ... Amanatides and Woo".
            # We assume the grid is 2D (x,y) and we step through cells.
            
            # Initialize t_max for x and y steps
            t_max_x = ( (current_cell_x + (1 if step_x > 0 else 0)) * cell_width - ndc_ray_origin.x ) / ndc_ray_direction.x
            t_max_y = ( (current_cell_y + (1 if step_y > 0 else 0)) * cell_height - ndc_ray_origin.y ) / ndc_ray_direction.y
            
            t_delta_x = cell_width / abs(ndc_ray_direction.x) if ndc_ray_direction.x != 0 else infinity
            t_delta_y = cell_height / abs(ndc_ray_direction.y) if ndc_ray_direction.y != 0 else infinity
            
            accumulated_color = (0, 0, 0, 0)
            accumulated_alpha = 0
            
            # Traverse until t_far is reached or no more lists
            t_current = 0 # Relative t in NDC space? No, we need to map back to world t for accumulation.
            # Let's track world t for accumulation.
            world_t_current = t_near
            
            # We need to iterate through the grid cells.
            # The Amanatides-Woo algorithm steps through cells.
            # We need to know when we exit the volume in world space.
            
            while True:
                # Determine next cell
                if t_max_x < t_max_y:
                    current_cell_x += step_x
                    t_current = t_max_x
                    t_max_x += t_delta_x
                else:
                    current_cell_y += step_y
                    t_current = t_max_y
                    t_max_y += t_delta_y
                
                # Check if we are still within the grid bounds
                if not (0 <= current_cell_x < grid_cols and 0 <= current_cell_y < grid_rows):
                    break
                
                # Map t_current (NDC space parameter) to world space t
                # This is tricky because NDC space is perspective-deformed.
                # The source says "calculations are done in ... NDC space".
                # The intersection points with L_i are determined in NDC space.
                # We need to convert these NDC intersection points back to world space to get t_near/t_far for the supersegment.
                
                # Get the entry and exit points of the ray with the current list cell L_i in NDC space
                # The list cell is a cuboid in NDC space:
                # x in [current_cell_x * cell_width, (current_cell_x + 1) * cell_width]
                # y in [current_cell_y * cell_height, (current_cell_y + 1) * cell_height]
                # z is determined by the ray's z in NDC space? 
                # Actually, the list L_i is a 2D grid cell. The "depth" is handled by the supersegments within it.
                # The Amanatides-Woo traversal here is over the 2D grid of lists.
                # The ray intersects the list cell. We need to find the supersegments within this list.
                
                # Find the supersegments S_j^i in vdi_lists[current_cell_y][current_cell_x]
                # that intersect the ray segment within this list.
                
                # To do this, we need the entry and exit points of the ray with the list cell in NDC space.
                # Since the list cell is a 2D grid cell, the "entry" and "exit" in terms of the ray's path through the list
                # are determined by the ray's z-coordinate in NDC space? 
                # No, the source says "L_i form a regular 2D grid of cuboids".
                # This implies L_i has a z-extent in NDC space? 
                # Or is it a 2D grid where each cell contains a list of supersegments that are ordered by depth?
                # The source says "supersegment lists L form a regular 2D grid of cuboids".
                # And "S_j^i are cuboids".
                # If L is a 2D grid, then each L_i is a column in the grid?
                # "L_i form a regular 2D grid" suggests L_i are the cells.
                # So each cell (i,j) is a list L_{i,j}.
                # The ray traverses the 2D grid of these lists.
                # For each list L_{i,j}, we find the supersegments S within it.
                
                # Let's assume the Amanatides-Woo traversal is over the 2D grid of lists.
                # The ray enters a list cell L_{i,j}.
                # We need to find the intersection of the ray with the supersegments in L_{i,j}.
                
                # The source says: "determines the next intersected list L_i and the intersection points with L_i".
                # This implies the traversal gives us the entry and exit points of the ray with the list cell L_i in NDC space.
                
                # Let's get the entry and exit points of the ray with the list cell L_{i,j} in NDC space.
                # The list cell is a cuboid in NDC space.
                # Its x and y bounds are known from the grid.
                # Its z bounds? The source doesn't specify z bounds for L_i.
                # It says "L_i form a regular 2D grid of cuboids".
                # This might mean L_i are 2D planes? Or cuboids with some z-depth?
                # If they are cuboids, they must have a z-depth.
                # However, the grid is 2D. This suggests the z-dimension is handled by the supersegments within the list.
                # Or the "2D grid" is in the x-y plane of NDC space, and each cell L_i is a list of supersegments along the z-axis?
                # The source says "supersegment lists L form a regular 2D grid of cuboids".
                # And "S_j^i are cuboids".
                # And "L_i form a regular 2D grid".
                # This is ambiguous. Let's look at the Amanatides-Woo part.
                # "With only two floating-point and two integer additions... the L intersected next is determined".
                # This implies a 2D traversal.
                # So L_i are 2D cells in the x-y plane of NDC space.
                # But they are "cuboids". This implies they have a z-extent.
                # Perhaps the z-extent is the entire depth of the volume in NDC space?
                # Or perhaps the "cuboid" description is for the supersegments, and the lists are just 2D planes?
                # "supersegment lists L form a regular 2D grid of cuboids".
                # This suggests L_i are cuboids.
                # If L_i are cuboids in a 2D grid, they must share the same z-extent?
                # Let's assume L_i are cuboids with z-extent covering the volume's NDC z-range.
                
                # So, for each list cell L_{i,j}, we find the intersection of the ray with the cuboid L_{i,j}.
                # This gives us an entry point and an exit point in NDC space.
                # Then, within this list, we find the supersegments S_j^i that intersect the ray segment between these points.
                
                # To find the supersegments, we need to know their positions in NDC space.
                # The source says "S_j^i are cuboids".
                # And "S within L are sorted by their position".
                # This likely means sorted by z-coordinate in NDC space.
                
                # So, for the current list cell L_{i,j}:
                # 1. Find the intersection of the ray with the cuboid L_{i,j} in NDC space.
                #    This gives t_entry_ndc and t_exit_ndc.
                # 2. Convert t_entry_ndc and t_exit_ndc to world space t values (t_entry_world, t_exit_world).
                # 3. Search for supersegments in L_{i,j} that intersect the ray segment between t_entry_world and t_exit_world.
                #    Since S are sorted by z, we can use adjacency search.
                
                # However, the Amanatides-Woo traversal gives us the next cell.
                # It doesn't directly give the intersection points with the cell.
                # We need to compute the intersection of the ray with the cell boundaries.
                
                # Let's refine the pseudocode to reflect this.
                
                # Get the bounds of the current list cell L_{i,j} in NDC space
                x_min = current_cell_x * cell_width
                x_max = (current_cell_x + 1) * cell_width
                y_min = current_cell_y * cell_height
                y_max = (current_cell_y + 1) * cell_height
                # z bounds? Assume full volume NDC z range? Or is L_i just a 2D plane?
                # If L_i is a cuboid, it must have z bounds.
                # Let's assume the z bounds are the same for all L_i, covering the volume's NDC z range.
                z_min_ndc, z_max_ndc = get_volume_ndc_z_range(view_V0)
                
                # Intersect ray with the cuboid L_{i,j}
                # This is a standard ray-cuboid intersection
                t_entry_ndc, t_exit_ndc = intersect_ray_cuboid(
                    ndc_ray_origin, ndc_ray_direction, 
                    x_min, x_max, y_min, y_max, z_min_ndc, z_max_ndc
                )
                
                if t_entry_ndc > t_exit_ndc:
                    continue # Ray does not intersect this list cell
                
                # Convert t_entry_ndc and t_exit_ndc to world space t
                # This requires inverting the NDC transformation
                t_entry_world = ndc_t_to_world_t(t_entry_ndc, ndc_ray_origin, ndc_ray_direction, view_V0)
                t_exit_world = ndc_t_to_world_t(t_exit_ndc, ndc_ray_origin, ndc_ray_direction, view_V0)
                
                # Clip to the original ray segment [t_near, t_far]
                t_entry_world = max(t_entry_world, t_near)
                t_exit_world = min(t_exit_world, t_far)
                
                if t_entry_world > t_exit_world:
                    continue
                
                # Search for supersegments in L_{i,j}
                # Get the list of supersegments for this cell
                supersegments = vdi_lists[current_cell_y][current_cell_x]
                
                # Find the first supersegment that intersects the ray segment [t_entry_world, t_exit_world]
                # Since they are sorted by z, we can start from the beginning or use a cache
                # The source says "once the first ... is determined, the next is found by adjacency search"
                # This implies we can maintain a pointer to the current supersegment index across frames or pixels?
                # Or within the ray traversal?
                # "exploit the spatial smoothness across lists" suggests caching the last found supersegment index for the next list.
                
                # For simplicity, let's assume we search from the beginning or use a cached index
                # Let's use a cached index for the current ray's traversal
                # But the source says "across lists", so maybe across adjacent cells?
                # Let's assume we search from the beginning for now, or use a simple linear search if the list is small.
                # For efficiency, we should use the sorted order.
                
                # Find supersegments intersecting the ray segment
                # We need to convert the world space t segment to NDC space to compare with supersegment bounds?
                # Or convert supersegment bounds to world space?
                # The source says "calculates the intersection length with each S".
                # And "intersection length ... in NDC space".
                # So we should work in NDC space for the intersection length calculation.
                
                # Convert the world space segment [t_entry_world, t_exit_world] to NDC space segment
                # This gives us a segment in NDC space [t_entry_ndc_clip, t_exit_ndc_clip]
                # But we already have t_entry_ndc and t_exit_ndc from the list cell intersection.
                # We need to clip these to the supersegments.
                
                # For each supersegment S in supersegments:
                #   Check if S intersects the ray segment within the list cell.
                #   Since S are cuboids in NDC space, we can intersect the ray with S.
                #   If they intersect, calculate the intersection length.
                
                # To optimize, we can skip supersegments that are outside the list cell's z-range?
                # Or use the sorted order to find the first intersecting S.
                
                # Let's assume we iterate through the supersegments and check for intersection.
                # Since they are sorted by z, we can stop when we pass the exit point.
                
                for s in supersegments:
                    # Intersect ray with supersegment S in NDC space
                    t_s_entry_ndc, t_s_exit_ndc = intersect_ray_cuboid(
                        ndc_ray_origin, ndc_ray_direction,
                        s.x_min, s.x_max, s.y_min, s.y_max, s.z_min, s.z_max
                    )
                    
                    if t_s_entry_ndc > t_s_exit_ndc:
                        continue # No intersection
                    
                    # Clip to the list cell's intersection segment
                    t_entry_clip = max(t_s_entry_ndc, t_entry_ndc)
                    t_exit_clip = min(t_s_exit_ndc, t_exit_ndc)
                    
                    if t_entry_clip > t_exit_clip:
                        continue # No overlap within the list cell
                    
                    # Calculate intersection length in NDC space
                    length_ndc = t_exit_clip - t_entry_clip
                    
                    # Convert length_ndc to a weight for color contribution
                    # The source says "uses this to adjust the color contribution"
                    # The exact formula is not given, but it likely involves the depth value stored in S.
                    # Let's assume the color contribution is proportional to the length_ndc.
                    # Or maybe the length_ndc is used to sample the VDI data.
                    
                    # Get color and alpha from the supersegment S
                    # The source doesn't specify how color is retrieved.
                    # Let's assume S has a color and alpha value.
                    color_s, alpha_s = s.get_color_and_alpha()
                    
                    # Accumulate
                    # Alpha compositing
                    # C_out = C_in + C_s * alpha_s * (1 - alpha_in)
                    # alpha_out = alpha_in + alpha_s * (1 - alpha_in)
                    
                    # But we need to weight by the intersection length?
                    # The source says "adjust the color contribution ... based on the intersection length"
                    # This might mean the alpha or color is scaled by the length.
                    # Or the length is used to determine the depth for sampling.
                    
                    # Let's assume the color contribution is scaled by the length.
                    # This is a guess [G].
                    weight = length_ndc
                    adjusted_color = color_s * weight
                    adjusted_alpha = alpha_s * weight
                    
                    # Accumulate
                    accumulated_color = accumulated_color + adjusted_color * (1 - accumulated_alpha)
                    accumulated_alpha = accumulated_alpha + adjusted_alpha * (1 - accumulated_alpha)
                    
                    # Early termination if alpha is close to 1
                    if accumulated_alpha > 0.99:
                        break
                
                # Set pixel color
                image[y, x] = accumulated_color

    return image

def clip_ray_to_volume_and_viewport(ray_origin, ray_direction, volume_bbox, viewport_plane):
    # Intersect ray with viewport plane
    t_viewport = intersect_ray_plane(ray_origin, ray_direction, viewport_plane)
    if t_viewport < 0:
        return infinity, -infinity # Ray does not hit viewport
    
    # Intersect ray with volume bounding box
    t_near_vol, t_far_vol = intersect_ray_aabb(ray_origin, ray_direction, volume_bbox)
    
    # The valid segment is from max(t_viewport, t_near_vol) to t_far_vol
    t_near = max(t_viewport, t_near_vol)
    t_far = t_far_vol
    
    if t_near > t_far:
        return infinity, -infinity
    
    return t_near, t_far

def transform_to_ndc_v0(world_origin, world_direction, view_V0):
    # Inverse projection and view transformation
    # This is a standard graphics operation
    ndc_origin = view_V0.inverse_projection * view_V0.inverse_view * world_origin
    ndc_direction = view_V0.inverse_projection * view_V0.inverse_view * world_direction
    # Perspective divide
    ndc_origin = ndc_origin / ndc_origin.w
    ndc_direction = ndc_direction / ndc_direction.w # Direction doesn't need w for DDA? 
    # Actually, for DDA in NDC space, we need the direction in NDC space.
    # The ray in NDC space is P(t) = O + t * D.
    # If we transform the world ray to NDC, we get O_ndc and D_ndc.
    # But the transformation is non-linear (perspective).
    # The source says "calculations are done in the perspective-deformed NDC space".
    # This implies we transform the ray to NDC space and then use DDA.
    # The DDA works on the linear parameter t in NDC space?
    # No, DDA works on the grid cells.
    # The ray in NDC space is still a line if we use the correct parameterization.
    # But perspective projection makes lines in world space become lines in NDC space?
    # Yes, lines map to lines in NDC space.
    # So we can transform the ray origin and direction to NDC space.
    # O_ndc = P_inv * V_inv * O_world
    # D_ndc = P_inv * V_inv * D_world
    # But we need to handle the w-component.
    # The ray in NDC space is P(t) = O_ndc + t * D_ndc.
    # This is valid if we use the correct t.
    # The t in NDC space is not the same as t in world space.
    # We need to map t_world to t_ndc.
    # This is complex.
    # The source says "computes S intersections in the NDC space".
    # And "traverses this grid using ... Amanatides and Woo".
    # This implies the DDA is performed in NDC space.
    # So we need the ray in NDC space.
    # Let's assume we have a function to get the ray in NDC space.
    return ndc_origin, ndc_direction

def ndc_t_to_world_t(t_ndc, ndc_ray_origin, ndc_ray_direction, view_V0):
    # Convert t_ndc to world space t
    # This requires inverting the NDC transformation
    # P_ndc = O_ndc + t_ndc * D_ndc
    # P_world = V * P_inv * P_ndc
    # This is non-linear.
    # The source doesn't specify how to convert t_ndc to t_world.
    # It says "intersection points with L_i ... are then used to search for S".
    # And "calculates the intersection length with each S".
    # And "uses this to adjust the color contribution".
    # It doesn't explicitly say how to get the world space t for accumulation.
    # Maybe the accumulation is done in NDC space?
    # Or maybe the intersection length in NDC space is used as a proxy for depth.
    # Let's assume the color contribution is adjusted by the NDC length.
    # The source says "based on the intersection length with a ray from the new viewpoint".
    # This suggests the length in NDC space is the key.
    # So we might not need to convert to world space t for accumulation.
    # We just accumulate the color contributions weighted by the NDC length.
    pass

def intersect_ray_cuboid(ray_origin, ray_direction, x_min, x_max, y_min, y_max, z_min, z_max):
    # Standard ray-cuboid intersection
    # Returns t_entry, t_exit
    # If no intersection, returns infinity, -infinity
    pass

def get_volume_ndc_z_range(view_V0):
    # Get the NDC z range of the volume
    # This depends on the volume's world space bounds and the view V0
    pass
```

**Edge Cases and Pitfalls:**
*   **Precision:** Floating-point precision in NDC space may affect traversal accuracy. Use double precision if necessary.
*   **Empty Regions:** The algorithm proposes methods to skip empty regions. Implement a check for empty lists or supersegments to avoid unnecessary traversal.
*   **Degenerate Input:** If the ray is parallel to a grid cell face, handle division by zero in DDA step calculations.
*   **Early Termination:** Implement early ray termination when accumulated alpha exceeds a threshold (e.g., 0.99) to save computation.

## Operating characteristics
*   **Time Complexity:** Depends on the number of supersegments intersected by each ray. The Amanatides-Woo traversal is efficient, with $O(1)$ per grid cell visited. The supersegment search is linear in the number of supersegments per list, but spatial smoothness allows for adjacency search, reducing the average case.
*   **Memory Complexity:** Proportional to the size of the VDI data structure (supersegment lists).
*   **Performance:** Scales better to large VDIs than object-space approaches because it avoids generating geometry for each supersegment. Allows early ray termination.
*   **Strengths:** No explicit geometry generation, efficient traversal, early ray termination, leverages anisotropy of VDI.
*   **Limitations:** Requires VDI data structure to be pre-computed. Precision issues in NDC space.
*   **When to Use:** Real-time rendering of large volumes using VDIs.
*   **When Not to Use:** When VDI pre-computation is not feasible or when the volume is small and object-space rendering is simpler.

## Background
This method was proposed by Lochmann et al. in "Efficient Raycasting of Volumetric Depth Images for Remote Visualization of Large Volumes at High Frame Rates" [S1]. It builds on the Amanatides-Woo voxel traversal algorithm [2] and aims to improve upon object-space raycasting methods that generate explicit geometry for supersegments.

## Sources
[S1] Efficient Raycasting of Volumetric Depth Images for Remote Visualization of Large Volumes at High Frame Rates, https://doi.org/10.1109/pacificvis56936.2023.00014, fetched 2026-10-24
