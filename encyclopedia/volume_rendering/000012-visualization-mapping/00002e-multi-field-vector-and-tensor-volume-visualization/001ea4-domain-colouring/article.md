# Domain Colouring

## Overview
Domain Colouring maps complex numbers, represented by their real and imaginary components, to colors using a cyclic colormap scaled by a degree parameter and brightness [S1, unofficial]. It solves the problem of visualizing the phase and magnitude of complex-valued fields, such as spin orientation in ray-marched scenes, allowing objects to be rendered differently based on their local orientation relative to the camera [S1, unofficial].

## How it works
**Inputs:**
- `real_part`: Tensor of real components of complex numbers [S1, unofficial].
- `imag_part`: Tensor of imaginary components of complex numbers [S1, unofficial].
- `cyclic_colourmap`: A 2D tensor of shape $(N, 3)$ representing the cyclic colormap, where each row is an RGB triplet [S1, unofficial].
- `degree`: An integer scaling factor for the angular resolution [S1, unofficial].

**Outputs:**
- `output`: A tensor of shape matching the input dimensions with 3 channels (RGB), representing the colored result [S1, unofficial].

**Data Structures:**
- The `cyclic_colourmap` is a lookup table where each row is an RGB triplet [S1, unofficial].

**Steps:**
1. Compute the argument (angle) of the complex number $z = \text{real\_part} + i \cdot \text{imag\_part}$ using the two-argument arctangent function: $\theta = \arg(z) = \operatorname{atan2}(\text{imag\_part}, \text{real\_part})$ [S1, unofficial].
2. Normalize the angle to the range $[0, 1)$ by adding $0.5$ and dividing by $2\pi$: $u = \frac{\theta}{2\pi} + 0.5$ [S1, unofficial].
3. Scale the normalized angle by the `degree` parameter and the colormap length $N$ to determine the index in the cyclic colormap: $idx_{float} = u \cdot \text{degree} \cdot N$ [S1, unofficial].
4. Compute the integer colormap index by flooring the scaled value and taking the modulo $N$ to handle wrapping: $idx = \lfloor idx_{float} \rfloor \bmod N$ [S1, unofficial].
5. Retrieve the color $c$ from the colormap at index $idx$: $c = \text{cyclic\_colourmap}[idx]$ [S1, unofficial].
6. Compute the brightness (magnitude) of the complex number: $b = |z| = \sqrt{\text{real\_part}^2 + \text{imag\_part}^2}$ [S1, unofficial].
7. Scale the color by the brightness to produce the final output: $\text{output} = b \cdot c$ [S1, unofficial].

**Formulas:**
$$ z = \text{real\_part} + i \cdot \text{imag\_part} $$
$$ c = \text{cyclic\_colourmap}\left[ \left\lfloor \left( \frac{\arg(z)}{2\pi} + 0.5 \right) \cdot \text{degree} \cdot N \right\rfloor \bmod N \right] $$
$$ b = |z| $$
$$ \text{output} = b \cdot c $$

**Parameters:**
- `degree`: Controls the number of color cycles across the $2\pi$ range. Higher values increase angular resolution [S1, unofficial]. Typical values are not specified in the sources [S1, unofficial].
- $N$: Length of the `cyclic_colourmap` [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Precision:** The modulo operation ensures indices stay within $[0, N-1]$ [S1, unofficial].
- **Degenerate Input:** If both `real_part` and `imag_part` are zero, the argument is undefined. The source code uses `atan2(0, 0)` which typically returns 0 [S1, unofficial]. The magnitude $b$ will be 0, resulting in a black output, which is a valid visual representation for zero magnitude [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import numpy as np

def domain_colouring(real_part, imag_part, cyclic_colourmap, degree):
    # Step 1: Compute argument
    theta = np.arctan2(imag_part, real_part)
    # Step 2: Normalize angle to [0, 1)
    u = theta / (2 * math.pi) + 0.5
    # Step 3: Scale by degree and colormap length
    N = cyclic_colourmap.shape[0]
    idx_float = u * degree * N
    # Step 4: Compute integer index with wrapping
    idx = np.floor(idx_float).astype(int) % N
    # Step 5: Retrieve color
    c = cyclic_colourmap[idx]
    # Step 6: Compute brightness
    brightness = np.sqrt(real_part**2 + imag_part**2)
    # Step 7: Scale color by brightness
    output = brightness[..., np.newaxis] * c
    return output
```

## Subtopics

### Tangent Shading
Tangent Shading is a variant that applies Domain Colouring to surface normals projected onto the tangent plane of a ray [S1, unofficial]. It projects surface normals $\mathbf{n}$ onto the plane perpendicular to the ray direction $\mathbf{r}$, rotates them to camera space, and then applies Domain Colouring to the resulting 2D tangent vectors [S1, unofficial]. This allows visualization of spin orientation relative to the camera [S1, unofficial]. The projection is computed as $\mathbf{t} = \mathbf{n} - (\mathbf{n} \cdot \mathbf{r})\mathbf{r}$, followed by rotation using the camera orientation conjugate [S1, unofficial]. The resulting 2D components $t_x, t_y$ are treated as the real and imaginary parts for the Domain Colouring function [S1, unofficial]. It is chosen when visualizing orientation-dependent effects on surfaces in ray-marched scenes [S1, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
- Time complexity is linear with respect to the number of pixels, $O(P)$, where $P$ is the number of samples, as it involves basic arithmetic operations and a single lookup [G].
- Memory complexity is $O(P)$ for storing the output tensor and intermediate fields [G].

**Measured Performance:**
- Performance data is not given in the sources.

**Scaling:**
- Scales linearly with resolution and degree parameter [G].

**Strengths and Limitations:**
- Strengths: Provides a dense, intuitive visualization of complex phase and magnitude. Simple to implement [G].
- Limitations: The brightness scaling can obscure phase information for large magnitudes. The choice of `degree` affects interpretability [G].

**When to use it and when not:**
- Use when visualizing complex-valued fields or spin orientations in ray-marched scenes [S1, unofficial].
- Do not use when discrete vector directions are required or when trajectory information is needed [G].

## Background
Domain Colouring is used in the context of ray marching and signed distance function (SDF) constructive geometry systems to render objects based on their spin orientation relative to the camera [S1, unofficial]. It is implemented in the `kyle-rosa/ray_marching` repository, which builds on work by Inigo Quilez for SDF primitives and operations [S1, unofficial].

## Sources
[S1] kyle-rosa/ray_marching, https://github.com/kyle-rosa/ray_marching, fetched 2026-10-26 (unofficial)
