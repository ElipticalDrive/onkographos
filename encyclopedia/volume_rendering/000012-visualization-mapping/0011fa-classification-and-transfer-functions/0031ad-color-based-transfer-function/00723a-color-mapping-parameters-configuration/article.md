# Color Mapping Parameters Configuration

Also called: Advanced Color Map Properties

## Overview
Configures the low-level rendering behavior of a color transfer function, specifically defining the interpolation space, handling of out-of-bounds data, and discretization. It sets apart from [Windowing](6611-windowing.md) by controlling the *quality* and *space* of the color gradient rather than the scalar range mapping, and from [Colormap Application for Visualization](7714-colormap-application-for-visualization.md) by providing explicit control over NaNs and range clamping.

## How it works
**Inputs:**
- A color transfer function object (LUT).
- Configuration parameters for interpolation, range handling, and discretization.

**Outputs:**
- Updated color mapping behavior for the associated data array.

**Data Structures:**
The transfer function maintains a list of control points (scalar value, color, opacity). The configuration parameters modify how these points are interpolated or clamped.

**Parameters:**
1.  **Color Interpolation Space**: Defines the mathematical space in which color blending occurs. Options: `RGB`, `HSV`, `Lab`, `Diverging`, `Lab/CIEDE2000` [S1].
2.  **Use Above Range Color**: Boolean flag. When enabled, data values exceeding the maximum scalar range of the transfer function are mapped to a user-defined special color instead of the maximum color [S1].
3.  **Use Below Range Color**: Boolean flag. When enabled, data values below the minimum scalar range are mapped to a user-defined special color [S1].
4.  **NaN Color and Opacity**: Specific color and opacity values assigned to `NaN` (Not a Number) entries in floating-point arrays [S1].
5.  **Discretization**: Boolean or integer flag. Determines if the transfer function uses smooth interpolation between control points or quantizes the output into a fixed number of discrete colors [S1].

**Steps:**
1.  Access the color transfer function object for the target array.
2.  Set the interpolation space property to the desired color model (e.g., `HSV` for perceptual uniformity in hue, `Lab` for perceptual uniformity in lightness) [S1].
3.  If `Use Above Range Color` is enabled, assign the specific color and opacity for values $> V_{max}$ [S1].
4.  If `Use Below Range Color` is enabled, assign the specific color and opacity for values $< V_{min}$ [S1].
5.  If the data contains `NaN` values, assign the specific `NaN Color` and `NaN Opacity` [S1].
6.  Set the discretization mode. If discretizing, specify the number of colors or enable the fixed-color mode [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Access the color transfer function for array 'RTData'
# Note: Implementation details depend on the specific API (e.g., ParaView's pvpython)
lut = GetColorTransferFunction('RTData')

# 1. Set Interpolation Space
# Options: 'RGB', 'HSV', 'Lab', 'Diverging', 'Lab/CIEDE2000'
lut.InterpolationType = 'HSV' 

# 2. Configure Out-of-Range Colors
# Enable clamping colors
lut.UseBelowRangeColor = True
lut.BelowRangeColor = [0.0, 0.0, 0.0, 1.0] # Example: Black
lut.BelowRangeOpacity = 1.0

lut.UseAboveRangeColor = True
lut.AboveRangeColor = [1.0, 1.0, 1.0, 1.0] # Example: White
lut.AboveRangeOpacity = 1.0

# 3. Configure NaN Handling
lut.NanColor = [1.0, 0.0, 0.0, 1.0] # Example: Red
lut.NanOpacity = 1.0

# 4. Configure Discretization
# If the API supports discrete mapping, enable it
# Note: Specific property name for discretization count varies by implementation
lut.Discretize = True 
# lut.NumberOfColors = 10 # If applicable
```

**Edge Cases and Pitfalls:**
- **Precision**: Interpolation in `RGB` space may produce perceptually non-uniform gradients (e.g., muddy colors in mid-tones). `Lab` or `HSV` is preferred for perceptual accuracy [S1].
- **NaN Handling**: If the data type is integer, `NaN` values do not exist, and this setting is ignored [S1].
- **Range Clamping**: If `Use Above Range Color` is disabled, values above the range are clamped to the maximum color, which may hide data intensity variations [S1].

## Subtopics

### Separate Color Map
A feature that forces the visualization system to use a color map uniquely associated with the array name AND the active representation, rather than just the array name. This allows different representations of the same data array to have different color maps. To enable, set the `UseSeparateColorMap` property to `True` on the display object or use the `separate=True` argument in the `ColorBy` interface. This configuration is saved across sessions [S1].

## Operating characteristics
- **Time Complexity**: Negligible overhead during rendering; configuration is applied at transfer function update time.
- **Memory Complexity**: Minimal; stores a few additional color/opacity tuples for range clamping and NaN handling.
- **Strengths**: Provides precise control over visual artifacts (NaNs, out-of-bounds) and perceptual quality (interpolation space).
- **Limitations**: Does not affect transfer functions already setup if changed after initial creation (requires re-application) [S1].
- **When to use**: When standard RGB interpolation produces poor visual results, or when specific handling for extreme values or missing data is required.

## Background
Originated in ParaView's color mapping module to provide advanced users with fine-grained control over how scalar data is mapped to RGBA values. It addresses the limitations of default linear RGB interpolation and lack of explicit NaN/range handling in basic transfer functions [S1].

## Sources
[S1] 2. Color maps and transfer functions — ParaView ..., https://cep.tacc.utexas.edu/user-guide/ReferenceManual/colorMapping.html, fetched 2026-10-29
