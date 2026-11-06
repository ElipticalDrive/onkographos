# Color Specification

Also called: RGBA Color Definition, glColor

## Overview
Color Specification defines the RGBA (Red, Green, Blue, Alpha) representation used to map scalar voxel values to visual colors in volume rendering. It solves the problem of translating abstract data values into perceptible color and opacity for display. It differs from [Voxel Color Mapping](2970-voxel-color-mapping.md) by focusing on the fundamental component definition rather than the spatial mapping strategy, and from [Parametric Function Generation](5349-parametric-function-generation.md) by specifying discrete values rather than continuous functions. It builds on [1D Transfer Function Generation and Sampling](2689-1d-transfer-function-generation-and-sampling.md) by providing the atomic color units sampled from the transfer function.

## How it works
**Inputs and Outputs:**
- Input: Four scalar values representing Red ($R$), Green ($G$), Blue ($B$), and Alpha ($A$).
- Output: A color specification object or state compatible with the rendering pipeline.

**Data Structures:**
The color is represented as a tuple or vector of four floating-point values:
$$ \mathbf{C} = (R, G, B, A) $$
Each component is a float in the range $[0.0, 1.0]$.

**Steps:**
1. Define the Red component $R \in [0.0, 1.0]$.
2. Define the Green component $G \in [0.0, 1.0]$.
3. Define the Blue component $B \in [0.0, 1.0]$.
4. Define the Alpha component $A \in [0.0, 1.0]$.
5. Combine these into the RGBA specification.

**Formulas:**
- Alpha 0.0 indicates a completely transparent color [S1, unofficial].
- Alpha 1.0 indicates a completely opaque color [S1, unofficial].

**Parameters:**
- $R, G, B$: Intensity of the respective color channels.
- $A$: Opacity of the color.

**Edge Cases and Pitfalls:**
- When using 3-component specification functions (e.g., `glColor3f`), the Alpha component is tacitly assumed to be 1.0 (opaque) [S1, unofficial]. This can lead to unexpected transparency behavior if semi-transparent rendering is intended but only RGB values are provided.
- Values outside $[0.0, 1.0]$ may be clamped or cause undefined behavior depending on the specific implementation, though the standard definition restricts them to this range [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def specify_color(r: float, g: float, b: float, a: float = 1.0) -> tuple:
    """
    Specifies a color using RGBA components.
    
    Args:
        r: Red component (0.0 to 1.0)
        g: Green component (0.0 to 1.0)
        b: Blue component (0.0 to 1.0)
        a: Alpha component (0.0 to 1.0). Defaults to 1.0 if not provided.
        
    Returns:
        A tuple (r, g, b, a) representing the color specification.
    """
    # Validate range [0.0, 1.0] for all components
    if not (0.0 <= r <= 1.0):
        raise ValueError("Red component must be in [0.0, 1.0]")
    if not (0.0 <= g <= 1.0):
        raise ValueError("Green component must be in [0.0, 1.0]")
    if not (0.0 <= b <= 1.0):
        raise ValueError("Blue component must be in [0.0, 1.0]")
    if not (0.0 <= a <= 1.0):
        raise ValueError("Alpha component must be in [0.0, 1.0]")
        
    return (r, g, b, a)

# Example usage for opaque red
color_opaque_red = specify_color(1.0, 0.0, 0.0)

# Example usage for semi-transparent blue
color_semi_transparent_blue = specify_color(0.0, 0.0, 1.0, 0.5)
```

## Subtopics

### [Color Scaling](5082-color-scaling.md)
Adjusts RGB intensity while preserving alpha. Use when enhancing contrast without changing opacity.

## Operating characteristics
- **Time Complexity:** $O(1)$ per color specification.
- **Memory Complexity:** $O(1)$ per color specification (4 floats).
- **Strengths:** Simple, intuitive, and widely supported by graphics APIs.
- **Limitations:** Does not handle color spaces or gamuts beyond the basic RGB definition. Requires explicit handling of alpha for transparency.
- **When to use:** When defining discrete color values for transfer functions or rendering states.
- **When not to use:** When complex color transformations or gamut mapping are required.

## Background
Color specification in the context of OpenGL and volume rendering tools like Smokeview was formalized to support semi-transparent rendering, allowing background objects to be seen through foreground objects [S1, unofficial]. The RGBA model extends the traditional RGB model by adding an alpha channel for opacity control, enabling more realistic visualizations of volumetric data [S1, unofficial]. The `glColorXXX` family of functions provides the interface for specifying these colors, with variants for 3 or 4 components and different data types (e.g., `f` for float, `v` for vector pointer) [S1, unofficial].

## Sources
[S1] Smokeview (Version 6) A Tool for Visualizing Fire Dynamics Simulation Data Volume II : Technical Reference Guide, https://doi.org/10.6028/nist.sp.1017-2, fetched 2026-10-26 (unofficial)
