# Rubbersheet Visualization

Also called: Height Map, Rubbersheet Projection

## Overview
Rubbersheet Visualization is a technique for visualizing data collected on a 2-dimensional grid by elevating the grid into the third dimension (Z), where the height or Z-value given to each vertex is proportional to the specified scalar data value at that vertex [S1]. It leverages human familiarity with physical height to show "ups and downs" in data values, resulting in a relief map-like appearance [S1]. Unlike sibling methods that may rely on thresholding or complex 3D table computations, this method provides a direct geometric deformation of the input grid [S1, 1515-grid-visualization-with-thresholding.md, 2083-3d-table-computation-on-gpu.md].

## How it works
**Inputs and Outputs:**
- **Input:** A 2-dimensional Field (grid) with scalar data values associated with vertices. Optionally, a "normals" component may be present [S1].
- **Output:** A 3-dimensional Field (height field) where vertices are displaced along the Z-axis [S1].

**Data Structures:**
The input is a 2D grid lying in the X-Y plane. Each vertex $v$ has coordinates $(x, y, 0)$ and an associated scalar value $s$. The output grid retains the $(x, y)$ coordinates but updates the Z coordinate to $z'$ [S1].

**Steps:**
1. **Check for Normals:** Determine if the input Field contains a "normals" component [S1].
   - If **no** normals component exists, the method assumes the input grid is flat and projects values in a perpendicular direction (along the Z-axis) [S1].
   - If a **normals** component exists, it is used to control the direction of projection [S1].
2. **Calculate Displacement:** For each vertex, calculate the new Z-coordinate based on the scalar data value and a scale parameter [S1].
3. **Apply Projection:** Displace the vertex from its original position $(x, y, 0)$ to $(x, y, z')$ [S1].
4. **Post-Processing (Optional):** To improve rendering quality, generate real surface normals for the projected surface using a Normals module, as the original normals (if any) may not accurately reflect the deformed geometry [S1].

**Formulas:**
Let $s_i$ be the scalar data value at vertex $i$, and $k$ be the scale parameter.
If no custom normals are provided, the new Z-coordinate $z'_i$ is:
$$ z'_i = k \cdot s_i $$
The vertex moves from $(x_i, y_i, 0)$ to $(x_i, y_i, z'_i)$ [S1].

If custom normals $\mathbf{n}_i$ are provided, the displacement direction is determined by $\mathbf{n}_i$. The exact formula for non-perpendicular projection is not specified in the sources, but the direction is controlled by the modified normals component [S1].

**Parameters:**
- **Scale ($k$):** Determines the proportionality between data values and Z-deflection. Example: if scale is 1.0 and minimum count is 0, a count of 2 results in a point at $z=2$ [S1].
- **Minimum Count:** Implicitly referenced as a baseline for displacement (e.g., if minimum count is 0, displacement is directly proportional to value) [S1]. Not given in the sources if this is a configurable parameter or a fixed assumption.

**Pitfalls:**
- **Misinterpretation of Data:** The 3-D representation is not a "realistic" image of the original 2-D sampling space. Data was collected at $z=0$, but is displayed at $z=z'$. It is the user's responsibility to make this clear to viewers [S1].
- **Flat Grid Assumption:** Rubbersheet assumes the input grid is flat if no "normals" component is present. If the input grid is not flat in the X-Y plane (e.g., already has Z variation), the behavior may be undefined or incorrect [S1].
- **Vector Data:** For vector data, the grid is elevated by the magnitude of the vector, as magnitude is a scalar value [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def rubbersheet_projection(field, scale=1.0):
    """
    Projects a 2D field into 3D by elevating vertices based on scalar values.
    
    Args:
        field: A 2D grid with vertices having (x, y, 0) coordinates and scalar values.
        scale: Proportionality factor for Z-deflection.
        
    Returns:
        A 3D field with vertices displaced along Z.
    """
    output_field = copy(field)
    
    # Check for normals component
    has_normals = "normals" in field.components
    
    for vertex in output_field.vertices:
        scalar_value = vertex.scalar_data
        
        if has_normals:
            # Use custom normals for projection direction
            # Note: Exact formula for non-perpendicular projection not specified in sources
            # Assuming displacement is along the normal direction scaled by scalar value
            normal = vertex.normals
            # Displacement vector = scalar_value * scale * normal
            # New position = old position + displacement
            # However, sources imply Z-deflection is the primary mechanism for 2D grids
            # If normals are provided, they control the direction.
            # For simplicity, if normals are Z-aligned (0,0,1), it reduces to Z-deflection.
            # If not, the implementation must project along the normal.
            # Since exact formula is missing, we assume standard Z-elevation if normals are not explicitly handled for direction.
            # [G] Guard: If normals are not Z-aligned, the displacement direction must be calculated.
            # For this implementation, we assume the common case of Z-elevation unless normals are explicitly used for direction.
            # If normals are present and not Z-aligned, the source does not specify the exact vector math.
            # We will assume the displacement is along the normal direction.
            displacement = scalar_value * scale * normal
            vertex.position = vertex.position + displacement
        else:
            # Assume flat grid, project in perpendicular direction (Z-axis)
            z_new = scale * scalar_value
            vertex.position.z = z_new
            
    return output_field
```

## Subtopics

### Height Map
A synonym for Rubbersheet Visualization, emphasizing the visual output resembling a topographic map [S1].

### Rubbersheet Projection
Emphasizes the geometric transformation process [S1].

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of vertices in the grid, as each vertex is processed independently [S1].
- **Memory Complexity:** Linear, as the output field requires storing the same number of vertices with an additional Z-coordinate [S1].
- **Strengths:** Leverages human familiarity with physical height; provides a powerful visualization technique for showing data variations [S1]. Combining color and elevation can be more instructive than either alone [S1].
- **Limitations:** Can be misleading if viewers interpret the 3-D height as actual physical elevation in the sampling space [S1]. Assumes input grid is flat if no normals are provided [S1].
- **When to Use:** When visualizing 2D grid data with scalar values and wanting to emphasize magnitude through height [S1].
- **When Not to Use:** When the data is not on a 2D grid, or when the Z-dimension has a different physical meaning than the scalar value [S1].

## Background
Rubbersheet Visualization originated in the IBM Visualization Data Explorer (DX) as a module for generating height maps [S1]. It is part of the broader family of direct volume ray casting and proxy geometry rendering techniques, specifically under slice-based and proxy geometry rendering [S1, 1507-2d-visualization-from-3d-grid-slow.md]. The technique is used to visualize data collected on a 2-dimensional grid, such as ecological data (e.g., grass species count) [S1].

## Sources
[S1] User's Guide - Visualization Techniques, https://theochem.mercer.edu/dxdocs/pages/usrgu063.htm, fetched 2026-10-28
