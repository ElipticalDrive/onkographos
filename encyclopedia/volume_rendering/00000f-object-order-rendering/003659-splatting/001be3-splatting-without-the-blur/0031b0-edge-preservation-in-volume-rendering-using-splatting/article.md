# Edge Preservation in Volume Rendering Using Splatting

Also called: Edge-enhanced splatting, Warping splatting

## Overview
Edge Preservation in Volume Rendering Using Splatting is a method to preserve sharp edge details in splatting for volume rendering by applying a non-linear warping to the footprints of conventional splats and building a table of footprints for different possible edge positions and edge strengths [S1]. Conventional splatting algorithms produce fuzzy images for views close to the volume model because overlapping interpolation kernels ensure a smooth transition from one voxel to another, preventing the preservation of edges [S1]. This method sets itself apart from siblings like [Splatting Without the Blur](1157-splatting-without-the-blur.md) and [Pre-Classified Splatting](5410-pre-classified-splatting.md) by manipulating the underlying reconstruction kernels and resulting footprints directly, rather than deferring classification or shading to post-projection stages [S1]. It allows for the embedding of non-linear and anisotropic interpolation functions into the footprint tables to maintain sharpness while zooming in [S1].

## How it works
**Inputs:**
*   Volume data set with scalar values.
*   Edge position and edge strength parameters ($t_1, t_2, m_1, m_2$).
*   Base kernel footprint $g(x,y)$, typically the Crawfis and Max kernel [S1].

**Outputs:**
*   Warped splat footprints $h(x,y)$ for rendering.

**Data Structures:**
*   **Footprint Table:** A lookup table storing pre-computed warped footprints $h(x,y)$ indexed by edge position and edge strength. The table covers different possible edge positions and edge strengths [S1].
*   **Splat Classification:** Each voxel near an edge is classified into one of four splat types (0, 1, 2, 3) based on its relative position to the closest edge and whether it is a primary or secondary kernel [S1].

**Definitions:**
*   **Primary Kernels:** Splats with edges closer than 1.0 voxel units to the center [S1].
*   **Secondary Kernels:** Splats with edges still within a splat’s extent, but farther than 1.0 voxel units to the center [S1].
*   **Base Kernel:** The Crawfis and Max kernel [Crawfis93], a low-pass filter that decays rapidly to zero at an extent of about 1.6 voxel units [S1]. Using this kernel, only 64 ($4 \times 4 \times 4$) space reconstruction basis functions come into play for any point in 3D space [S1]. Consequently, on the direction perpendicular to the edge, no more than 4 consecutive voxels (2 on each side of the edge) need to be handled [S1].
*   **Intervals:** The region of interaction is divided into three parts: Interval 1, Interval 2, and Interval 3. The edge lies within Interval 2. Interval 2 is compressed to a smaller region, while Intervals 1 and 3 are stretched [S1].
*   **Parameters:**
    *   $t_1, t_2$: Parameters separating Interval 1 and 2, and Interval 2 and 3, respectively [S1].
    *   $m_1, m_2$: Parameters specifying the smaller region to which Interval 2 is compressed. $t_1$ is warped to $m_1$, and $t_2$ is warped to $m_2$ [S1].
    *   $t_1, t_2, m_1, m_2$ together specify the strength of the edge being constructed [S1].

**Mapping Function:**
The method defines a change of variables mapping $x = \text{map}(x')$, which maps from $[0,1]$ on the vertical axis to $[0,1]$ on the horizontal axis [S1]. The inverse function $p(x) = \text{map}^{-1}(x)$ is used for computation [S1].

1.  **Interval 2 (Compression):** A cosine profile is chosen for the compression curve [S1]:
    $$ p(x) = m_1 + \frac{m_2 - m_1}{2} \left( \cos \left( \left( \frac{x - t_1}{t_2 - t_1} \right) (-\pi) \right) + 1 \right) $$
    [S1]

2.  **Intervals 1 and 3 (Stretching):** A Cubic Spline of the form $p(x) = ax^3 + bx^2 + cx + d$ is used [S1]. The coefficients $a, b, c, d$ are obtained from boundary conditions specifying the function value and the first derivative at the end points [S1].

    *   **For Interval 1:**
        Boundary conditions: $p(0)=0, p'(0)=1, p(t_1)=m_1, p'(t_1)=0$ [S1].
        Derived coefficients:
        $$ a = \frac{-2(m_1 - t_1) - t_1}{t_1^3} $$
        $$ b = \frac{3(m_1 - t_1) + t_1}{t_1^2} $$
        $$ c = 0 $$
        $$ d = 0 $$
        [S1]

    *   **For Interval 3:**
        Boundary conditions: $p(t_2)=m_2, p'(t_2)=0, p(1)=1, p'(1)=1$ [S1].
        Derived coefficients:
        $$ a = \frac{m_2 - t_2}{2 t_2^2} $$
        $$ b = \frac{-2 t_2 + (3 m_2 - 4) t_2 + 3 m_2}{2 t_2^2} $$
        $$ c = \frac{4 t_2 + (-6 m_2 + 7) t_2 - 9 m_2}{2 t_2} $$
        $$ d = \frac{-2 t_2 + (3 m_2 - 6) t_2 + 5 m_2}{2} $$
        [S1]

    *Note on Continuity:* The source states that $C^1$ continuity is preserved because $p'(0)=1$ and $p'(1)=1$ [S1]. However, the provided cubic spline equations for Interval 1 and 3 specify $p'(0)=0$ and $p'(t_1)=0$ and $p'(t_2)=0$ and $p'(1)=0$ in the coefficient derivation context, creating a contradiction in the source text regarding the derivative values at the boundaries [S1]. The implementation should follow the coefficient formulas provided, but be aware of this discrepancy in the source's claim about $C^1$ continuity [S1].

**Splat Transformation:**
Let $g(x,y)$ be the values in the isotropic splat and $h(x,y)$ be the values in the stretched/compressed splat. The transformation depends on the splat type (0, 1, 2, 3) and the side of the edge [S1]. The compression/stretching function is not symmetric, so the side is differentiated by the data value magnitude (larger or lower) [S1].

*   **Splat 0 (Secondary kernel with larger data value):**
    $$ h(x,y) = \begin{cases} g(\text{map}(x - 1) + 1, y) & \text{if } x \ge 1 \\ g(x,y) & \text{else} \end{cases} $$
    [S1]

*   **Splat 1 (Primary kernel with larger data value):**
    $$ h(x,y) = \begin{cases} g(\text{map}(x), y) & \text{if } 0 \le x \le 1 \\ g(x,y) & \text{else} \end{cases} $$
    [S1]

*   **Splat 2 (Primary kernel with lower data value):**
    $$ h(x,y) = \begin{cases} g(\text{map}(1 + x) - 1, y) & \text{if } -1 \le x \le 0 \\ g(x,y) & \text{else} \end{cases} $$
    [S1]

*   **Splat 3 (Secondary kernel with lower data value):**
    $$ h(x,y) = \begin{cases} g(\text{map}(2 + x) - 2, y) & \text{if } x \le -1 \\ g(x,y) & \text{else} \end{cases} $$
    [S1]

**Rendering Steps:**
1.  **Pre-computation:** Build a table of footprints for different possible edge positions and edge strengths using the mapping functions defined above [S1].
2.  **Edge Detection:** For each voxel, determine the relative position to the closest edge [S1].
3.  **Splat Selection:** Pick a footprint from the table for each splat based on the relative position of the voxel to the closest edge [S1].
4.  **Rotation:** Rotate a canonical edge-enhanced splat to handle any edge direction, assuming the edge is locally linear [S1].
5.  **Classification:** Differentiate which side a splat is on by how large its data value is, as the compression/stretching function is not symmetric [S1].
6.  **Compositing:** Render the selected warped footprints using standard splatting compositing [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def compute_cubic_spline_interval_1(t1, m1):
    """
    Computes coefficients a, b, c, d for Interval 1 cubic spline.
    Boundary conditions: p(0)=0, p'(0)=1, p(t1)=m1, p'(t1)=0
    """
    a = (-2 * (m1 - t1) - t1) / (t1 ** 3)
    b = (3 * (m1 - t1) + t1) / (t1 ** 2)
    c = 0
    d = 0
    return a, b, c, d

def compute_cubic_spline_interval_3(t2, m2):
    """
    Computes coefficients a, b, c, d for Interval 3 cubic spline.
    Boundary conditions: p(t2)=m2, p'(t2)=0, p(1)=1, p'(1)=1
    """
    a = (m2 - t2) / (2 * t2 ** 2)
    b = (-2 * t2 + (3 * m2 - 4) * t2 + 3 * m2) / (2 * t2 ** 2)
    c = (4 * t2 + (-6 * m2 + 7) * t2 - 9 * m2) / (2 * t2)
    d = (-2 * t2 + (3 * m2 - 6) * t2 + 5 * m2) / 2
    return a, b, c, d

def p_interval_1(x, t1, m1):
    """
    Inverse map function p(x) for Interval 1.
    """
    a, b, c, d = compute_cubic_spline_interval_1(t1, m1)
    return a * x**3 + b * x**2 + c * x + d

def p_interval_3(x, t2, m2):
    """
    Inverse map function p(x) for Interval 3.
    """
    a, b, c, d = compute_cubic_spline_interval_3(t2, m2)
    return a * x**3 + b * x**2 + c * x + d

def p_interval_2(x, t1, t2, m1, m2):
    """
    Inverse map function p(x) for Interval 2 (Cosine profile).
    """
    return m1 + ((m2 - m1) / 2) * (math.cos(((x - t1) / (t2 - t1)) * (-math.pi)) + 1)

def map_inverse(x, t1, t2, m1, m2):
    """
    Computes p(x) based on which interval x falls into.
    Assumes 0 <= x <= 1.
    """
    if x < t1:
        return p_interval_1(x, t1, m1)
    elif x > t2:
        return p_interval_3(x, t2, m2)
    else:
        return p_interval_2(x, t1, t2, m1, m2)

def get_splat_footprint(splat_type, x, y, base_kernel_g, t1, t2, m1, m2):
    """
    Computes the warped footprint h(x,y) for a given splat type.
    splat_type: 0, 1, 2, or 3
    base_kernel_g: function g(x, y) returning the isotropic splat value
    """
    # Define the rmap function based on splat type
    # rmap(x) maps the coordinate for the base kernel lookup
    if splat_type == 0:
        # Secondary kernel, larger data value
        # h(x,y) = g(map(x-1)+1, y) if x >= 1 else g(x,y)
        if x >= 1:
            mapped_x = map_inverse(x - 1, t1, t2, m1, m2)
            return base_kernel_g(mapped_x + 1, y)
        else:
            return base_kernel_g(x, y)
            
    elif splat_type == 1:
        # Primary kernel, larger data value
        # h(x,y) = g(map(x), y) if 0 <= x <= 1 else g(x,y)
        if 0 <= x <= 1:
            mapped_x = map_inverse(x, t1, t2, m1, m2)
            return base_kernel_g(mapped_x, y)
        else:
            return base_kernel_g(x, y)
            
    elif splat_type == 2:
        # Primary kernel, lower data value
        # h(x,y) = g(map(1+x)-1, y) if -1 <= x <= 0 else g(x,y)
        if -1 <= x <= 0:
            mapped_x = map_inverse(1 + x, t1, t2, m1, m2)
            return base_kernel_g(mapped_x - 1, y)
        else:
            return base_kernel_g(x, y)
            
    elif splat_type == 3:
        # Secondary kernel, lower data value
        # h(x,y) = g(map(2+x)-2, y) if x <= -1 else g(x,y)
        if x <= -1:
            mapped_x = map_inverse(2 + x, t1, t2, m1, m2)
            return base_kernel_g(mapped_x - 2, y)
        else:
            return base_kernel_g(x, y)
            
    else:
        raise ValueError("Invalid splat type")

def render_edge_preserved_splatting(volume_data, edge_position, edge_strength_params, base_kernel):
    """
    Main rendering loop for edge-preserving splatting.
    """
    t1, t2, m1, m2 = edge_strength_params
    
    # Pre-compute footprint table
    # In practice, this would be a 2D array or hash map indexed by edge position/strength
    footprint_table = {}
    
    # Iterate over possible edge positions and strengths to build table
    # For each combination, compute the 4 splat footprints
    for edge_pos in possible_edge_positions:
        for strength in possible_strengths:
            t1, t2, m1, m2 = strength
            footprints = {}
            for splat_type in [0, 1, 2, 3]:
                # Compute footprint grid
                footprint_grid = []
                for x in x_range:
                    row = []
                    for y in y_range:
                        val = get_splat_footprint(splat_type, x, y, base_kernel, t1, t2, m1, m2)
                        row.append(val)
                    footprint_grid.append(row)
                footprints[splat_type] = footprint_grid
            footprint_table[(edge_pos, strength)] = footprints

    # Rendering pass
    for voxel in volume_data:
        # Determine relative position to closest edge
        rel_pos = voxel.position - edge_position
        
        # Classify splat type based on rel_pos and data value magnitude
        # This logic depends on the specific edge geometry and data values
        # Simplified logic for illustration:
        if voxel.data_value > threshold:
            if rel_pos > 1.0:
                splat_type = 0
            else:
                splat_type = 1
        else:
            if rel_pos < -1.0:
                splat_type = 3
            else:
                splat_type = 2
        
        # Retrieve warped footprint from table
        footprints = footprint_table[(edge_position, edge_strength_params)]
        warped_footprint = footprints[splat_type]
        
        # Rotate canonical splat to match edge direction
        rotated_footprint = rotate_footprint(warped_footprint, edge_direction)
        
        # Add to screen buffer
        add_to_screen_buffer(rotated_footprint, voxel.position)
```

## Operating characteristics
*   **Time Complexity:** The method requires pre-computation of a footprint table, which depends on the number of edge positions and strengths. During rendering, for each voxel, a lookup and rotation are performed. The base kernel ensures that only 64 ($4 \times 4 \times 4$) basis functions are involved per point, limiting the computational load per voxel [S1].
*   **Memory Complexity:** The footprint table size depends on the resolution of edge positions and strengths stored. Each entry stores a 2D footprint grid.
*   **Performance:** The method achieves sharp edge details in views close to the volume model, which conventional splatting fails to provide [S1]. It is effective for both synthetic and medical data [S1].
*   **Strengths:** Preserves sharp edge details; local manipulation of kernels; compatible with standard splatting compositing.
*   **Limitations:** Requires pre-computation of footprints; assumes locally linear edges; the asymmetry of the compression/stretching function requires careful handling of splat side differentiation [S1].
*   **When to use:** When sharp edge details are critical, especially in close-up views, and conventional splatting produces unacceptable blurring [S1].
*   **When not to use:** When performance is paramount and edge sharpness is less important, or when edges are highly non-linear and the local linearity assumption breaks down [S1].

## Background
The method was presented by Jian Huang, Roger Crawfis, and Don Stredney from The Ohio State University and the Ohio Supercomputing Center [S1]. It addresses the limitation of conventional splatting algorithms, which produce fuzzy images for views close to the volume model, hindering the study and manipulation of data sets using virtual navigation [S1]. The work builds on the Crawfis and Max kernel [Crawfis93], which was optimized for optimal reconstruction in areas of uniform intensity [S1].

## Sources
[S1] Edge Preservation in Volume Rendering Using Splatting, https://web.eecs.utk.edu/~huangj/papers/edgesplat.pdf, fetched 2026-10-28
