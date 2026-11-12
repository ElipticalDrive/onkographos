# Spatial Style Zones

Also called: Multi-level Interface Zones, Zones with Different Styles

## Overview
Spatial Style Zones is a technique for volume illustration where distinct regions of volume space (zones) are assigned different rendering styles, allowing for complex, multi-material visual effects [S1][S2, unofficial]. It differs from sibling methods by explicitly decoupling spatial location from material identity via "zones," enabling users to apply specific combinations of transfer functions (e.g., sketch, illumination) to arbitrary spatial subsets, rather than relying solely on local data statistics or model-based geometry [S1][S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A volume dataset $V$, a set of defined spatial zones $\mathcal{Z} = \{Z_1, ..., Z_n\}$, and a set of styles $\mathcal{S} = \{S_1, ..., S_m\}$. Each style $S_j$ is a set of transfer functions controlling color, opacity, and enhancement effects [S1][S2, unofficial].
*   **Output:** A rendered image where each sample point $\mathbf{x}$ is colored and shaded according to the style assigned to the zone containing $\mathbf{x}$ [S1][S2, unofficial].

**Data Structures:**
*   **Zone:** Defined geometrically as either an ellipsoid or a rectangle within the volume space [S1][S2, unofficial].
    *   *Ellipsoid:* Defined by center $\mathbf{c}$, axes $\mathbf{a}$, and orientation [S1][S2, unofficial].
    *   *Rectangle:* Defined by min/max bounds [S1][S2, unofficial].
*   **Style:** A collection of transfer function parameters (color, opacity, sketch intensity, illumination direction) [S1][S2, unofficial].
*   **Material Mapping:** Materials are regions in the transfer domain assigned meaningful names. Widgets in the interface are associated with Material IDs [S2, unofficial].

**Steps:**
1.  **Define Zones:** Create spatial regions (ellipsoids or rectangles) within the volume [S1][S2, unofficial].
2.  **Define Materials:** Identify regions in the transfer function domain (e.g., density ranges) and assign them meaningful names and Material IDs [S2, unofficial].
3.  **Assign Styles to Zones:** Associate each zone with a specific style. A style consists of a set of transfer functions that can be modified independently [S2, unofficial].
4.  **Map Materials to Styles:** Link Material IDs to specific widgets in the transfer function editor. Multiple widgets can share the same Material ID [S2, unofficial].
5.  **Render:** For each sample point $\mathbf{x}$:
    a. Determine which zone $Z_i$ contains $\mathbf{x}$ [S1][S2, unofficial].
    b. Retrieve the style $S_j$ associated with $Z_i$ [S2, unofficial].
    c. Evaluate the transfer functions in $S_j$ using the data value at $\mathbf{x}$ to determine color, opacity, and effects [S1][S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_spatial_style_zones(volume, zones, styles, materials):
    """
    volume: 3D array of scalar values
    zones: list of Zone objects, each with a 'contains(x)' method and 'style_id'
    styles: dict mapping style_id to a Style object
    materials: dict mapping material_name to material_id
    """
    image = empty_image(volume.shape)
    
    for x in volume.indices():
        # Determine which zone contains the current sample point
        zone = None
        for z in zones:
            if z.contains(x):
                zone = z
                break
        
        # If no zone contains the point, use a default style or background
        if zone is None:
            style = styles.get('default', None)
        else:
            style = styles[zone.style_id]
        
        # Get the data value at the sample point
        value = volume[x]
        
        # Determine the material ID for this value
        # Note: The source implies materials are regions in the transfer domain.
        # The specific mapping logic from value to material_id is not detailed,
        # but typically involves checking if value falls within the material's range.
        material_id = get_material_id_for_value(value, materials)
        
        # Apply the style's transfer functions
        # The style may contain multiple transfer functions (e.g., sketch, illumination)
        color = style.apply_color(material_id, value)
        opacity = style.apply_opacity(material_id, value)
        sketch_intensity = style.apply_sketch(material_id, value)
        
        # Composite the result
        image[x] = composite(image[x], color, opacity, sketch_intensity)
        
    return image

def get_material_id_for_value(value, materials):
    """
    Placeholder for the specific logic mapping a scalar value to a material ID.
    The source states materials are regions of the transfer domain.
    """
    # Implementation depends on the specific transfer function editor's data structure.
    # Not given in the sources.
    pass
```

**Parameters:**
*   **Zone Shape:** Ellipsoid or Rectangle [S2, unofficial].
*   **Style Components:** Color, opacity, sketch, illumination, enhancement parameters [S1][S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Zone Overlap:** The sources do not specify behavior for overlapping zones. [G] Typically, the first matching zone or a priority system is used.
*   **Unassigned Points:** Points not contained in any zone require a default style. [G]
*   **Complexity:** The number of parameters can be very large, making the system hard to master for novices [S2, unofficial].

## Subtopics

### Multi-level Interface
A UI approach where transfer functions are parameterized at different levels of abstraction (high-level materials, middle-level parameters, low-level transfer functions) to manage complexity [S1][S2, unofficial].

### Illustration Motifs
Pre-defined style combinations (e.g., "white sketch on everything except bone") that can be applied to zones [S1][S2, unofficial].

## Operating characteristics
*   **Complexity:** Not given in the sources.
*   **Strengths:** Allows a large set of possible effects by combining spatial zones with independent styles [S1][S2, unofficial]. Solves the problem of tedious management of many parameters by using materials and high-level descriptions [S2, unofficial].
*   **Limitations:** Hard to master due to the large number of parameters [S2, unofficial]. Requires user to define zones and styles manually [S1][S2, unofficial].
*   **When to use:** When creating illustrative visualizations for medicine or science where specific materials need distinct visual treatments (e.g., sketching, illumination) in specific spatial regions [S1][S2, unofficial].

## Background
Originated in the context of interactive volume illustration systems for medicine and science [S1][S2, unofficial]. The method is part of a broader effort to make volume visualization more accessible and expressive by using multi-level interfaces and illustration motifs [S1][S2, unofficial].

## Sources
[S1] Volume illustration for medicine and flows, https://doi.org/10.1145/1185657.1185694, fetched 2026-10-28
[S2] Illustrative Visualization for Medicine and Science, https://doi.org/10.2312/egt.20061068, fetched 2026-10-26 (unofficial)
