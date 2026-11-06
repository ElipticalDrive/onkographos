# Voxel Color Mapping

Also called: Color Index Mapping, Deterministic Voxel Material Assignment, Height-based Voxel Coloring

## Overview
Voxel Color Mapping is a technique that maps scalar data values to integer indices for direct lookup in a color table, translating numerical volume data into visual appearance [S1, S2, unofficial]. It serves as the final step in the classification pipeline, distinct from complex transfer functions that map multiple attributes to opacity and color; instead, it performs a direct lookup based on a normalized input, often used for maximum intensity projection or surface visualization where opacity is fixed or handled separately [S1, S2, unofficial]. Unlike [Parametric Function Generation](5349-parametric-function-generation.md) which computes colors via continuous functions, this method relies on discrete table indexing [S1, unofficial].

## How it works
**Inputs:**
- Scalar voxel value $v$.
- Data bounds $v_{min}$ and $v_{max}$ [S1, unofficial].
- Color table of size 256 entries [S1, unofficial].

**Outputs:**
- RGBA color value.

**Data Structures:**
- **Color Table:** An array of 256 entries. Each entry contains 4 components: Red, Green, Blue, and Opacity. Each component is scaled from 0.0 to 1.0 [S1, unofficial].

**Steps:**
1. **Determine Data Bounds:** Establish $v_{min}$ and $v_{max}$. These can be set by the user, calculated via percentile bounds (e.g., 1st and 99th percentiles from a histogram scan), or set to global data bounds [S1, unofficial].
2. **Compute Color Index:** Map $v$ to an integer index $i$ between 0 and 255 using the formula below.
3. **Lookup Color:** Retrieve the RGBA components from the color table at index $i$.
4. **Display:** Use the retrieved color, potentially via 1D texture maps for interpolation between grid points [S1, unofficial].

**Formula:**
The color index $i$ is computed as [S1, unofficial]:

$$
i = \begin{cases} 
0 & v < v_{min} \\
1 + 253 \frac{v - v_{min}}{v_{max} - v_{min}} & v_{min} \leq v \leq v_{max} \\
255 & v > v_{max}
\end{cases}
$$

**Parameters:**
- Color table size: 256 entries [S1, unofficial].

**Edge Cases and Pitfalls:**
- Values outside $[v_{min}, v_{max}]$ are clamped to indices 0 or 255 [S1, unofficial].
- Precision issues may arise if $v_{min} = v_{max}$; division by zero must be handled [G].
- In height-based variants, specific thresholds (e.g., $h > 0.70$ for grass, $h > 0.40$ for dirt) determine discrete material colors rather than continuous interpolation [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def voxel_color_mapping(v, v_min, v_max, color_table):
    """
    Map a scalar value v to an RGBA color using a 256-entry color table.
    
    Args:
        v (float): The scalar voxel value.
        v_min (float): Minimum data bound.
        v_max (float): Maximum data bound.
        color_table (list of tuples): List of 256 (r, g, b, a) tuples, 
                                      where each component is in [0.0, 1.0].
    
    Returns:
        tuple: (r, g, b, a) color components in [0.0, 1.0].
    """
    # Handle degenerate case where min equals max
    if v_min == v_max:
        # [G] Default to middle of table if bounds are identical
        index = 128
    else:
        # Compute index based on formula [S1, unofficial]
        if v < v_min:
            index = 0
        elif v > v_max:
            index = 255
        else:
            # Linear mapping to range [1, 254]
            # Note: Formula uses 1 + 253 * ... resulting in range [1, 254]
            # for v in [v_min, v_max]. 
            # Indices 0 and 255 are reserved for out-of-bounds.
            index = 1 + 253 * (v - v_min) / (v_max - v_min)
            
    # Clamp index to valid array bounds [G]
    index = int(round(index))
    index = max(0, min(255, index))
    
    # Lookup color
    return color_table[index]

def height_based_voxel_coloring(y, grid_height=64.0):
    """
    Deterministic voxel material assignment based on height [S2, unofficial].
    
    Args:
        y (float): The y-coordinate (height) of the voxel.
        grid_height (float): Total height of the volume grid.
        
    Returns:
        int: Packed RGB color (r << 16) | (g << 8) | b.
    """
    # Normalize height [S2, unofficial]
    h = y / grid_height
    
    if h > 0.70:
        # Grass: (0x4D, 0x9E, 0x2E)
        r, g, b = 0x4D, 0x9E, 0x2E
    elif h > 0.40:
        # Dirt: (0x85, 0x61, 0x38)
        r, g, b = 0x85, 0x61, 0x38
    else:
        # Stone: (0x85, 0x85, 0x85)
        r, g, b = 0x85, 0x85, 0x85
        
    # Pack as (r << 16) | (g << 8) | b [S2, unofficial]
    return (r << 16) | (g << 8) | b
```

## Subtopics

### [Scalar/Component to Color Mapping](6697-scalar-component-to-color-mapping.md)
Scalar/Component to Color Mapping converts a single scalar volume value or a specific component of a surface normal into an RGB color using a 1D colormap texture. It serves as the final step in the classification pipeline, translating numerical data into visual appearance. Unlike complex transfer functions that map multiple attributes to opacity and color, this method performs a direct lookup based on a normalized input, often used for maximum intensity projection or surface visualization where opacity is fixed or handled separately.

### Multi-hypothesis Voxel Coloring
Multi-hypothesis Voxel Coloring identifies a set of hypotheses (possible colorings) for each voxel and narrows them down during a hypothesis removal step, carving inconsistent voxels [S3, unofficial]. This approach, proposed by Eisert et al. [ESG99], allows for the reconstruction of semi-transparent voxels (e.g., Roxels [DebVio99]) and can be refined using volumetric optimization methods like greedy or simulated annealing [S3, unofficial]. It requires accurately calibrated camera inputs, though approximate space carving extensions can handle calibration errors [S3, unofficial].

## Operating characteristics
- **Time Complexity:** $O(1)$ per voxel for index computation and table lookup [S1, unofficial].
- **Memory Complexity:** $O(N)$ for the color table, where $N=256$ [S1, unofficial].
- **Strengths:** Simple implementation; efficient for real-time visualization; leverages hardware texture interpolation [S1, unofficial].
- **Limitations:** Limited to scalar data; discrete indexing may cause banding without interpolation; opacity must be handled separately or fixed [S1, S2, unofficial].
- **When to use:** When visualizing scalar fields with fixed opacity or when performance is critical and complex transfer functions are unnecessary [S1, S2, unofficial].
- **When not to use:** When multi-attribute classification or complex opacity variations are required [S1, S2, unofficial].

## Background
Voxel Color Mapping originated in tools like Smokeview for visualizing Fire Dynamics Simulation data, where it was used to convert quantitative data into color using 1D texture maps [S1, unofficial]. The technique has been extended in volumetric reconstruction research, such as multi-hypothesis voxel coloring by Eisert et al. [ESG99], which introduced hypothesis-based coloring for space carving [S3, unofficial].

## Sources
[S1] Smokeview (Version 6) A Tool for Visualizing Fire Dynamics Simulation Data Volume II : Technical Reference Guide, https://doi.org/10.6028/nist.sp.1017-2, fetched 2026-10-26 (unofficial)
[S2] scarlyle0/voxel-compare, https://github.com/scarlyle0/voxel-compare, fetched 2026-10-27 (unofficial)
[S3] Accelerated volumetric reconstruction from uncalibrated camera views, http://doras.dcu.ie/157/, fetched 2026-10-26 (unofficial)
