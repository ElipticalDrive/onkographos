# Numerically Stable Perspective Transformation

Also called: Eye-offset perspective, Non-origin eye transformation

## Overview
This method prevents floating-point errors in screen-z values that occur when translating vertices far from the origin in standard perspective projections, which otherwise cause ordering inconsistencies during the compositing of thousands of closely spaced triangles [S1]. It differs from standard perspective transformations by positioning the eye away from the world origin, thereby establishing a geometric basis where relative errors are minimized without requiring double-precision arithmetic [S1].

## How it works
**Inputs and Outputs:**
- Input: Screen-space coordinates ($z_{t1}, z_{t2}$), eye offset parameter ($e_o$), and the angle $\theta$ between the sight line and the screen-z axis.
- Output: World-space distance ($z_{v2} - z_{v1}$) between two points along a sight line.

**Data Structures:**
Not given in the sources.

**Steps:**
1. Shift the camera eye position away from the world origin to establish a new geometric basis [S1].
2. Apply the derived numerically stable equations to convert screen-space coordinates to world-space coordinates [S1].
3. Subdivide large nodal regions on the fly and interpolate data to their corners until nodes cover a reasonably small number of pixels to avoid color-interpolation inaccuracies from hardware Gouraud shading [S1].

**Formulas:**
The core relationship defines the world distance between two points on a sight line as inversely proportional to the cosine of the angle $\theta$ between the sight line and the screen-z axis [S1]:

$$ z_{v2} - z_{v1} = \frac{z_{t2} - z_{t1}}{\cos(\theta)} $$

The specific equation for converting screen-space $z_t$ to world-space $z_v$ differences is given by [S1]:

$$ z_{v2} - z_{v1} = \frac{(z_{t2} - e_o)(z_{t1} - e_o) - (z_{t1} - e_o)(z_{t2} - e_o)}{e_o} $$

*Note: The source text for this formula contains apparent typographical errors in the numerator terms which cancel out to zero in standard algebraic interpretation. The source explicitly states this equation gives the difference in world-z in a numerically stable form [S1]. The variable $e_o$ represents the eye offset.*

**Parameters:**
- $e_o$: Eye offset parameter. Not given in the sources.
- $\theta$: Angle between the sight line and the screen-z axis. Not given in the sources.

**Edge Cases and Pitfalls:**
- Using standard perspective (eye at origin) causes substantial relative floating-point errors in screen-z when vertices are translated far from the origin, leading to ordering inconsistencies during compositing [S1].
- Double precision is rejected due to unacceptable memory overhead [S1].
- Color-interpolation inaccuracies may arise from hardware Gouraud shading if nodal regions are too large; these must be subdivided until nodes cover a reasonably small number of pixels [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def numerically_stable_perspective_transform(z_t1, z_t2, e_o, cos_theta):
    """
    Calculate the world-space distance between two points on a sight line.
    
    Args:
        z_t1 (float): Screen-space z-coordinate of the first point.
        z_t2 (float): Screen-space z-coordinate of the second point.
        e_o (float): Eye offset parameter.
        cos_theta (float): Cosine of the angle between the sight line and screen-z axis.
        
    Returns:
        float: World-space distance (z_v2 - z_v1).
    """
    # Formula derived from source [S1]:
    # z_v2 - z_v1 = (z_t2 - z_t1) / cos(theta)
    
    if cos_theta == 0:
        raise ValueError("Cosine of theta cannot be zero (sight line perpendicular to screen-z axis)")
        
    return (z_t2 - z_t1) / cos_theta
```

## Subtopics

### [Custom Perspective Projection Matrix Construction](2175-custom-perspective-projection-matrix-construction.md)
A method for constructing custom projection matrices. Choose this when matrix-based transformations are preferred over direct coordinate mapping.

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Measured Performance:** Not given in the sources.
- **Strengths:** Avoids floating-point errors in screen-z without the memory overhead of double precision [S1].
- **Limitations:** Requires subdivision of nodal regions to maintain color accuracy with hardware shading [S1].
- **When to use:** When compositing thousands of closely spaced triangles where standard perspective transformations cause ordering inconsistencies due to floating-point errors [S1].
- **When not to use:** When double precision is acceptable and memory overhead is not a concern, or when hardware shading inaccuracies are not a primary concern.

## Background
The method was introduced in a 1996 paper by the authors of [S1] to address issues in hierarchical and parallelizable direct volume rendering for irregular and multiple grids. It was developed to overcome limitations of standard computer graphics software which bases perspective transformations on the eye being at the origin [S1]. The technique was tested on a curvilinear multi-grid and demonstrated performance on space-shuttle grid data [S1].

## Sources
[S1] Hierarchical and parallelizable direct volume rendering for irregular and multiple grids, https://doi.org/10.1109/visual.1996.567606, fetched 2026-10-24
