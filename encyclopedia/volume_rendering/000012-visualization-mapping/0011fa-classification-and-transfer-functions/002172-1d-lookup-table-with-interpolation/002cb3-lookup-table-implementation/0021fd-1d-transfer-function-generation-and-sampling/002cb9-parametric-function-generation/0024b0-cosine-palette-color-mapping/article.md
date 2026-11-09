# Cosine Palette Color Mapping
Also called: Inigo Quilez Palette
## Overview
Cosine Palette Color Mapping generates a smooth color gradient from a scalar input using a cosine function with specific parameters, as described by Inigo Quilez [S1, unofficial]. It differs from the [Gaussian Transfer Function](4408-gaussian-transfer-function.md) and [Sigmoid Function with Jacobian](6497-sigmoid-function-with-jacobian.md) by using trigonometric interference rather than additive basis functions or localized peaks [G].
## How it works
**Inputs and Outputs:**
- Input: A normalized scalar value $t \in [0, 1]$ [G].
- Output: An RGBA vector $\mathbf{C} = (R, G, B, A)$ [G].

**Data Structures:**
The method uses four parameter vectors $\mathbf{a}, \mathbf{b}, \mathbf{c}, \mathbf{d}$, each of size 3 representing the Red, Green, and Blue channels [S1, unofficial].
- $\mathbf{a} = (a_r, a_g, a_b)$: Offset vector.
- $\mathbf{b} = (b_r, b_g, b_b)$: Amplitude vector.
- $\mathbf{c} = (c_r, c_g, c_b)$: Frequency vector.
- $\mathbf{d} = (d_r, d_g, d_b)$: Phase offset vector.

**Formula:**
The RGB color is computed component-wise as:
$$ \mathbf{C}_{rgb} = \mathbf{a} + \mathbf{b} \odot \cos(2\pi (\mathbf{c} \cdot t + \mathbf{d})) $$
where $\odot$ denotes element-wise multiplication and $\cdot$ denotes scalar-vector multiplication [S1, unofficial].

**Parameters:**
The canonical parameters are:
- $\mathbf{a} = (0.5, 0.5, 0.5)$
- $\mathbf{b} = (0.5, 0.5, 0.5)$
- $\mathbf{c} = (1.0, 1.0, 1.0)$
- $\mathbf{d} = (0.00, 0.33, 0.67)$ [S1, unofficial].

**Steps:**
1. Clamp the input scalar $t$ to the range $[0, 1]$ [G].
2. For each channel $k \in \{r, g, b\}$:
   a. Calculate the argument: $arg_k = 2\pi (c_k t + d_k)$ [S1, unofficial].
   b. Evaluate the cosine: $val_k = \cos(arg_k)$ [S1, unofficial].
   c. Scale and offset: $C_k = a_k + b_k val_k$ [S1, unofficial].
3. Clamp each resulting channel $C_k$ to $[0, 1]$ to handle numerical precision or out-of-bounds results [G].
4. Assign a constant alpha value $A = 1.0$ [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def cosine_palette(t: float) -> tuple[float, float, float, float]:
    # Canonical parameters
    a = (0.5, 0.5, 0.5)
    b = (0.5, 0.5, 0.5)
    c = (1.0, 1.0, 1.0)
    d = (0.0, 0.33, 0.67)

    # Clamp input to [0, 1]
    t = max(0.0, min(1.0, t))

    rgb = []
    for k in range(3):
        # Calculate argument: 2 * pi * (c[k] * t + d[k])
        arg = 2.0 * math.pi * (c[k] * t + d[k])
        # Evaluate cosine
        cos_val = math.cos(arg)
        # Scale and offset
        val = a[k] + b[k] * cos_val
        # Clamp to [0, 1]
        val = max(0.0, min(1.0, val))
        rgb.append(val)

    # Alpha is not defined by the core cosine formula in the source
    # [G] Default to opaque
    return (rgb[0], rgb[1], rgb[2], 1.0)
```
## Subtopics

### [Blue-Red Colormap](3921-blue-red-colormap.md)
A piecewise linear depth visualization map [G]. Unlike the smooth trigonometric gradients of Cosine Palette, it uses linear segments for high-contrast depth cues [G].

### [Hue Transfer Function](4574-hue-transfer-function.md)
Uses smoothstep functions to create distinct spectral bands [G]. Unlike Cosine Palette's continuous wave interference, it constructs sharp transitions between primary hues [G].

## Operating characteristics
Not given in the sources.
## Background
The method was popularized by Inigo Quilez in the context of GLSL shader programming for procedural color generation [S1, unofficial].
## Sources
[S1] Show HN: Volume rendering 3D data in Three.js and GLSL, https://github.com/SuboptimalEng/volume-rendering, fetched 2026-10-26 (unofficial)
