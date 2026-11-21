# Color Specification

Also called: RGBA Color Definition, glColor

## Overview
Color Specification defines the RGBA (Red, Green, Blue, Alpha) representation used to specify color and opacity in rendering pipelines [S1, unofficial]. It solves the problem of defining discrete color values with explicit transparency control, allowing background objects to be seen through semi-transparent foreground objects [S1, unofficial]. It differs from [Voxel Color Mapping](2970-voxel-color-mapping.md) and [Color Blending](5150-color-blending.md) by strictly defining the color state itself rather than combining or mapping it [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: Four scalar values representing Red ($R$), Green ($G$), Blue ($B$), and Alpha ($A$) [S1, unofficial].
- Output: A color specification object or state compatible with the rendering pipeline [S1, unofficial].

**Data Structures:**
The color is represented as a tuple or vector of four floating-point values:
$$ \mathbf{C} = (R, G, B, A) $$
Each component is a float in the range $[0.0, 1.0]$ [S1, unofficial].

**Steps:**
1. Define the Red component $R \in [0.0, 1.0]$ [S1, unofficial].
2. Define the Green component $G \in [0.0, 1.0]$ [S1, unofficial].
3. Define the Blue component $B \in [0.0, 1.0]$ [S1, unofficial].
4. Define the Alpha component $A \in [0.0, 1.0]$ [S1, unofficial].
5. Combine these into the RGBA specification [S1, unofficial].

**Formulas:**
- Alpha $0.0$ indicates a completely transparent color [S1, unofficial].
- Alpha $1.0$ indicates a completely opaque color [S1, unofficial].

**Parameters:**
- $R, G, B$: Intensity of the respective color channels [S1, unofficial].
- $A$: Opacity of the color [S1, unofficial].

**Edge Cases and Pitfalls:**
- When using 3-component specification functions (e.g., `glColor3f`), the Alpha component is tacitly assumed to be $1.0$ (opaque) [S1, unofficial]. This requires explicit handling of alpha when semi-transparent rendering is intended [S1, unofficial].
- Values are restricted to the range $[0.0, 1.0]$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def specify_color(r: float, g: float, b: float, a: float = 1.0) -> tuple:
    """Specifies a color using RGBA components.
    If a is omitted, alpha defaults to 1.0 (opaque) per 3-component convention [S1, unofficial]."""
    return (r, g, b, a)
```

## Subtopics

### [Color Scaling](5082-color-scaling.md)
Adjusts RGB intensity while preserving the alpha channel. It differs from general color transformations by isolating luminance/chromaticity from transparency, keeping perceived opacity constant. Choose when enhancing visual contrast without altering opacity.

## Operating characteristics

## Background
Color specification in OpenGL and tools like Smokeview defines color and opacity to support semi-transparent rendering, allowing background objects to be seen through foreground objects [S1, unofficial]. The RGBA model extends the traditional RGB model by adding an alpha channel for opacity control [S1, unofficial]. The `glColorXXX` family provides the interface, with variants for 3 or 4 components and suffixes indicating data types (`f` for float, `v` for pointer to data) [S1, unofficial].

## Sources
[S1] Smokeview (Version 6) A Tool for Visualizing Fire Dynamics Simulation Data Volume II : Technical Reference Guide, https://doi.org/10.6028/nist.sp.1017-2, fetched 2026-10-26 (unofficial)
