# Gaussian Transfer Function

Also called: 3-Gaussian Transfer Function, TransferFunctionWidget3

## Overview
The Gaussian Transfer Function is a parametric method for generating a 4-channel RGBA lookup table (LUT) using three additive Gaussian basis functions to control color and opacity based on normalized scalar input values [S1, unofficial]. It differs from its siblings, such as [Cosine Palette Color Mapping](3359-cosine-palette-color-mapping.md) and [Sigmoid Function with Jacobian](6497-sigmoid-function-with-jacobian.md), by using localized Gaussian bumps rather than global periodic or sigmoidal curves, allowing for distinct, isolated highlighting of specific scalar ranges [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input**: Three sets of parameters, each containing a `level` (center position), `opacity` (maximum contribution), and `width` (standard deviation). The input scalar $t$ is normalized to $[0, 1]$ by the parent method [1D Transfer Function Generation and Sampling](2689-1d-transfer-function-generation-and-sampling.md).
*   **Output**: A numpy array `rgba` of shape $(1024, 4)$, where index $i$ corresponds to scalar value $t = i / 1023.0$. Channels 0-2 are RGB color, channel 3 is opacity [S1, unofficial].

**Data Structures**
*   `levels`: List of 3 floats $[l_1, l_2, l_3]$, default $[0.1, 0.5, 0.8]$.
*   `opacities`: List of 3 floats $[o_1, o_2, o_3]$, default $[0.4, 0.1, 0.1]$.
*   `widths`: List of 3 floats $[w_1, w_2, w_3]$, default $[0.1, 0.1, 0.1]$.
*   `colors`: Fixed mapping of Gaussian index $j \in \{0, 1, 2\}$ to RGB vectors:
    *   $j=0$: Red $[1, 0, 0]$
    *   $j=1$: Green $[0, 1, 0]$
    *   $j=2$: Blue $[0, 0, 1]$
    *   *Note*: The source code explicitly uses "red", "green", "blue" from matplotlib [S1, unofficial].

**Formulas**
For each texel $i$ in the output LUT:
1.  Calculate position: $t_i = \frac{i}{1023.0}$
2.  For each Gaussian $j \in \{0, 1, 2\}$:
    $$ I_j = \exp\left( -\left( \frac{t_i - l_j}{w_j} \right)^2 \right) $$
3.  Accumulate contributions:
    $$ C_{rgb} = \sum_{j=0}^{2} \mathbf{c}_j \cdot o_j \cdot I_j $$
    $$ A = \sum_{j=0}^{2} o_j \cdot I_j $$
    Where $\mathbf{c}_j$ is the color vector for Gaussian $j$.
4.  Normalize RGB:
    $$ M = \max(C_{rgb}[0], C_{rgb}[1], C_{rgb}[2]) $$
    If $M > 0$:
    $$ C_{rgb} = \frac{C_{rgb}}{M} $$
    Else:
    $$ C_{rgb} = [0, 0, 0] $$
5.  Clamp:
    $$ \text{rgba}[i] = \text{clip}(C_{rgb}, 0, 1) \cup \text{clip}(A, 0, 1) $$

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def generate_gaussian_transfer_function(levels, opacities, widths):
    """
    Generate a 1024x4 RGBA transfer function using 3 Gaussians.
    
    Args:
        levels: List of 3 floats [l1, l2, l3] in [0, 1]
        opacities: List of 3 floats [o1, o2, o3] in [0, 1]
        widths: List of 3 floats [w1, w2, w3] in (0, 1]
        
    Returns:
        numpy array of shape (1024, 4)
    """
    N = 1024
    rgba = np.zeros((N, 4))
    
    # Fixed color mapping: Red, Green, Blue
    colors = np.array([
        [1.0, 0.0, 0.0],
        [0.0, 1.0, 0.0],
        [0.0, 0.0, 1.0]
    ])
    
    for i in range(N):
        t = i / 1023.0
        intensity_sum_rgb = np.zeros(3)
        intensity_sum_alpha = 0.0
        
        for j in range(3):
            l = levels[j]
            o = opacities[j]
            w = widths[j]
            
            # Avoid division by zero if width is 0
            if w == 0:
                intensity = 0.0
            else:
                intensity = np.exp(-((t - l) / w) ** 2)
            
            intensity_sum_rgb += colors[j] * o * intensity
            intensity_sum_alpha += o * intensity
            
        # Normalize RGB by max component to prevent overflow and ensure perceptual consistency
        max_rgb = np.max(intensity_sum_rgb)
        if max_rgb > 0:
            intensity_sum_rgb /= max_rgb
        else:
            intensity_sum_rgb = np.zeros(3)
            
        # Clamp to [0, 1]
        rgba[i, 0:3] = np.clip(intensity_sum_rgb, 0.0, 1.0)
        rgba[i, 3] = np.clip(intensity_sum_alpha, 0.0, 1.0)
        
    return rgba
```

**Edge Cases and Pitfalls**
*   **Division by Zero**: If `max_rgb` is 0 (no Gaussian contributes to any color channel), the division step results in NaN. The code must explicitly handle this by setting RGB to zero [S1, unofficial].
*   **Width Zero**: If `widths[j]` is 0, the Gaussian formula involves division by zero. The implementation should treat intensity as 0 or handle it as a delta function (not specified in source, but implied by the exponential form) [G].
*   **Opacity Overflow**: The alpha channel accumulates opacities. If multiple Gaussians overlap significantly, alpha can exceed 1.0 before clamping. The source clamps alpha to [0, 1] [S1, unofficial].

## Subtopics

### [Explorable Image Representation](6507-explorable-image-representation.md)
A method for interactive volume visualization that pre-renders scalar data into multi-layered 2D images, allowing users to modify layer transparency and color without re-rendering the 3D scene. It differs from standard ray-casting by decoupling interaction from the 3D data, enabling image-based exploration on devices with limited computational resources.

## Operating characteristics
*   **Time Complexity**: $O(N \cdot K)$ where $N$ is the LUT size (1024) and $K$ is the number of Gaussians (3). This is constant time relative to scene complexity, making it very fast for LUT generation.
*   **Memory Complexity**: $O(N)$ for the LUT storage. For $N=1024$, this is 4KB (float32).
*   **Performance**: The method is computationally inexpensive for LUT generation. The source code uses a Python loop, which is slow for real-time updates but acceptable for static LUT generation. Vectorization (as noted in the source's `# TODO: vectorize`) would improve performance [S1, unofficial].
*   **Strengths**: Simple parameterization; intuitive control over distinct scalar ranges; fixed color mapping ensures consistent semantic meaning (Red/Green/Blue).
*   **Limitations**: Fixed color mapping limits artistic flexibility; additive opacity can lead to non-physical alpha values if not clamped; normalization of RGB may reduce intensity if multiple Gaussians overlap [S1, unofficial].
*   **When to Use**: When distinct, isolated scalar ranges need to be highlighted with specific colors (e.g., bone, muscle, fat in medical imaging).
*   **When Not to Use**: When smooth, continuous color transitions are required (use [Cosine Palette Color Mapping](3359-cosine-palette-color-mapping.md)) or when complex, non-localized color mappings are needed.

## Background
The Gaussian Transfer Function is implemented in the `ipyvolume` library, specifically in the `TransferFunctionWidget3` class [S1, unofficial]. It was developed by Maarten Breddels as part of the `ipyvolume` project, which provides interactive 3D visualization in Jupyter notebooks [S1, unofficial]. The method is designed to provide a simple, widget-driven interface for users to adjust Gaussian parameters (level, opacity, width) for three distinct scalar ranges [S1, unofficial]. It differs from `TransferFunctionJsBumps` by computing the RGBA texture in Python using matplotlib colors, whereas the JS version uses a JavaScript-based model [S1, unofficial].

## Sources
[S1] https://github.com/maartenbreddels/ipyvolume/blob/master/ipyvolume/transferfunction.py, https://github.com/maartenbreddels/ipyvolume/blob/master/ipyvolume/transferfunction.py, fetched 2026-10-29 (unofficial)
