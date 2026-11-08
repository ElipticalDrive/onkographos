# Categorical Color Maps

Also called: Discrete Coloring, Interpret Values As Categories

## Overview
Categorical color maps are a visualization technique for rendering data arrays containing discrete enumerations (e.g., element types, gear types) rather than continuous scalar fields [S1]. They solve the problem that traditional interpolated color transfer functions are tedious and cumbersome for discrete data, where users would otherwise need to manually configure multiple control points and discrete steps [S1]. Unlike continuous methods, this approach maps specific data values to exact colors using a lookup table, rendering non-matching values with a designated NaN color [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A data array containing discrete identifiers (typically integers, but potentially floating-point values) and a user-defined mapping of these values to colors and annotations [S1].
*   **Output:** A rendered image where each element displays the color associated with its exact data value, or the NaN color if no match is found [S1].

**Data Structures:**
The method relies on two primary user-configurable lists within the color map object:
1.  **Annotations:** A flattened list of tuples mapping data values to annotation text strings. Format: `[value1, text1, value2, text2, ...]` [S1].
2.  **IndexedColors:** An ordered list of flattened RGB tuples defining the colors for each category. The first RGB triplet corresponds to the first annotation, the second to the second, etc. Format: `[r1, g1, b1, r2, g2, b2, ...]` [S1].

**Parameters:**
*   **InterpretValuesAsCategories:** A boolean flag (or integer `1`) that enables categorical mode in the color map editor [S1]. When enabled, the standard mapping data controls are hidden, and the annotations widget becomes visible [S1].
*   **NaN Color:** A fallback color used for any data value that does not have an exact match in the lookup table [S1].

**Algorithm:**
1.  Enable categorical mode by setting `InterpretValuesAsCategories` to true [S1].
2.  Define the set of discrete values and their corresponding colors by populating the `Annotations` and `IndexedColors` lists [S1].
3.  For each rendered element (voxel, point, or cell) with data value $v$:
    a.  Check if $v$ exists in the `Annotations` list [S1].
    b.  If an exact match is found at index $i$ (where the annotation is `Annotations[2*i]`), retrieve the color from `IndexedColors` at the corresponding RGB triplet index [S1].
    c.  Render the element using the retrieved color [S1].
    d.  If no exact match is found, render the element using the designated NaN color [S1].
4.  Render the color legend (scalar bar) as a series of swatches with annotations for each value in the lookup table, rather than a continuous gradient bar [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_categorical_element(data_value, annotations, indexed_colors, nan_color):
    """
    Renders a single element using categorical color mapping.
    
    Args:
        data_value: The discrete identifier of the element.
        annotations: List of [val1, text1, val2, text2, ...]
        indexed_colors: List of [r1, g1, b1, r2, g2, b2, ...]
        nan_color: Fallback color (r, g, b)
        
    Returns:
        Color tuple (r, g, b) to render the element.
    """
    # Iterate through annotations to find an exact match
    # Annotations are stored as [value1, text1, value2, text2, ...]
    # We only care about the values at even indices: 0, 2, 4, ...
    for i in range(0, len(annotations), 2):
        category_value = annotations[i]
        
        # Check for exact match
        # Note: Floating point comparison may fail due to precision issues [S1]
        if data_value == category_value:
            # Calculate index in indexed_colors
            # Each category uses 3 floats (R, G, B)
            color_index = i * 3  # Wait, i is index in annotations, not categories.
            # If i is 0 (1st category), color index is 0.
            # If i is 2 (2nd category), color index is 3.
            # So color_index = i * 3 / 2 * 3? No.
            # i is the index in the flattened annotations list.
            # The number of categories is len(annotations) / 2.
            # The category index is i / 2.
            # The color index is (i / 2) * 3.
            
            cat_idx = i // 2
            color_start = cat_idx * 3
            
            r = indexed_colors[color_start]
            g = indexed_colors[color_start + 1]
            b = indexed_colors[color_start + 2]
            
            return (r, g, b)
            
    # No match found, return NaN color
    return nan_color
```

**Edge Cases and Pitfalls:**
*   **Floating-Point Precision:** When using floating-point data arrays, exact value matching may fail due to precision differences between the annotation value and the dataset value, even if they appear identical. In this case, the NaN color is used instead of the specified category color [S1].
*   **Data Type Mismatch:** The method is designed for enumerations, typically integer arrays. Using it with floating-point numbers requires careful handling of precision [S1].

## Operating characteristics
*   **Time Complexity:** Lookup is $O(N)$ where $N$ is the number of defined categories, assuming a linear search through the annotations list. For large numbers of categories, a hash map or binary search could optimize this, but the source describes a direct lookup [S1].
*   **Memory Complexity:** Proportional to the number of categories defined in `Annotations` and `IndexedColors` [S1].
*   **Strengths:** Provides an elegant and simple solution for visualizing discrete data without the complexity of configuring continuous transfer functions [S1]. The legend automatically adapts to show discrete swatches [S1].
*   **Limitations:** Not suitable for continuous data. Precision issues with floating-point data can lead to unexpected NaN rendering [S1].
*   **When to use:** When the data represents distinct categories (e.g., material types, labels) rather than a continuous physical quantity [S1].

## Background
Categorical color maps are a feature of the ParaView visualization tool, introduced to address the limitations of standard scalar coloring for discrete data [S1]. The method is implemented in the ParaView Reference Manual, specifically in section 4.6 [S1].

## Sources
[S1] 4. Color maps and transfer functions, https://docs.paraview.org/en/latest/ReferenceManual/colorMapping.html, fetched 2026-10-29
