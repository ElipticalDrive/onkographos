# End-point Corrector

Also called: Line Width Adjustment, Streamline End-point Adjustment

## Overview
The End-point Corrector is a technique used in volume rendering of vector fields to address radius shrinkage and field variation divergence near skeleton end-points. It forces an additional sampling position or adjusts streamline endpoints to increase the correlation between the minima of individual primitives and the global field [S1][S2, unofficial]. It sets itself apart from standard elongation by specifically targeting the boundary artifacts at the start and end of streamlines or skeleton segments, rather than modifying the length along the tangent [S1][S2, unofficial].

## How it works
The method operates in two distinct contexts found in the literature: correcting implicit surface radius errors in SCALIS fields [S1] and adjusting streamline geometry for line width in CUDA-based visualization [S2, unofficial].

### Inputs and Outputs
*   **Inputs:**
    *   Start point of the primitive/streamline ($P_{start}$) [S1][S2, unofficial].
    *   End point of the primitive/streamline ($P_{end}$) [S1][S2, unofficial].
    *   Prescribed radius ($r$) or line width ($m_{lineWidth}$) [S1][S2, unofficial].
    *   Field volume dimensions ($m_{vfInfo.realSize}$) [S2, unofficial].
    *   Boolean flags: `m_wingLine` (wing line mode) and `m_orientationX` (X orientation mode) [S2, unofficial].
*   **Outputs:**
    *   Adjusted start and end points ($P'_{start}, P'_{end}$) [S2, unofficial].
    *   Or, an additional sampling position near the end-point [S1].

### Data Structures
The implementation relies on 3D vector representations for points. In the CUDA implementation [S2, unofficial], points are stored as `float3` structures. The field information includes `realSize`, typically a vector representing the physical or voxel dimensions of the volume.

### Steps

#### Method A: Implicit Surface Correction (SCALIS)
This method addresses the issue where the prescribed radius is not achieved in the tangential direction near skeleton end-points [S1].

1.  Identify skeleton end-points where the field diverges from the expected behavior [S1].
2.  Add a segment primitive with a constant radius at these end vertices [S1].
3.  Determine the length of this segment. The length is proportional to the prescribed radius at the end vertices and a constant depending on the kernel used [S1].
4.  Force an additional sampling position near the skeleton end-points [S1].

*Note:* If end-point correctors are absent, a heuristic presented in section 6.2 of the source can compensate for this shortcoming [S1].

#### Method B: Streamline Line Width Adjustment
This method adjusts the start and end points of a streamline to account for line width, either by expanding from the midpoint (wing line) or by setting fixed bounds [S2, unofficial].

1.  Retrieve the start point ($m_{startPt}$) and end point ($m_{endPt}$) [S2, unofficial].
2.  Retrieve the line width parameter ($m_{lineWidth}$) [S2, unofficial].
3.  Check the `m_wingLine` flag:
    *   If `m_wingLine` is true:
        1.  Calculate the midpoint: $mid = \frac{m_{startPt} + m_{endPt}}{2.0}$
        2.  Calculate the start vector: $startVect = m_{startPt} - mid$
        3.  Calculate the end vector: $endVect = m_{endPt} - mid$
        4.  Update start point: $m_{startPt} = mid + startVect \times (m_{lineWidth} + 0.5)$
        5.  Update end point: $m_{endPt} = mid + endVect \times (m_{lineWidth} + 0.5)$
        6.  Return.
    *   If `m_wingLine` is false:
        1.  Check the `m_orientationX` flag.
        2.  If `m_orientationX` is true:
            1.  Set $m_{startPt}.y = (1.0 - m_{lineWidth}) \times \frac{m_{vfInfo.realSize.y}}{2}$
            2.  Set $m_{endPt}.y = m_{lineWidth} \times \frac{m_{vfInfo.realSize.y}}{2} + \frac{m_{vfInfo.realSize.y}}{2}$
        3.  Else (Y orientation):
            1.  Set $m_{startPt}.z = (1.0 - m_{lineWidth}) \times \frac{m_{vfInfo.realSize.z}}{2}$
            2.  Set $m_{endPt}.z = m_{lineWidth} \times \frac{m_{vfInfo.realSize.z}}{2} + \frac{m_{vfInfo.realSize.z}}{2}$

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def adjust_endpoints(start_pt, end_pt, line_width, field_size, wing_line, orientation_x):
    """
    Adjusts streamline endpoints based on line width and orientation.
    
    Args:
        start_pt: Tuple (x, y, z) of start point.
        end_pt: Tuple (x, y, z) of end point.
        line_width: Float, width of the streamline.
        field_size: Tuple (y_size, z_size) representing realSize.y and realSize.z.
        wing_line: Boolean, if True uses midpoint expansion.
        orientation_x: Boolean, if True adjusts Y axis, else Z axis.
        
    Returns:
        Tuple (new_start_pt, new_end_pt)
    """
    if wing_line:
        # Calculate midpoint
        mid_x = (start_pt[0] + end_pt[0]) / 2.0
        mid_y = (start_pt[1] + end_pt[1]) / 2.0
        mid_z = (start_pt[2] + end_pt[2]) / 2.0
        
        # Calculate vectors from midpoint to original points
        start_vect_x = start_pt[0] - mid_x
        start_vect_y = start_pt[1] - mid_y
        start_vect_z = start_pt[2] - mid_z
        
        end_vect_x = end_pt[0] - mid_x
        end_vect_y = end_pt[1] - mid_y
        end_vect_z = end_pt[2] - mid_z
        
        # Scale by (line_width + 0.5)
        scale = line_width + 0.5
        
        # Update points
        new_start_x = mid_x + start_vect_x * scale
        new_start_y = mid_y + start_vect_y * scale
        new_start_z = mid_z + start_vect_z * scale
        
        new_end_x = mid_x + end_vect_x * scale
        new_end_y = mid_y + end_vect_y * scale
        new_end_z = mid_z + end_vect_z * scale
        
        return (new_start_x, new_start_y, new_start_z), (new_end_x, new_end_y, new_end_z)
    
    else:
        y_size = field_size[0]
        z_size = field_size[1]
        
        if orientation_x:
            # Adjust Y coordinates
            new_start_y = (1.0 - line_width) * (y_size / 2.0)
            new_end_y = line_width * (y_size / 2.0) + (y_size / 2.0)
            
            # X and Z remain unchanged (assuming start_pt/end_pt contain original X/Z)
            new_start_x = start_pt[0]
            new_start_z = start_pt[2]
            new_end_x = end_pt[0]
            new_end_z = end_pt[2]
            
            return (new_start_x, new_start_y, new_start_z), (new_end_x, new_end_y, new_end_z)
        else:
            # Adjust Z coordinates
            new_start_z = (1.0 - line_width) * (z_size / 2.0)
            new_end_z = line_width * (z_size / 2.0) + (z_size / 2.0)
            
            # X and Y remain unchanged
            new_start_x = start_pt[0]
            new_start_y = start_pt[1]
            new_end_x = end_pt[0]
            new_end_y = end_pt[1]
            
            return (new_start_x, new_start_y, new_start_z), (new_end_x, new_end_y, new_end_z)
```

### Parameters
*   **Line Width (`m_lineWidth`):** The width of the streamline. Typical values depend on the visualization scale, but the formula uses it as a multiplier relative to the field size or radius [S2, unofficial].
*   **Wing Line (`m_wingLine`):** Boolean flag. When true, the correction expands symmetrically from the midpoint [S2, unofficial].
*   **Orientation X (`m_orientationX`):** Boolean flag. Determines whether the Y-axis (if true) or Z-axis (if false) is used for the fixed bounds calculation [S2, unofficial].
*   **Kernel Constant:** In the SCALIS method, the length of the corrector segment depends on a constant related to the kernel used [S1]. Not given in the sources.

### Edge Cases and Pitfalls
*   **Absence of Correctors:** In structures without end-points (e.g., truss structures), the assumptions related to end-point correctors may not apply, and heuristics from section 6.2 of the source should be used instead [S1].
*   **Precision:** The calculation involves floating-point arithmetic for midpoint and scaling. Standard `float` precision is used in the CUDA implementation [S2, unofficial].
*   **Boundary Conditions:** The fixed bound method assumes the field size is centered or aligned such that the calculated bounds fit within the volume. If `line_width` is outside [0, 1], the bounds may exceed the field size [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per streamline/primitive, as it involves a constant number of arithmetic operations [S1][S2, unofficial].
*   **Memory Complexity:** $O(1)$ additional memory per primitive for storing adjusted points [S1][S2, unofficial].
*   **Strengths:** Effectively reduces radius shrinkage artifacts at skeleton end-points [S1]. Provides flexible adjustment of streamline geometry for better visual continuity [S2, unofficial].
*   **Limitations:** Requires knowledge of field dimensions or kernel constants [S1][S2, unofficial]. The heuristic alternative for missing correctors may not be as accurate [S1].
*   **When to use:** When visualizing vector fields with skeleton-based primitives where end-point artifacts are visible, or when rendering streamlines with specific line width requirements [S1][S2, unofficial].

## Background
The concept of end-point correction in vector field visualization originated from the need to accurately render implicit surfaces defined by skeleton-based primitives. The SCALIS method, described in [S1], addresses the divergence of field values near skeleton end-points, a problem related to radius shrinkage observed in [ZBQC13]. The CUDA-based implementation [S2, unofficial] adapts similar concepts for streamline visualization, allowing for line width adjustments via wing lines or fixed bounds.

## Sources
[S1] Fast Ray Tracing of Scale‐Invariant Integral Surfaces, https://doi.org/10.1111/cgf.14208, fetched 2026-10-26
[S2] NightElfik/Vector-field-visualization-using-cuda NightElfik/Vector-field-visualization-using-cuda on GitHub, https://github.com/NightElfik/Vector-field-visualization-using-cuda, fetched 2026-10-29 (unofficial)
