# 3D Texture Slicing with Back-to-Front Blending

Also called: TMRenderAction Pipeline

## Overview
This method implements volume rendering by polygonizing volumetric geometry into slices parallel to the viewport, sorting them back-to-front, and compositing them using the over operator [S1]. It differs from sibling methods by relying on explicit application-side visibility sorting and OpenGL state management rather than depth peeling or object-aligned plane generation [S1].

## How it works
**Inputs and Outputs:**
- **Inputs:** A list of volumetric shapes (residents), OpenGL transformation matrices, and blending parameters.
- **Outputs:** A composited 2D image on the framebuffer.

**Data Structures:**
The source does not specify internal data structure layouts for the render action itself, but notes that it manages texture objects and lookup tables [S1].

**Steps:**
1. **Application Setup:** The application computes the resident shapes and sorts them in the correct visibility order. Note that `TMRenderAction` does not perform this sorting; it must be done by the application [S1].
2. **OpenGL State Initialization:** The application enables blending and sets the blending function for the standard over operator:
   $$ \text{glEnable(GL\_BLEND)} $$
   $$ \text{glBlendFunc(GL\_SRC\_ALPHA, GL\_ONE\_MINUS\_SRC\_ALPHA)} $$
   [S1]
3. **Begin Draw Phase:** The application calls `beginDraw()`. The render action performs the following:
   - Computes total resources required for the list of managed shapes [S1].
   - Manages OpenGL state by pushing the application's state and storing transformation matrices queried directly from OpenGL [S1].
   - Manages OpenGL resources by creating and downloading texture objects and lookup tables [S1].
4. **Rendering Loop:** The application iterates through the shapes in the pre-sorted visibility order. For each shape:
   - The render action invokes the shader's initialization routine, which binds texture objects and enables lookup tables [S1].
   - The render action polygonizes the volumetric geometry using the stored transformation matrices [S1].
   - The render action draws the polygonized geometry in a back-to-front order [S1].
   - *Note:* The polygonized geometry is always parallel to the viewport unless the application has explicitly set slicing planes on the volumetric geometry [S1].
5. **End Draw Phase:** The application calls `endDraw()`. The render action restores the modified OpenGL state, including texture settings, lookup tables, and pixel store [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_volume(shapes, gl_context):
    # 1. Application computes resident shapes and sorts them
    sorted_shapes = sort_shapes_by_visibility(shapes)
    
    # 2. Application sets OpenGL state for over operator
    gl_context.enable(GL_BLEND)
    gl_context.blend_func(GL_SRC_ALPHA, GL_ONE_MINUS_SRC_ALPHA)
    
    # 3. Begin Draw Phase
    gl_context.push_state()
    # Render action computes resources, creates textures/LUTs
    # Render action queries and stores transformation matrices from OpenGL
    gl_context.begin_draw()
    
    # 4. Rendering Loop
    for shape in sorted_shapes:
        # Render action invokes shader initialization
        # Binds textures, enables lookup tables
        gl_context.bind_shader(shape.shader)
        
        # Render action polygonizes volumetric geometry
        # Uses stored transformation matrices
        # Geometry is parallel to viewport unless slicing planes are set
        polygons = polygonize_volumetric_geometry(shape, gl_context.stored_matrices)
        
        # Render action draws polygonized geometry back-to-front
        for polygon in polygons:
            gl_context.draw_polygon(polygon)
            
    # 5. End Draw Phase
    gl_context.end_draw()
    # Render action restores modified OpenGL state (textures, LUTs, pixel store)
    gl_context.pop_state()
```

**Parameters:**
- **Blending Function:** Default is `GL_SRC_ALPHA, GL_ONE_MINUS_SRC_ALPHA` for the over operator [S1].
- **MIP Alternative:** To implement Maximum Intensity Projection, change the blending equation to `glBlendEquation(GL_MAX)` [S1].

**Edge Cases and Pitfalls:**
- **Sorting:** The render action does not sort shapes; the application must provide them in visibility-sorted order [S1].
- **Transformation Matrices:** Matrices are queried once in `beginDraw()` and used for all subsequent draws until `endDraw()`. Changes to matrices between these calls will not be reflected [S1].
- **Slicing Planes:** Polygonized geometry is always parallel to the viewport unless the application explicitly sets slicing planes [S1].

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Texture memory is described as a "very valuable resource" requiring efficient management for interactive rates, but specific memory usage formulas are not provided [S1].
- **Strengths:** Hides machine-specific details of texture management; allows flexibility in blending functions (e.g., MIP) [S1].
- **Limitations:** Requires application-side sorting; transformation matrices are static during the draw phase [S1].

## Background
This method is implemented by the `TMRenderAction` class in the SGI OpenGL Volume Rendering toolkit [S1]. It is based on the 3D texture slicing technique described in the associated documentation [S1].

## Sources
[S1] Chapter 4. Texture Mapping Render Action, https://techpubs.jurassic.nl/library/manuals/4000/007-4389-010/sgi_html/ch04.html, fetched 2026-10-27
