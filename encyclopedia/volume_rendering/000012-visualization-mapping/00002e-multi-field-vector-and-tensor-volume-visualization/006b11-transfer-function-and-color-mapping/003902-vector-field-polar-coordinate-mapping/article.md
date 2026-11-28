# Vector Field Polar Coordinate Mapping

Also called: Magnitude-Azimuth-Elevation mapping, Spherical coordinate mapping for vector fields

## Overview
Vector Field Polar Coordinate Mapping is a transfer function technique that visualizes vector fields by decomposing each vector into magnitude, azimuth, and elevation relative to a reference axis, then mapping these components to opacity, hue, and saturation respectively [S1, unofficial]. It distinguishes itself from scalar-only transfer functions by explicitly encoding directional deviation from a free-stream flow into color properties, allowing the visualization of perturbation structures and shock waves in fluid dynamics [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** A 3D vector field $\mathbf{V}(x,y,z)$ and a reference "free stream" direction vector $\mathbf{V}_{ref}$ [S1, unofficial].
*   **Output:** A volume rendering where each voxel's color is defined by an RGB value and an opacity $\alpha$ [S1, unofficial].

**Data Structures**
The method operates on the vector field values directly. No auxiliary data structures are required beyond the standard voxel grid storing the vector components $(V_x, V_y, V_z)$ [S1, unofficial].

**Steps**
1.  **Define Reference Axis:** Designate the free stream velocity direction as the zero direction vector $\mathbf{V}_{ref}$ [S1, unofficial]. This vector serves as the polar axis [S1, unofficial].
2.  **Calculate Azimuth:** Compute the azimuth angle $\phi$ of the local vector $\mathbf{V}$ around the axis $\mathbf{V}_{ref}$ [S1, unofficial].
    *   The alignment of the zero azimuth is arbitrary; implementations may use a specific axis (e.g., the y-axis) as the zero reference [S1, unofficial].
    *   Formula: $\phi = \text{atan2}(\text{component perpendicular to } \mathbf{V}_{ref}, \text{component parallel to } \mathbf{V}_{ref})$ [G].
3.  **Map Azimuth to Hue:** Map the azimuth angle $\phi$ to the Hue component of the color space [S1, unofficial].
    *   Since hue is circular, the 360-degree range of $\phi$ maps naturally to the hue circle [S1, unofficial].
4.  **Calculate Elevation:** Compute the elevation angle $\theta$ between the local vector $\mathbf{V}$ and the reference vector $\mathbf{V}_{ref}$ [S1, unofficial].
    *   Formula: $\theta = \arccos\left(\frac{\mathbf{V} \cdot \mathbf{V}_{ref}}{\|\mathbf{V}\| \|\mathbf{V}_{ref}\|}\right)$ [G].
5.  **Map Elevation to Saturation:** Map the elevation angle $\theta$ to the Saturation component [S1, unofficial].
    *   Vectors parallel to the free stream ($\theta \approx 0^\circ$) are mapped to low saturation (nearly gray) [S1, unofficial].
    *   Vectors anti-parallel to the free stream ($\theta \approx 180^\circ$) are mapped to maximum saturation [S1, unofficial].
6.  **Map Magnitude to Opacity:** Calculate the magnitude $\|\mathbf{V}\|$ of the vector [S1, unofficial].
    *   Map the magnitude to the opacity $\alpha$ [S1, unofficial].
    *   Note: If $\mathbf{V}$ represents perturbation velocity, mapping magnitude directly to opacity renders the bulk irrelevant free stream flow invisible, highlighting boundaries and disturbances [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def vector_field_polar_mapping(V_local, V_ref, hue_zero_axis):
    """
    Maps a local vector V_local to color (H, S, L) and opacity alpha
    based on polar coordinates relative to V_ref.
    """
    # Normalize reference vector
    V_ref_norm = normalize(V_ref)
    
    # Calculate magnitude
    magnitude = length(V_local)
    
    # Calculate elevation angle (theta) between V_local and V_ref
    # Using dot product: cos(theta) = (A . B) / (|A| * |B|)
    cos_theta = dot(V_local, V_ref_norm) / (magnitude + epsilon)
    # Clamp to [-1, 1] to handle floating point errors
    cos_theta = clamp(cos_theta, -1.0, 1.0)
    theta = acos(cos_theta)
    
    # Calculate azimuth angle (phi) around V_ref
    # Project V_local onto the plane perpendicular to V_ref
    # Component parallel to V_ref
    parallel_component = dot(V_local, V_ref_norm) * V_ref_norm
    # Component perpendicular to V_ref
    perp_component = V_local - parallel_component
    
    # Define a basis vector for the perpendicular plane to calculate angle
    # Use hue_zero_axis (e.g., y-axis) projected onto the perpendicular plane
    # If hue_zero_axis is parallel to V_ref, this requires a fallback [G]
    zero_axis_proj = hue_zero_axis - dot(hue_zero_axis, V_ref_norm) * V_ref_norm
    zero_axis_proj = normalize(zero_axis_proj)
    
    # Calculate azimuth using atan2
    # We need a second orthogonal vector in the plane to form a coordinate system
    # cross(V_ref_norm, zero_axis_proj) gives the second axis
    second_axis = cross(V_ref_norm, zero_axis_proj)
    
    # Coordinates in the perpendicular plane
    x_coord = dot(perp_component, zero_axis_proj)
    y_coord = dot(perp_component, second_axis)
    
    phi = atan2(y_coord, x_coord)
    
    # Map to HSV
    # Hue: phi in [0, 2*pi] maps to [0, 1]
    hue = (phi + pi) / (2 * pi) # Shift to [0, 1]
    
    # Saturation: theta in [0, pi] maps to [0, 1]
    # Parallel (0) -> 0 Saturation (Gray)
    # Anti-parallel (pi) -> 1 Saturation (Full Color)
    saturation = theta / pi
    
    # Lightness: Can be fixed or derived from magnitude if needed. 
    # The source maps magnitude to opacity, not lightness [S1, unofficial].
    # We assume a constant lightness or a separate scalar field for L.
    lightness = 0.5 # Placeholder [G]
    
    # Opacity: Mapped from magnitude
    # Specific transfer function for magnitude is not detailed, 
    # but typically increases with magnitude for perturbations [S1, unofficial]
    opacity = magnitude_to_opacity(magnitude)
    
    return hsv_to_rgb(hue, saturation, lightness), opacity
```

**Parameters**
*   **Free Stream Vector ($\mathbf{V}_{ref}$):** The direction of undisturbed flow [S1, unofficial].
*   **Hue Zero Axis:** An arbitrary axis used to define the zero point for azimuth (e.g., y-axis) [S1, unofficial].

**Edge Cases**
*   **Zero Magnitude:** If $\|\mathbf{V}\| = 0$, the direction is undefined. The elevation and azimuth calculations involve division by zero or undefined cross products. A guard clause is required [G].
*   **Parallel Reference:** If the local vector is parallel to the reference vector, the perpendicular component is zero, making azimuth undefined. The saturation will be 0 (gray) regardless of azimuth [S1, unofficial].

## Subtopics

### Vector Field Color Map
A bivariate palette that visualizes both the direction and magnitude (scalar velocity) of a vector field simultaneously [S2, unofficial]. It associates luminance with the vector field direction and chrominance with scalar velocity [S2, unofficial]. This mapping is used to identify principal isoclines (lines where a directional derivative vanishes) and critical points (determined by the crossing of isoclines) [S2, unofficial].

## Operating characteristics
*   **Strengths:** Effectively visualizes perturbation structures in fluid dynamics by suppressing the dominant free-stream flow [S1, unofficial]. The use of polar coordinates provides a natural mapping to color space dimensions (hue for rotation, saturation for deviation) [S1, unofficial].
*   **Limitations:** Requires a well-defined free-stream direction; performance degrades or becomes ambiguous in turbulent or recirculating flows where a single reference axis is not representative [S1, unofficial]. The mapping of magnitude to opacity may require careful tuning to avoid obscuring relevant features if the free-stream magnitude is high [S1, unofficial].
*   **Complexity:** The computational cost per voxel is dominated by vector normalization, dot products, and trigonometric functions (atan2, acos) [G].

## Background
The method was introduced in the context of volume rendering for computational fluid dynamics (CFD) [S1, unofficial]. It was developed to address the challenge of visualizing velocity fields where the bulk flow obscures interesting perturbations and boundary effects [S1, unofficial]. The technique leverages the spherical coordinate system to create a more natural mapping into a multidimensional color space [S1, unofficial].

## Sources
[S1] Volume Rendering for Computational Fluid Dynamics: Initial Results, https://www.nas.nasa.gov/assets/nas/pdf/techreports/1991/rnr-91-026.pdf, fetched 2026-10-26 (unofficial)
[S2] Nuevos resultados en la visualización de campos vectoriales, http://sedici.unlp.edu.ar/handle/10915/21371, fetched 2026-10-26 (unofficial)
