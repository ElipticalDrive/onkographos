# Clipping Cube and Cutting Plane Interaction

Also called: Focus-plus-context clipping

## Overview
This method enables interactive, selective suppression of volume datasets using an axis-aligned Clipping Cube and an arbitrary Cutting Plane, implemented via client-side widgets controlling server-side RTVR clipping capabilities [S1, unofficial]. It distinguishes itself from standard polygon clipping by supporting non-uniform cube resizing via corner spheres and rendering the context region outside the cube as non-photorealistic contours rather than transparent geometry [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A volume dataset, a 3D input device (pen and PIP pad), and a rendering engine supporting RTVR axis-aligned and oblique clipping planes [S1, unofficial].
*   **Output:** A rendered view where the interior of the clipping cube (or correct side of the cutting plane) is rendered as a solid surface (focus) and the exterior is rendered as invisible, transparent, or NPR contours (context), or where a single cutting plane isolates a specific slice [S1, unofficial].

**Data Structures:**
1.  **Clipping Cube Widget:** Represented as a wireframe cube with spheres at its corners, remaining axis-aligned, with spheres serving as interaction handles [S1, unofficial].
2.  **Cutting Plane:** Defined by an arbitrary alignment, acting like a single face of the clipping cube, rendered as a visual indicator during placement [S1, unofficial].

**Parameters:**
*   **Clipping Mode:** Determines the behavior of the volume outside the clipping cube, supporting "invisible" (hard clip), "transparent", or "NPR" (focus-plus-context) [S1, unofficial].
*   **Interaction Mode:** Switches the PIP pad between "moving object", "cutting plane", and "clipping cube" modes [S1, unofficial].

**Steps:**
1.  **Initialize Clipping Cube:** The system renders a wireframe cube with spheres at the corners, which light up when the pen enters their trigger volume [S1, unofficial].
2.  **Resize Clipping Cube:** The user drags a lit sphere, causing the cube to resize non-uniformly while maintaining its axis-aligned orientation, updating the six face planes [S1, unofficial].
3.  **Place Cutting Plane:** The user switches the PIP pad to "cutting" mode, where the pad itself acts as the cutting plane, moves the pad to position the plane through the volume, and presses the second pen button to freeze the plane in place [S1, unofficial]. The primary pen button is reserved for moving the volume object, enabling two-handed interaction (one hand moves the plane via the pad, the other moves the object via the pen) [S1, unofficial].
4.  **Render Focus and Context:** The region inside the clipping cube (or on the correct side of the cutting plane) is rendered using standard surface shading, while the region outside is rendered using Non-Photorealistic Rendering (NPR) contours or other specified modes [S1, unofficial]. Selected segmented objects can be excluded from the clipping effect, remaining shaded even when intersected by the clipping boundaries [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def update_interaction(pen, pad, cube, plane, mode):
    if mode == "CLIPPING_CUBE":
        for i, sphere in enumerate(cube.spheres):
            if distance(pen.position, sphere.center) <= sphere.radius:
                sphere.highlight = True
                if pen.is_dragging():
                    cube.spheres[i].center = pen.position
                    cube.update_axis_aligned_bounds()
            else:
                sphere.highlight = False
    elif mode == "CUTTING_PLANE":
        plane.position = pad.position
        plane.normal = pad.normal
        if pen.second_button_pressed:
            mode = "CLIPPING_CUBE"
    return mode

def compute_clipping_planes(cube, plane):
    planes = []
    for i in range(6):
        planes.append(cube.get_face_plane(i))
    planes.append(plane)
    return planes

def render_volume(clipping_planes, outside_mode):
    # Apply clipping planes to volume data
    clipped_data = apply_planes_to_volume(clipping_planes)
    # Render inside region as surface
    render_surface(clipped_data, region="inside")
    # Render outside region according to mode
    if outside_mode == "NPR_CONTOURS":
        render_npr_contours(clipped_data, region="outside")
    elif outside_mode == "INVISIBLE":
        pass
    elif outside_mode == "TRANSPARENT":
        render_transparent(clipped_data, region="outside")
```

**Edge Cases and Pitfalls:**
*   **Occlusion:** The combination of image-based rendering with polygon-based interaction widgets yields occlusion problems [S1, unofficial].
*   **Convoluted Objects:** For complex structures like vessels, distinguishing cut parts from the context can be difficult, making the NPR context rendering particularly valuable for clarity [S1, unofficial].

## Subtopics

### Clipping Cube Focus+Context with NPR Context
This variant specifically utilizes Non-Photorealistic Rendering (NPR) for the context region [S1, unofficial]. It isolates a focus region using a clipping cube widget (rendered as a surface) while rendering the context outside the cube as non-photorealistic contours [S1, unofficial]. This approach is chosen when the user needs to see the structural outline of the clipped volume without the visual clutter of full volume rendering [S1, unofficial]. The clipping cube widget is resizable via corner dragging and defines the bounded region for clipping [S1, unofficial]. Selected segmented objects can be excluded from clipping, remaining shaded even when covered by the clipping planes [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources [S1, unofficial].
*   **Memory Complexity:** Not given in the sources [S1, unofficial].
*   **Strengths:** Allows intuitive, direct manipulation of the clipping region [S1, unofficial]. Two-handed interaction (pad for plane, pen for object) improves workflow [S1, unofficial]. NPR context reduces visual clutter [S1, unofficial].
*   **Limitations:** The clipping cube is axis-aligned, limiting its orientation flexibility [S1, unofficial]. The cutting plane is single-sided [S1, unofficial]. Stereoscopic occlusion can be problematic with image-based rendering and polygon widgets [S1, unofficial].
*   **When to use:** When exploring internal structures of a volume while needing to see the surrounding context's shape [S1, unofficial].

## Background
This method was developed for distributed software-based volume visualization in a virtual environment [S1, unofficial]. It builds on earlier focus-plus-context work [FG98] and integrates with RTVR capabilities for axis-aligned and oblique clipping [S1, unofficial]. The approach was presented at the Eurographics conference in 2005 [S1, unofficial].

## Sources
[S1] Distributed Software-Based Volume Visualization in a Virtual Environment, https://doi.org/10.2312/egve/ipt_egve2005/129-140, fetched 2026-10-26 (unofficial)
