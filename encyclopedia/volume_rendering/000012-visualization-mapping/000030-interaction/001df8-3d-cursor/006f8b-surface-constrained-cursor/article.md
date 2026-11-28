# Surface-constrained Cursor

Also called: 2D Point Cursor on Volumetric Display Surface

## Overview
The Surface-constrained Cursor is a visualization mapping technique that restricts a 2D input device's cursor to the outer surface of a volumetric display enclosure [S1]. It solves the problem of selecting objects in 3D space when the user's viewpoint is arbitrary and the display surface is non-planar, adapting standard ray-casting techniques for volumetric environments [S1]. Unlike the Silk Cursor, which projects a continuous surface mesh, this method treats the display boundary as a hard constraint for a single point or ray origin [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** 2D device coordinates (e.g., touch position, mouse click) and the geometric definition of the volumetric display enclosure (e.g., sphere, dome, cylinder) [S1].
*   **Output:** A 3D position on the enclosure surface and a selection of volumetric data objects intersected by the associated ray [S1].

**Data Structures:**
*   **Enclosure Geometry:** Defined by its surface equation or mesh. For a dome, this includes the ground plane and the hemispherical cap [S1].
*   **Ray:** Defined by an origin point on the surface and a direction vector [S1].
*   **Visual Aid:** A line, cone, or cylinder representing the ray for depth perception [S1].

**Steps:**
1.  **Ray Casting:** Cast a ray from the input device into the 3D scene. The ray orientation is determined by the enclosure geometry [S1]:
    *   **Perpendicular:** The ray is cast perpendicular to the tangent plane of the enclosure at the input point [S1].
    *   **Snap-to-Nearest:** The ray direction is adjusted to snap to the nearest object in the 3D scene [S1].
    *   **Ground Plane (Domes):** For non-planar enclosures like domes, the ray is cast straight down from the 2D cursor position to the ground plane of the dome [S1].
2.  **Visualization:** Display the ray as a visible line, cone, or cylinder to aid depth perception, as users view the ray from arbitrary angles rather than looking down along it [S1].
3.  **Selection:** Determine which object along the ray's path to select. Options include:
    *   Selecting the first object hit [S1].
    *   Selecting the last object hit [S1].
    *   Selecting all objects hit [S1].
    *   Using a controller on the input device to move a point along the ray, effectively acting as a constrained 3D point cursor [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def surface_constrained_cursor(input_2d_pos, enclosure_geometry, scene_objects):
    # Step 1: Determine ray origin and direction based on enclosure type
    if enclosure_geometry.type == "dome":
        # Cast straight down to ground plane
        ray_origin = input_2d_pos.project_to_ground_plane()
        ray_direction = Vector3(0, -1, 0)
    else:
        # Cast perpendicular to enclosure surface
        ray_origin = input_2d_pos.project_to_surface(enclosure_geometry)
        ray_direction = enclosure_geometry.get_normal_at(ray_origin)
        
    # Optional: Snap to nearest object
    nearest_object = find_nearest_object(scene_objects, ray_origin, ray_direction)
    if snap_to_nearest_enabled:
        ray_direction = normalize(nearest_object.position - ray_origin)

    # Step 2: Visualize ray
    render_ray(ray_origin, ray_direction, style="line") # or "cone", "cylinder"

    # Step 3: Select object
    intersections = ray_cast_scene(ray_origin, ray_direction, scene_objects)
    
    if selection_mode == "first_hit":
        selected = intersections[0] if intersections else None
    elif selection_mode == "last_hit":
        selected = intersections[-1] if intersections else None
    elif selection_mode == "all_hits":
        selected = intersections
    elif selection_mode == "constrained_3d":
        # Use controller to move point along ray
        selected = move_point_along_ray(ray_origin, ray_direction, controller_input)
        
    return selected
```

**Parameters:**
*   **Selection Mode:** Enum {first_hit, last_hit, all_hits, constrained_3d}. Default not specified [S1].
*   **Ray Visualization Style:** Enum {line, cone, cylinder}. Default not specified [S1].
*   **Snap Mode:** Boolean. Default not specified [S1].

**Edge Cases:**
*   **Arbitrary Viewpoints:** Standard 2D cursors assume a fixed viewpoint; this method must account for users viewing the ray from any angle, requiring explicit ray visualization [S1].
*   **Non-planar Enclosures:** Domes require specific handling (ground plane casting) compared to planar surfaces [S1].

## Subtopics

### [Volumetric Display Interaction System](21925-volumetric-display-interaction-system.md)
A system for interacting with a 3D volumetric display using direct touch and gestural input, leveraging the ability of users to walk around the display and interact from all directions. It differs from standard 3D cursors by simulating a touch-sensitive enclosure surface via motion tracking and providing specific techniques for file management (SurfaceBrowser) and 3D model transformation that respect the display's hemispheric geometry rather than imposing arbitrary global axes.

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Strengths:** Adapts familiar 2D interaction metaphors to 3D volumetric displays; provides clear depth cues via ray visualization [S1].
*   **Limitations:** Restricted to the display surface; selection ambiguity (first vs. last hit) requires explicit user control or mode selection [S1].
*   **When to use:** When users need to interact with the boundary of a volumetric display and the input device is 2D [S1].
*   **When not to use:** When interaction needs to occur freely within the volume interior or when the display geometry is complex and non-enclosing [S1].

## Background
The Surface-constrained Cursor was identified as one of three classes of cursors for 3D data in the context of volumetric display user interfaces [S1]. It was proposed as an adaptation of the status quo 2D cursor to volumetric environments, recognizing that traditional 2D cursors are insufficient due to the arbitrary viewpoints and 3D nature of volumetric displays [S1].

## Sources
[S1] User interfaces for volumetric displays, https://doi.org/10.1109/2.910892, fetched 2026-10-26
