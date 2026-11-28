# Flow Vector Visualization

Also called: Flow direction arrows, Vector Glyph Visualization, Velocity vector plot

## Overview
Flow Vector Visualization renders velocity vectors on a grid where the length and direction indicate flow speed and direction, while color encodes a scalar quantity [S1]. It solves the problem of visualizing vector fields by copying a generic object (glyph), positioning it at sample points, and scaling/coloring it based on vector data [S2]. It is distinguished from streamline methods by its discrete, point-based representation, which can create a "porcupine" effect [S2].

## How it works
**Inputs and Outputs:**
*   **Input:** A vector field containing a `positions` component (to identify sample points) and a `data` component (containing vector quantities) [S2]. Optionally, a `connections` component may exist [S2].
*   **Output:** A set of glyphs (arrows or rockets) representing the vector field [S2].

**Data Structures:**
*   **Field:** Must include:
    *   `positions`: Coordinates of sample points.
    *   `data`: Vector values at those points.
*   **Glyph:** A generic object (e.g., arrow) with a base and a tip. A directional dot is drawn at the vector tip to indicate flow direction [S1].

**Parameters:**
*   `vector_density_skip`: Integer step for sampling grid cells. Default is 1 (all vectors). Setting to 2 skips every other vector to reduce density [S1].
*   `vector_scale`: Float multiplier for vector length. Adjusted interactively [S1].

**Steps:**
1.  **Position the Base:**
    *   For position-dependent data: Fix the base of the vector glyph at the vertex position (sample point) [S2].
    *   For connection-dependent data: Fix the base at the center of the connection element [S2].
2.  **Scale the Length:** Scale the length of each vector glyph based on the vector magnitude relative to all other vectors in the field [S2].
3.  **Orient the Glyph:** Point the glyph in the direction of the vector given by the datum at that point [S2].
4.  **Color the Glyph:** Color the vector based on the value of the currently displayed scalar quantity [S1].
5.  **Draw the Tip:** Draw a small dot at the end of the line to indicate flow direction [S1].
6.  **Reduce Density (Optional):** Use a `Reduce` module to downsample the data field or a `Sample` module to extract a subset of points (e.g., from an isosurface) to feed to the Glyph module, thereby decreasing visual clutter [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_flow_vectors(field, vector_scale, vector_density_skip):
    """
    Renders flow vectors for a given field.
    
    Args:
        field: Object with 'positions' and 'data' components.
        vector_scale: Float multiplier for vector length.
        vector_density_skip: Integer step for sampling grid cells.
    """
    positions = field.positions
    vectors = field.data
    
    # Get magnitude of all vectors for relative scaling
    magnitudes = [norm(v) for v in vectors]
    max_magnitude = max(magnitudes) if magnitudes else 1.0
    
    rendered_glyphs = []
    
    for i, pos in enumerate(positions):
        # Apply density skip
        if i % vector_density_skip != 0:
            continue
            
        vector = vectors[i]
        magnitude = magnitudes[i]
        
        # Calculate scaled length
        # Note: Sources state scaling is relative to all other vectors.
        # A common implementation normalizes by max magnitude and applies scale factor.
        scaled_length = (magnitude / max_magnitude) * vector_scale
        
        # Determine base position
        # Assuming position-dependent data for simplicity as per S2 default
        base_pos = pos
        
        # Determine tip position
        # Direction is given by vector datum
        if magnitude > 0:
            direction = vector / magnitude
            tip_pos = base_pos + direction * scaled_length
        else:
            tip_pos = base_pos
            
        # Create glyph (arrow from base to tip)
        glyph = create_arrow_glyph(base_pos, tip_pos)
        
        # Color based on scalar quantity (not explicitly defined in sources, assumed available)
        # scalar_value = field.scalar_data[i] # [G] Assumed scalar data exists
        # glyph.color = map_scalar_to_color(scalar_value)
        
        # Add directional dot at tip
        glyph.add_tip_dot(tip_pos)
        
        rendered_glyphs.append(glyph)
        
    return rendered_glyphs
```

**Edge Cases and Pitfalls:**
*   **Intersection and Occlusion:** High density of vectors or large value disparities can cause vectors to intersect or occlude each other, making interpretation difficult [S2].
*   **Visual Clutter:** Fine meshes can result in very dense vector plots [S1].

## Operating characteristics
*   **Strengths:** Provides immediate visual feedback on flow direction and relative magnitude at specific points [S2].
*   **Limitations:** Can become hard to interpret if there are many vector data points or if vectors intersect/occlude each other [S2].
*   **Performance:** Rendering time depends on the number of vectors. Downsampling via `Reduce` or `Sample` modules can improve performance and clarity [S2].
*   **When to use:** When discrete vector information at specific points is needed, and the field is not excessively dense.
*   **When not to use:** When the field is very dense or when continuous flow paths are more important than discrete vectors (consider streamlines instead) [S2].

## Background
Flow Vector Visualization is a technique used in scientific visualization, particularly in tools like Data Explorer and Smokeview [S1, S2]. It is part of the broader category of glyph-based vector field visualization [S2].

## Sources
[S1] Smokeview (Version 6) A Tool for Visualizing Fire Dynamics Simulation Data Volume I : User’s Guide, https://doi.org/10.6028/nist.sp.1017-1, fetched 2026-10-26
[S2] User's Guide - Visualization Techniques, https://theochem.mercer.edu/dxdocs/pages/usrgu063.htm, fetched 2026-10-28
