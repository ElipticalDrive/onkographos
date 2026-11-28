# Dual Isosurface Intersection

Also called: Dual Scalar Field Intersection, Isosurface Intersection Actor

## Overview
Dual Isosurface Intersection computes and renders the 3-D lines formed by the intersection of two distinct isosurfaces derived from different scalar fields defined on the same 3-D grid [S1]. It solves the problem of visualizing complex 3-D features where two physical thresholds cross, such as the extratropical tropopause delineated by potential vorticity and potential temperature surfaces [S1]. Unlike single isosurface extraction methods like [Marching Cubes](30-isosurface-extraction.md) which produce closed surfaces, this method produces 1-D line structures (ribbons or tubes) representing the locus of points where $f_a(\mathbf{x}) = \alpha_a$ and $f_b(\mathbf{x}) = \alpha_b$ simultaneously [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Two scalar fields ($V_a, V_b$) defined on the same 3-D grid; two isovalue thresholds ($\alpha_a, \alpha_b$); a bounding box defining the computational volume; optional filter variable $V_f$ and threshold $\alpha_f$; line length threshold $L_{min}$; ensemble member selection.
*   **Outputs:** A set of 3-D line segments (polylines) representing the intersection curves, rendered as tubes or lines with configurable color and thickness [S1].

**Data Structures:**
The method operates cell-by-cell across the volume defined by the bounding box [S1]. For each cell, the algorithm determines if the isosurfaces of $V_a$ and $V_b$ intersect within that cell. The specific geometric primitive used to represent the intersection within a cell (e.g., a line segment connecting two points on the cell edges) is not explicitly detailed in the sources, but the result is aggregated into line segments [S1].

**Steps:**
1.  **Variable Selection:** Select Variable $a$ and its Isovalue $\alpha_a$, and Variable $b$ and its Isovalue $\alpha_b$. Both must be defined on the same 3-D grid [S1].
2.  **Cell-wise Intersection:** Compute the intersection cell by cell across the volume defined by the bounding box [S1]. This involves finding points $\mathbf{x}$ within each cell where $V_a(\mathbf{x}) = \alpha_a$ and $V_b(\mathbf{x}) = \alpha_b$.
3.  **Line Length Filtering:** Apply a minimum length filter. Remove any line segment with length $L < L_{min}$. The default $L_{min}$ is 200 km [S1]. This discards small, isolated intersection artifacts caused by noise [S1].
4.  **Variable Value Filtering (Optional):** If a filter variable $V_f$ and threshold $\alpha_f$ are specified, remove line segments where the value of $V_f$ at any point on the segment falls below $\alpha_f$ [S1].
5.  **Rendering Preparation:**
    *   **Coloring:** Apply color based on the selected mode:
        *   *Constant:* Uniform color (default red) [S1].
        *   *Map pressure (hPa):* Color encodes the pressure level at each vertex via a transfer function [S1].
        *   *Map variable:* Color is sampled from a chosen actor variable at each vertex via a transfer function [S1].
    *   **Thickening:** Apply tube radius based on the selected mode:
        *   *Constant:* Uniform tube radius [S1].
        *   *Map variable:* Tube radius is linearly mapped from a chosen actor variable. The mapping uses an input value range and an output thickness range [S1].
    *   **Droplines:** Optionally place vertical pole markers at specific points along each intersection line to indicate pressure levels [S1].
6.  **Ensemble Handling:** If multiple ensemble members are selected, repeat steps 1–5 for each member independently [S1].

**Parameters:**
*   $L_{min}$: Minimum line length in km. Default: 200 km [S1].
*   Recompute on property change: Boolean. Default: Enabled. If disabled, computation is triggered manually via a "Compute intersection" button [S1].
*   Color Mode: Enum {Constant, Map pressure (hPa), Map variable} [S1].
*   Thickness Mode: Enum {Constant, Map variable} [S1].
*   Thickness Mapping: Sub-group for variable, input value range, and output thickness range [S1].

**Pitfalls and Edge Cases:**
*   **Noise:** Noise in input variables $V_a$ or $V_b$ produces short, fragmented segments [S1]. Mitigation: Smooth input variables using actor variable smoothing settings before computing the intersection [S1].
*   **Performance:** Computational expense scales linearly with the number of ensemble members, as each triggers a separate intersection computation [S1].
*   **Grid Alignment:** Both variables must be defined on the exact same 3-D grid; mismatched grids are not supported [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_isosurface_intersection(grid, var_a, var_b, alpha_a, alpha_b, 
                                  bounding_box, min_length_km=200.0, 
                                  var_f=None, alpha_f=None, 
                                  color_mode='constant', thickness_mode='constant',
                                  ensemble_members=None):
    """
    Computes 3-D intersection lines of two isosurfaces.
    
    Args:
        grid: 3-D grid structure containing cell connectivity and coordinates.
        var_a, var_b: Scalar field data arrays on the grid.
        alpha_a, alpha_b: Isovalue thresholds.
        bounding_box: AABB defining the region of interest.
        min_length_km: Minimum length of output line segments in km.
        var_f, alpha_f: Optional filter variable and threshold.
        color_mode: 'constant', 'map_pressure', or 'map_variable'.
        thickness_mode: 'constant' or 'map_variable'.
        ensemble_members: List of ensemble member indices to process.
        
    Returns:
        List of line segments (list of points), with associated metadata for rendering.
    """
    if ensemble_members is None:
        ensemble_members = [0]
        
    all_segments = []
    
    for member in ensemble_members:
        # Get data for this ensemble member
        data_a = var_a[member]
        data_b = var_b[member]
        data_f = var_f[member] if var_f is not None else None
        
        segments_for_member = []
        
        # Iterate over cells within the bounding box
        for cell in grid.cells_in_bounding_box(bounding_box):
            # Step 2: Compute intersection cell by cell
            # The specific geometric intersection logic (e.g., solving for 
            # x where var_a(x) = alpha_a and var_b(x) = alpha_b) is implementation-specific.
            # We assume a helper function returns the intersection segment(s) in this cell.
            cell_segments = compute_cell_intersection(
                cell, data_a, data_b, alpha_a, alpha_b
            )
            
            for seg in cell_segments:
                # Step 3: Line length filter
                length_km = calculate_length_km(seg)
                if length_km < min_length_km:
                    continue
                
                # Step 4: Variable value filter (optional)
                if data_f is not None:
                    min_val_f = min(data_f[point_idx] for point_idx in seg.indices)
                    if min_val_f < alpha_f:
                        continue
                
                segments_for_member.append(seg)
        
        # Step 5: Rendering preparation
        rendered_lines = []
        for seg in segments_for_member:
            # Apply coloring
            if color_mode == 'constant':
                color = (1.0, 0.0, 0.0, 1.0) # Default red
            elif color_mode == 'map_pressure':
                color = apply_pressure_transfer_function(seg)
            elif color_mode == 'map_variable':
                color = apply_variable_transfer_function(seg, var_f)
            
            # Apply thickness
            if thickness_mode == 'constant':
                radius = default_radius
            elif thickness_mode == 'map_variable':
                radius = linear_map_variable_to_thickness(seg, var_f)
            
            # Add droplines if enabled (not implemented here as it's a rendering detail)
            
            rendered_lines.append({
                'points': seg.points,
                'color': color,
                'radius': radius
            })
        
        all_segments.append(rendered_lines)
        
    return all_segments

def compute_cell_intersection(cell, data_a, data_b, alpha_a, alpha_b):
    """
    [G] Placeholder for the specific geometric algorithm to find the 
    intersection of two isosurfaces within a single cell. 
    Common approaches involve solving the system of equations or 
    using trilinear interpolation to find roots.
    """
    # Implementation detail not provided in sources.
    # Returns a list of line segments (pairs of 3D points) found in this cell.
    return []
```

## Subtopics

### Topologically Accurate Dual Isosurfacing Using Ray Intersection
Uses ray casting to find intersections, potentially offering different topological guarantees or performance characteristics compared to cell-wise grid traversal. [Link](800-topologically-accurate-dual-isosurfacing-using-ray-intersect.md)

### Isosurface Extraction API
Provides the interface for integrating isosurface methods, including dual isosurface intersection, into larger visualization pipelines. [Link](836-isosurface-extraction-api.md)

### VESTA
A method for volume rendering that may offer alternative visualization techniques for scalar fields, though not specifically focused on dual isosurface intersection lines. [Link](900-vesta.md)

### Marching Squares
The 2-D precursor to Marching Cubes, extracting contours from 2-D grids. Dual isosurface intersection is the 3-D analog for two fields. [Link](918-marching-squares.md)

### Dual Contouring and Surface Nets
Extracts a single isosurface with optimized geometry (dual contouring) or simplified vertices (surface nets), whereas dual isosurface intersection extracts the *intersection* of two surfaces. [Link](6319-dual-contouring-and-surface-nets.md)

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of cells in the bounding box and the number of ensemble members. Each cell requires solving for the intersection of two scalar fields, and each ensemble member requires a full pass [S1].
*   **Memory Complexity:** Linear with respect to the number of intersection segments generated. Noise can significantly increase the number of short segments before filtering, impacting memory usage [S1].
*   **Performance:** Computationally expensive for large domains or many ensemble members, especially when "Recompute on property change" is enabled [S1]. Disabling auto-recompute and using manual triggers is recommended for large domains [S1].
*   **Strengths:** Directly visualizes the 3-D locus of intersection between two physical thresholds, useful for meteorological features like the tropopause [S1]. Flexible filtering and appearance options aid in interpreting complex data [S1].
*   **Limitations:** Sensitive to noise in input data, requiring pre-smoothing [S1]. Limited to variables on the same 3-D grid [S1]. Performance degrades with many ensemble members [S1].
*   **When to use:** When the scientific question involves the interaction or boundary between two distinct scalar thresholds (e.g., PV and temperature) in 3-D space [S1].
*   **When not to use:** When single isosurface visualization is sufficient, or when variables are on different grids [S1].

## Background
Dual Isosurface Intersection is used primarily in meteorological visualization to study features such as PV (Potential Vorticity) streamers and cutoffs near the extratropical tropopause [S1]. The method builds on the concept of isosurface extraction but extends it to the intersection of two distinct fields [S1]. Key references in this domain include Wernli & Sprenger (2007) and Kunz et al. (2015), who identified and analyzed these features using ERA-15 and other datasets [S1].

## Sources
[S1] Isosurface intersection, https://met3d.readthedocs.io/en/latest/03_user_manual/actors/isosurface_intersection.html, fetched 2026-10-27
