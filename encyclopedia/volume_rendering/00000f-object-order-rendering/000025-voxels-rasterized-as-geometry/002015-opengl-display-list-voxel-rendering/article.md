# OpenGL Display List Voxel Rendering

Also called: Display Lists, Fixed-pipeline Display Lists

## Overview
OpenGL Display List Voxel Rendering is a technique that pre-compiles voxel geometry into OpenGL 1.0 fixed-pipeline display lists to enable fast, repeated rendering without re-sending vertex data [S1, S2, unofficial]. It is distinguished from sibling methods like [Vertex Array Object Voxel Rendering](2226-vertex-array-object-voxel-rendering.md) and [Geometry Shader Processing](2524-geometry-shader-processing.md) by its reliance on legacy API commands (`glNewList`/`glEndList`) and its trade-off of significantly higher memory usage for superior per-frame render times [S1, S2, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** A set of voxel objects (e.g., spheres or cubes) defined by their visible faces. Each face contains position data (4 vertices) and a color. Interior voxels and occluded faces are removed prior to list creation [S1, S2, unofficial].
*   **Output:** A set of OpenGL display list IDs, one per voxel object instance, which can be executed via `glCallList` to render the geometry [S1, S2, unofficial].

**Data Structures**
The source does not specify a custom C++ struct layout for the display list itself, as this is managed by the OpenGL driver. However, the input data structure for faces is implied by the pseudocode:
*   `face`: Contains `color` (vector of 3 floats) and `points` (array of 4 vertices, each a vector of 3 floats) [S1, S2, unofficial].

**Steps**
1.  **Pre-processing:** Remove interior voxels and occluded faces from the voxel objects to optimize geometry [S1, S2, unofficial].
2.  **List Creation:** For each voxel object instance, generate a unique display list ID [S1, S2, unofficial].
3.  **Compilation:** Begin a new display list with `glNewList(id, GL_COMPILE)` [S1, S2, unofficial].
4.  **Geometry Emission:**
    *   Begin a quad primitive with `glBegin(GL_QUADS)` [S1, S2, unofficial].
    *   Iterate over each visible face of the voxel object [S1, S2, unofficial].
    *   Set the current color once per face using `glColor3fv(face.color)` [S1, S2, unofficial].
    *   Emit the four vertices of the quad using `glVertex3fv` for each point in `face.points` [S1, S2, unofficial].
    *   End the primitive with `glEnd()` [S1, S2, unofficial].
5.  **Finalization:** End the display list with `glEndList()` [S1, S2, unofficial].
6.  **Rendering:** Execute the display list during the render loop using `glCallList(id)` [G].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def create_display_list_for_voxel_object(voxel_object):
    """
    Creates an OpenGL display list for a single voxel object instance.
    Assumes OpenGL context is active.
    """
    # Step 2: Generate unique ID for this instance
    display_list_id = glGenLists(1)
    
    # Step 3: Begin compilation
    glNewList(display_list_id, GL_COMPILE)
    
    # Step 4: Begin quad primitive
    glBegin(GL_QUADS)
    
    # Pre-processing step (S1, S2): Ensure voxel_object.faces contains only visible faces
    for face in voxel_object.faces:
        # Step 4b: Set color once per face
        glColor3fv(face.color)
        
        # Step 4c: Emit four vertices
        for i in range(4):
            glVertex3fv(face.points[i])
            
    # Step 4d: End primitive
    glEnd()
    
    # Step 5: Finalize list
    glEndList()
    
    return display_list_id

def render_voxel_object(display_list_id):
    """
    Renders the voxel object by executing its display list.
    """
    glCallList(display_list_id)
```

**Parameters**
*   `GL_COMPILE`: Flag for `glNewList` indicating the list should be compiled but not executed [S1, S2, unofficial].
*   `GL_QUADS`: Primitive type for rendering voxel faces [S1, S2, unofficial].

**Edge Cases and Pitfalls**
*   **Memory Usage:** Display lists consume high main memory and VRAM, reported to be an order of magnitude greater than geometry shaders [S1, S2, unofficial].
*   **Setup Time:** Creating display lists is slow, although this does not affect per-frame render time in tests [S1, S2, unofficial].
*   **Immutability:** Display lists cannot be modified after creation [S1, S2, unofficial].
*   **Color Storage:** Specifying color once per face is more efficient than per-vertex, but the driver decides how to store this, which may not be optimal [S1, S2, unofficial].

## Operating characteristics
*   **Render Time:** Display lists are the fastest among compared techniques (VAOs, Geometry Shaders) in terms of average frame time [S1, S2, unofficial].
*   **Memory Usage:** High. Significantly higher than VAOs and an order of magnitude greater than geometry shaders [S1, S2, unofficial].
*   **Setup Time:** Slow. Creation of display lists is time-consuming, though this is a one-time cost per object instance [S1, S2, unofficial].
*   **Scalability:** Performance scales with the number of objects, but memory overhead grows linearly with the number of display lists [S1, S2, unofficial].
*   **Strengths:** Fastest render time; simple implementation; no need to manage vertex buffers or shaders [S1, S2, unofficial].
*   **Limitations:** High memory footprint; slow setup; immutable geometry; deprecated API [S1, S2, unofficial].
*   **When to use:** When render time is critical and memory is abundant, and legacy OpenGL support is available [S1, S2, unofficial].
*   **When not to use:** When memory is constrained, geometry needs to be dynamic, or modern OpenGL is required [S1, S2, unofficial].

## Background
This technique is based on OpenGL 1.0 fixed-pipeline display lists, which have been deprecated but remain supported in modern drivers [S1, S2, unofficial]. It was analyzed by Caleb Leak in "Quads All the Way Down: Simple Voxel Rendering" [S1, S2, unofficial]. The method was tested on 32x32x32 voxel spheres to compare render times and memory usage against VAOs and Geometry Shaders [S1, S2, unofficial].

## Sources
[S1] Quads All the Way Down: Simple Voxel Rendering - Medium, https://medium.com/@calebleak/quads-all-the-way-down-simple-voxel-rendering-fea1e4488e26, fetched 2026-10-27 (unofficial)
[S2] here, https://www.calebleak.com/posts/quads-all-the-way-down-voxel-rendering/, fetched 2026-10-27 (unofficial)
