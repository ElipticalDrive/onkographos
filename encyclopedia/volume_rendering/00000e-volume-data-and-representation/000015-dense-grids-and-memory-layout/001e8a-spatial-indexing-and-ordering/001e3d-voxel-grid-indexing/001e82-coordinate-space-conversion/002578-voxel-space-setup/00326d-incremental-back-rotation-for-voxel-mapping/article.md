# Incremental Back-Rotation for Voxel Mapping

Also called: Back rotation, Destination-to-source rotation

## Overview
Incremental back-rotation is an algorithm used to map destination voxel coordinates to their corresponding source voxel positions during a room rotation operation, specifically to eradicate holes produced by forward rotation [S1]. It sets itself apart from forward mapping by iterating over the destination grid and computing source locations, ensuring every destination voxel is filled [S1]. This method builds on [Voxel Space Setup](3520-voxel-space-setup.md) by utilizing the established transformation matrices to perform efficient, incremental coordinate calculations [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Destination voxel grid dimensions, rotation angle $\theta$ (about the Z-axis), and the source voxel data structure [S1].
*   **Output:** Source voxel indices $(X_{src}, Y_{src})$ for each destination voxel $(X_{dest}, Y_{dest})$ [S1].

**Data Structures:**
*   **Source Room:** Defined by corners $(X_{min}, Y_{min}, Z_{min})$ and $(X_{max}, Y_{max}, Z_{max})$ [S1].
*   **Destination Room:** Determined by rotating the eight corners of the source room [S1].
*   **Coordinate System:** 2D projection for rotation logic (Z-axis rotation), with Z-coordinates handled via beam movement [S1].

**Steps:**
1.  **Locate Destination Room:** Rotate the eight corners of the source room using the standard rotation equations to determine the bounding box of the destination room [S1].
2.  **Determine Scan Start:** Identify the minimum coordinates of the destination room. To overcome overlap problems, start the scan at the source vertex that is also located within the destination room [S1].
3.  **Back-Rotate Minimum Coordinates:** Calculate the back-rotated starting positions for the destination room's minimum coordinates. This is achieved by replacing $\theta$ with $-\theta$ in the rotation equations [S1].
4.  **Incremental Back-Scan:** Iterate through the destination room using an incremental procedure. For each step, update the source coordinates using constant increments of $\sin(\theta)$ and $\cos(\theta)$ [S1].
5.  **Beam Movement:** Since all voxels along beams parallel to the rotation axis (Z-axis) are identically rotated, entire beams are moved for each new coordinate calculation [S1].

**Formulas:**
The standard forward rotation about the Z-axis is:
$$ X_{dest} = X_{src} \cos\theta - Y_{src} \sin\theta $$
$$ Y_{dest} = X_{src} \sin\theta + Y_{src} \cos\theta $$

The back-rotation equations, derived by replacing $\theta$ with $-\theta$, are used to find the source coordinates $(X_{src}, Y_{src})$ from destination coordinates $(X_{dest}, Y_{dest})$ [S1]:

For the horizontal increment (moving along X in destination):
$$ X_{src} = (X_{dest} + 1) \cos\theta + Y_{dest} \sin\theta $$
$$ Y_{src} = -(X_{dest} + 1) \sin\theta + Y_{dest} \cos\theta $$

For the vertical increment (moving along Y in destination):
$$ X_{src} = X_{dest} \cos\theta + (Y_{dest} + 1) \sin\theta $$
$$ Y_{src} = -X_{dest} \sin\theta + (Y_{dest} + 1) \cos\theta $$

Note: The source text presents these as Equations 11-14, where the left-hand side represents the back-rotated point from $(X_{dest}+1, Y_{dest})$ and $(X_{dest}, Y_{dest}+1)$ respectively [S1].

**Parameters:**
*   $\theta$: Rotation angle.
*   $\sin\theta, \cos\theta$: Precomputed trigonometric values used as constant increments [S1].

**Edge Cases and Pitfalls:**
*   **Holes:** Forward rotation can produce holes in the destination grid. Back-rotation solves this by ensuring every destination voxel maps to a source voxel [S1].
*   **Overlap:** The overlap problem is overcome by starting the scan at the source vertex which is also located within the destination room [S1].
*   **Clipping:** The destination room can be clipped first, so only visible parts need to be back-rotated [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def incremental_back_rotation(source_room, dest_room, theta):
    # Precompute trigonometric values
    cos_theta = cos(theta)
    sin_theta = sin(theta)
    
    # Step 1: Locate destination room by rotating source corners
    # (Assuming source_room has min/max corners)
    dest_corners = rotate_corners(source_room.corners, theta)
    dest_room.bounding_box = compute_bounding_box(dest_corners)
    
    # Step 2: Determine start point
    # Start at source vertex within destination room
    start_src = find_overlapping_source_vertex(source_room, dest_room)
    
    # Step 3: Back-rotate minimum coordinates of destination room
    # This gives the initial source coordinates for the top-left of dest room
    min_dest = dest_room.min_coords
    # Back-rotate using -theta logic (equivalent to swapping signs in inverse matrix)
    # Based on Eq 11/12 and 13/14 structure, we compute base src coords
    base_src_x = min_dest.x * cos_theta + min_dest.y * sin_theta
    base_src_y = -min_dest.x * sin_theta + min_dest.y * cos_theta
    
    # Step 4: Incremental back-scan
    # Iterate through destination room
    for y_dest in range(dest_room.height):
        for x_dest in range(dest_room.width):
            # Calculate current source coordinates
            # Using incremental updates: add sin/cos increments
            # Note: The exact incremental update depends on the direction of scan
            # The source mentions constant increments of sin(theta) and cos(theta)
            
            # For horizontal step (x_dest + 1):
            # dx_src = cos(theta), dy_src = -sin(theta)
            # For vertical step (y_dest + 1):
            # dx_src = sin(theta), dy_src = cos(theta)
            
            # Compute src for current dest
            src_x = base_src_x + x_dest * cos_theta + y_dest * sin_theta
            src_y = base_src_y - x_dest * sin_theta + y_dest * cos_theta
            
            # Map voxel
            dest_room.voxels[y_dest, x_dest] = source_room.voxels[src_y, src_x]
```
[G] The pseudocode above synthesizes the incremental nature described in the source. The source explicitly states that "constant increments for all voxels/beams are merely sin(theta) and cos(theta)" and that the calculation employs "two fixed-point additions" [S1]. The exact incremental update logic (whether to add or subtract based on scan direction) is implied by the derivative of the back-rotation equations.

## Subtopics

### Incremental Rotation Algorithm
This variant emphasizes the hardware implementation aspect where entire beams parallel to the rotation axis are moved using parallel memory and a barrel shifter for each new coordinate calculation [S1]. It relies on the same back-rotation principle to eradicate holes but focuses on the efficiency of moving whole beams rather than individual voxels [S1].

## Operating characteristics
*   **Time Complexity:** The coordinate calculation employs only two fixed-point additions that can be performed in parallel, making it computationally efficient [S1].
*   **Memory Complexity:** Requires storage for the source and destination rooms, plus precomputed trigonometric values [S1].
*   **Strengths:** Eliminates holes caused by forward rotation; efficient due to incremental updates and parallelizable additions [S1].
*   **Limitations:** Requires precomputation of destination room bounds; clipping must be handled separately [S1].
*   **When to use:** When rotating voxel data and hole-free results are critical, and hardware supports parallel fixed-point additions [S1].
*   **When not to use:** If rotation is not required, or if forward mapping with hole-filling is acceptable and simpler to implement [S1].

## Background
The method originates from the Voxblt Engine, a voxel frame buffer processor described in a 1988 paper by the authors of the Voxblt project [S1]. It was developed to address the specific challenges of rotating voxel data in real-time graphics, where forward rotation led to visual artifacts (holes) [S1]. The technique leverages the fact that voxels along beams parallel to the rotation axis are identically rotated, allowing for beam-based processing [S1].

## Sources
[S1] The voxblt Engine: A Voxel Frame Buffer Processor, https://doi.org/10.2312/eggh/eggh88/085-102, fetched 2026-10-26
