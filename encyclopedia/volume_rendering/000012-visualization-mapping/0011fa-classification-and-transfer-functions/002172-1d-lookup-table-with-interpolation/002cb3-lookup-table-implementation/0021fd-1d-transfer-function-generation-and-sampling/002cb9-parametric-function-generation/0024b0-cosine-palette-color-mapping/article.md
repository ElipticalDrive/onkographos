# Cosine Palette Color Mapping
Also called: Inigo Quilez Palette
## Overview
Cosine Palette Color Mapping generates a smooth color gradient from a scalar input using a cosine function with specific parameters, as described by Inigo Quilez [S1, unofficial]. It maps a normalized scalar input $t \in [0, 1]$ to an RGBA output vector $\mathbf{C}$ using a parametric equation [G]. Unlike sibling methods that rely on probability distributions or piecewise linear interpolation, this approach uses trigonometric periodicity to define color transitions [G].
## How it works
**Inputs and Outputs:**
- Input: A normalized scalar value $t \in [0, 1]$ [G].
- Output: An RGBA color vector $\mathbf{C}$ where the RGB channels are computed via the cosine palette and the Alpha channel is set to $1.0$ unless blended with other transfer function logic [G].

**Data Structures:**
- `vec3 a`: Base color offset [G].
- `vec3 b`: Amplitude scaling [G].
- `vec3 c`: Frequency scaling [G].
- `vec3 d`: Phase shift [G].

**Formula:**
The color $\mathbf{C}_{\text{rgb}}$ is calculated as:
$$ \mathbf{C}_{\text{rgb}} = \mathbf{a} + \mathbf{b} \cdot \cos(2\pi \cdot (\mathbf{c} \cdot t + \mathbf{d})) $$

Where:
- $\mathbf{a} = (0.5, 0.5, 0.5)$ [S1, unofficial]
- $\mathbf{b} = (0.5, 0.5, 0.5)$ [S1, unofficial]
- $\mathbf{c} = (1.0, 1.0, 1.0)$ [S1, unofficial]
- $\mathbf{d} = (0.00, 0.33, 0.67)$ [S1, unofficial]
- $\cos$ is applied component-wise [G].
- $t$ is the scalar input [G].

**Parameters:**
- The parameters $\mathbf{a}, \mathbf{b}, \mathbf{c}, \mathbf{d}$ are fixed constants defined by Inigo Quilez [S1, unofficial].
- $t$ must be clamped to $[0, 1]$ to prevent out-of-bounds evaluation if raw values are not pre-normalized [G].

**Edge Cases and Pitfalls:**
- **Precision:** Floating-point precision errors in $t$ may cause slight color banding if the output is quantized to 8-bit integers without dithering [G].
- **Boundaries:** If $t$ exceeds $[0, 1]$, the color will wrap around due to the periodic nature of the cosine function [G]. Clamping $t$ is essential for monotonic data mapping [G].
- **Alpha:** This method defines RGB only. Alpha must be handled separately, often by setting it to $1.0$ or blending with opacity transfer functions [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def cosine_palette(t: float) -> tuple[float, float, float, float]:
    # Clamp input to [0, 1] to prevent wrapping artifacts
    t = max(0.0, min(1.0, t))
    
    # Define vector parameters
    a = (0.5, 0.5, 0.5)
    b = (0.5, 0.5, 0.5)
    c = (1.0, 1.0, 1.0)
    d = (0.00, 0.33, 0.67)
    
    # Compute RGB using component-wise cosine
    r = a[0] + b[0] * math.cos(2 * math.pi * (c[0] * t + d[0]))
    g = a[1] + b[1] * math.cos(2 * math.pi * (c[1] * t + d[1]))
    blue = a[2] + b[2] * math.cos(2 * math.pi * (c[2] * t + d[2]))
    
    # Alpha is not defined by the palette formula; set to 1.0 [G]
    alpha = 1.0
    
    return (r, g, blue, alpha)
```
## Subtopics

### [Blue-Red Colormap](3921-blue-red-colormap.md)
The Blue-Red Colormap normalizes scalar depth values to the range $[0, 1]$ and maps them to a piecewise linear color gradient ranging from blue at minimum depth to red at maximum depth. It solves the problem of interpreting 3D spatial structure from rendered depth buffers by providing a high-contrast continuous representation. Unlike the parent Cosine Palette Color Mapping which generates smooth gradients via trigonometric functions, this method uses explicit linear piecewise functions optimized for depth visualization.

### [Hue Transfer Function](4574-hue-transfer-function.md)
The Hue Transfer Function is a procedural method that maps a scalar input value to an RGB color by cycling through hue bands using smoothstep functions. It serves as a specific implementation of color mapping within volume rendering classification, distinct from parent [Cosine Palette Color Mapping](3359-cosine-palette-color-mapping.md) by using piecewise linear interpolation (smoothstep) rather than trigonometric functions to define color transitions.

## Operating characteristics
**Time and Memory Complexity:**
- Time: $O(1)$ per voxel, involving a constant number of arithmetic operations and trigonometric evaluations [G].
- Memory: $O(1)$, requiring only storage for the four vector parameters and the input scalar [G].

**Measured Performance:** Not given in the sources. [S1, unofficial]

**Scaling:** Not given in the sources. [S1, unofficial]

**Strengths and Limitations:**
- **Strengths:** Not given in the sources. [S1, unofficial]
- **Limitations:** Not given in the sources. [S1, unofficial]

**When to Use:** Not given in the sources. [S1, unofficial]
## Background
**Origin:**
- The method was described by Inigo Quilez [S1, unofficial].

**History and Development:** Not given in the sources. [S1, unofficial]

**Where it is Used:** Not given in the sources. [S1, unofficial]
## Sources
[S1] Show HN: Volume rendering 3D data in Three.js and GLSL, https://github.com/SuboptimalEng/volume-rendering, fetched 2026-10-26 (unofficial)
