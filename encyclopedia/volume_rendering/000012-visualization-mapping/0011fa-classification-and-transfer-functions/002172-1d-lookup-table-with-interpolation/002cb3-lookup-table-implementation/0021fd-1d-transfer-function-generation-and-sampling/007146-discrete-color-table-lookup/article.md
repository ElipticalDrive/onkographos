# Discrete Color Table Lookup

Also called: Discrete LUT, Index-based Color Lookup, L(d), Linear Interpolation Lookup, Linear TF, Linear Transfer Function

## Overview
Discrete Color Table Lookup maps volumetric scalar data values to RGBA color and opacity using a user-defined lookup table (LUT) [S1, unofficial]. The user specifies a minimum and maximum data value that correspond to the first and last entries of the table, respectively, with linear interpolation applied between these bounds [S1, unofficial]. This method is distinguished from sibling techniques like [Parametric Function Generation](5349-parametric-function-generation.md) by its reliance on a fixed-size discrete array rather than continuous mathematical functions, and from [2D Transfer Function](7592-2d-transfer-function.md) by its restriction to scalar-to-color mapping without joint density or gradient dependencies.

## How it works
**Inputs:**
- `dataValue`: The scalar value of the current voxel or sample point.
- `minValue`: The minimum data value in the volume (maps to index 0).
- `maxValue`: The maximum data value in the volume (maps to the last index).
- `colorTable`: An array of RGBA color/opacity structures. The length is $N$.

**Outputs:**
- `color`: The RGBA color corresponding to `dataValue`.
- `opacity`: The opacity corresponding to `dataValue`.

**Data Structures:**
- `colorTable`: Array of size $N$. Each element contains 4 floating-point values (R, G, B, A) in the range [0, 1].

**Steps:**
1. Calculate the step size between table entries:
   $$ step = \frac{maxValue - minValue}{N - 1} $$
2. Round the input `dataValue` to the nearest step boundary to determine the discrete index:
   $$ value_{rounded} = round\left(\frac{dataValue}{step}\right) \cdot step $$
3. Calculate the integer index into the table:
   $$ index = \left\lfloor \frac{-minValue + value_{rounded}}{step} \right\rfloor $$
   *Note: The source formula uses `round(value / step) * step` before subtracting `minValue`. This effectively snaps the value to the nearest grid point defined by the step size.*
4. Clamp `index` to the range $[0, N-1]$.
5. Return `colorTable[index]`.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def discrete_color_table_lookup(data_value, min_value, max_value, color_table):
    """
    Maps a scalar data value to a color/opacity entry in a discrete lookup table.
    
    Args:
        data_value (float): The scalar value to map.
        min_value (float): The minimum data value (maps to index 0).
        max_value (float): The maximum data value (maps to index len(color_table)-1).
        color_table (list of tuples): List of (r, g, b, a) tuples.
        
    Returns:
        tuple: (r, g, b, a) from the table, or None if out of bounds.
    """
    n = len(color_table)
    if n < 2:
        return None
        
    # Calculate step size
    step = (max_value - min_value) / (n - 1)
    
    # Handle edge case where min == max to avoid division by zero
    if step == 0:
        return color_table[0]
        
    # Round value to nearest step
    # Note: Source [S2, unofficial] uses round(value / step) * step. 
    # This snaps the value to the nearest discrete level.
    value_rounded = round(data_value / step) * step
    
    # Calculate index
    # Source [S2, unofficial] formula: index = floor((-minValue + valueRoundedToNearestStep) / step)
    index = int((-min_value + value_rounded) / step)
    
    # Clamp index to valid range
    if index < 0:
        index = 0
    elif index >= n:
        index = n - 1
        
    return color_table[index]
```

**Parameters:**
- `minValue`, `maxValue`: Define the dynamic range of the volume. Typical values depend on the dataset (e.g., Hounsfield units in CT: -1000 to 3000).
- `colorTable`: User-defined. Size $N$ is typically 256 or 1024.

**Edge Cases and Pitfalls:**
- **Precision:** Floating-point rounding errors in `step` calculation can cause index misalignment. Using `round()` as specified in [S2, unofficial] mitigates this by snapping to the nearest grid point.
- **Boundaries:** Values outside `[minValue, maxValue]` must be clamped to the first or last entry to prevent out-of-bounds access.
- **Degenerate Input:** If `minValue == maxValue`, `step` becomes 0. The implementation must handle this by returning the first (or only) table entry.

## Subtopics

### [Voxel Color Mapping](2970-voxel-color-mapping.md)
General mapping of voxel values to colors; Discrete Color Table Lookup is a specific implementation using a fixed LUT.

### [Color Specification](3969-color-specification.md)
Defines the color space (e.g., sRGB, linear); Discrete Color Table Lookup assumes a specific color space for its entries.

### [Gradient/Color Ramp Sampling](4337-gradient-color-ramp-sampling.md)
Uses continuous ramps; Discrete Color Table Lookup uses discrete indices.

### [Color Blending](5150-color-blending.md)
Combines multiple colors; Discrete Color Table Lookup retrieves a single color.

### [Parametric Function Generation](5349-parametric-function-generation.md)
Uses mathematical functions; Discrete Color Table Lookup uses a pre-computed array.

### [Transfer Function Editor (Alpha Pins)](6772-transfer-function-editor-alpha-pins.md)
UI for defining transfer functions; Discrete Color Table Lookup is the underlying sampling method.

### [Colormap LUT Generation](7229-colormap-lut-generation.md)
Creates the LUT; Discrete Color Table Lookup consumes it.

### [2D Transfer Function](7592-2d-transfer-function.md)
Maps two dimensions; Discrete Color Table Lookup maps one.

### [Bipolar Scalar-to-Color Mapping with Level Set Support](13036-bipolar-scalar-to-color-mapping-with-level-set-support.md)
Handles positive/negative values; Discrete Color Table Lookup is typically unipolar unless extended.

### [Color Table Filtering](16454-color-table-filtering.md)
Smooths the LUT; Discrete Color Table Lookup uses the raw LUT.

### [Discrete Terrain Color Mapping](17937-discrete-terrain-color-mapping.md)
Specific to terrain; Discrete Color Table Lookup is general.

### [1D Texture Color Mapping](18347-1d-texture-color-mapping.md)
Uses texture hardware; Discrete Color Table Lookup can be implemented in CPU or GPU shader without texture units.

## Operating characteristics
- **Time Complexity:** $O(1)$ per sample. The lookup involves a constant number of arithmetic operations and an array access.
- **Memory Complexity:** $O(N)$ for the color table, where $N$ is the table size.
- **Performance:** Extremely fast on both CPU and GPU. On GPU, it can be implemented as a simple array lookup or texture fetch.
- **Strengths:** Simple to implement, user-friendly (direct control over color/opacity at specific values), efficient.
- **Limitations:** Limited resolution (depends on $N$), discontinuities if interpolation is not used (though this method uses rounding to nearest step, it is effectively discrete), less flexible than parametric functions for complex mappings.
- **When to use:** When performance is critical, or when the user wants direct control over specific data values.
- **When not to use:** When smooth, continuous transitions are required without aliasing artifacts from discrete steps, or when the mapping is too complex for a 1D table.

## Background
The method was described in the context of analytic rendering of curvilinear volume data [S1, unofficial]. It provides a convenient method for mapping data values to color and opacity with user-controlled parameters. The transfer function is denoted as $L(d)$ where $d$ is the input data value. The approach was also implemented in WebGL radar rendering applications [S2, unofficial], where it was used to map radar data to color tables for visualization.

## Sources
[S1] Analytic rendering of curvilinear volume data, https://doi.org/10.2172/10112506, fetched 2026-10-26 (unofficial)
[S2] amitukind/webgl-radar-rendering, https://github.com/amitukind/webgl-radar-rendering, fetched 2026-10-27 (unofficial)
