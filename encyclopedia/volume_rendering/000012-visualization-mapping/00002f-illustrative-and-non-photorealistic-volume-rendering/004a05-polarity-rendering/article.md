# Polarity Rendering


## Overview
Polarity rendering visualizes volumetric data by coloring voxels based on the sign (positive or negative) of a selected field component basis ($B_x, B_y,$ or $B_z$) [S1, unofficial]. It distinguishes regions of positive and negative orientation along a chosen axis, unlike methods that rely on magnitude or surface normals [S1, unofficial].

## How it works
**Inputs:**
1. A volumetric field dataset where a component basis is selected [S1, unofficial].
2. Two distinct colors: $c_{\text{pos}}$ for positive values and $c_{\text{neg}}$ for negative values [G].

**Outputs:**
* A rendered image where each pixel color is determined by the sign of the selected component at the corresponding sample location [S1, unofficial].

**Data Structures:**
* Not given in the sources.

**Steps:**
1. **Preprocessing:** Replace any `NaN` or `Inf` values in the input field with zero to prevent rendering artifacts at array boundaries [S1, unofficial].
2. **Component Selection:** Extract the scalar component $C$ corresponding to the chosen basis ($B_x, B_y,$ or $B_z$) [S1, unofficial].
3. **Sign Evaluation:** For each sample point $p$, evaluate the sign of $C(p)$:
   $$ \text{sign}(C(p)) = \begin{cases} +1 & \text{if } C(p) > 0 \\ -1 & \text{if } C(p) < 0 \\ 0 & \text{if } C(p) = 0 \end{cases} $$
4. **Color Mapping:** Assign the output color $c_{\text{out}}$ based on the sign:
   $$ c_{\text{out}} = \begin{cases} c_{\text{pos}} & \text{if } \text{sign}(C(p)) > 0 \\ c_{\text{neg}} & \text{if } \text{sign}(C(p)) < 0 \\ c_{\text{zero}} & \text{if } \text{sign}(C(p)) = 0 \end{cases} $$

**Parameters:**
* **Component Basis:** The specific axis ($B_x, B_y,$ or $B_z$) to evaluate [S1, unofficial].
* **Typical Values:** Not given in the sources.

**Edge Cases and Pitfalls:**
* **Zero Crossings:** The behavior at exactly zero is not explicitly defined in the primary source [S1, unofficial]. Implementers should define a default color or transparency for $C(p)=0$ [G].
* **Incomplete Topology:** Polarity rendering does not represent full magnetic topology, connectivity class, or radial polarity unless additional modes are implemented [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def polarity_render(field, component_index, color_pos, color_neg, color_zero):
    """
    Renders a volume based on the sign of a specific field component.
    Args:
        field: 3D list/array of shape (D, H, W, 3) containing vector components.
        component_index: Integer 0, 1, or 2 corresponding to x, y, or z.
        color_pos: Tuple/list of 3 floats for positive values.
        color_neg: Tuple/list of 3 floats for negative values.
        color_zero: Tuple/list of 3 floats for zero values.
    Returns:
        3D list of shape (D, H, W, 3) with colored voxels.
    """
    output = []
    for z in range(len(field)):
        row = []
        for y in range(len(field[0])):
            col = []
            for x in range(len(field[0][0])):
                val = field[z][y][x][component_index]
                if val > 0:
                    col.append(color_pos)
                elif val < 0:
                    col.append(color_neg)
                else:
                    col.append(color_zero)
            row.append(col)
        output.append(row)
    return output
```

## Subtopics

### Scalar Value Assignment for Polarity
This variant is used when the input data is not a native vector field but rather scalar data representing different polarities, such as radar cross-section (RCS) data [S2, unofficial]. Instead of evaluating the sign of a vector component, distinct scalar values are assigned to each polarity type [S2, unofficial]. Vertical polarity (W) and horizontal polarities (HH) are given unique values, while cross-pole polarities (HV and VH) are assigned the same scalar weight [S2, unofficial]. A spacing of 50 units between scalar values is recommended to ensure accurate representation and avoid bias [S2, unofficial]. A key pitfall is that closely spaced scalar values or values close to zero significantly bias the final results [S2, unofficial].

## Operating characteristics
* **Time Complexity:** Not given in the sources.
* **Memory Complexity:** Not given in the sources.
* **Strengths:** The method is computationally inexpensive and effectively highlights directional asymmetry along a chosen axis [S1, unofficial].
* **Limitations:** It does not convey full vector direction, magnitude, or topological connectivity [S1, unofficial].
* **When to use:** When the primary interest is distinguishing regions of positive vs. negative orientation along a specific axis [S1, unofficial].
* **When not to use:** When full vector direction, magnitude, or topological connectivity is required [S1, unofficial].

## Background
Polarity rendering was introduced in the context of astrophysical vector field visualization, specifically for magnetograms [S1, unofficial]. It was developed to provide a native volumetric visualization method that could quickly indicate magnetic polarity in solar physics data [S1, unofficial]. The method is also applied in radar cross-section (RCS) data visualization to facilitate three-dimensional interpretation [S2, unofficial].

## Sources
[S1] VitorFigueiredoResearch/AstroSplat-Browser-native-volumetric-visualization-for-astrophysical-vector-fields, https://github.com/VitorFigueiredoResearch/AstroSplat-Browser-native-volumetric-visualization-for-astrophysical-vector-fields, fetched 2026-10-26 (unofficial)
[S2] DTIC ADA258905: Methods for Viewing Radar Cross Section Data in Three Dimensions, https://archive.org/details/DTIC_ADA258905, fetched 2026-10-26 (unofficial)
