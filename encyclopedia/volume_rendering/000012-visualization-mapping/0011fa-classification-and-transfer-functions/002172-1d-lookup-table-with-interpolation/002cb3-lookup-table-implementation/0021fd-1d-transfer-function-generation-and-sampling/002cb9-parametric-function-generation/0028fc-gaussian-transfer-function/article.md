# Gaussian Transfer Function

Also called: 3-Gaussian Transfer Function, TransferFunctionWidget3

## Overview
The Gaussian Transfer Function constructs a 4-channel RGBA lookup table using three Gaussian bumps to map scalar values to color and opacity. It differs from its siblings by explicitly normalizing RGB channels by their maximum sum to prevent overflow and by using fixed primary colors (red, green, blue) for the three respective bumps, whereas [Cosine Palette Color Mapping](3359-cosine-palette-color-mapping.md) uses a continuous palette and [Sigmoid Function with Jacobian](6497-sigmoid-function-with-jacobian.md) uses monotonic transitions.

## How it works
**Inputs and Outputs:**
- Input: Three sets of parameters $(level_j, opacity_j, width_j)$ for $j \in \{1, 2, 3\}$.
- Output: A texture `rgba` of shape $(1024, 4)$, where index $i$ corresponds to normalized scalar $t = i/1023$.

**Data Structures:**
- `rgba`: Array of shape $(1024, 4)$. Channels 0-2 are RGB, channel 3 is opacity.
- `colors`: Fixed list of RGB vectors: $[ (1,0,0), (0,1,0), (0,0,1) ]$.

**Steps:**
1. Initialize `rgba` with zeros.
2. For each texel index $i$ from 0 to 1023:
   a. Calculate position $t = i / 1023.0$.
   b. Initialize `intensity_sum` = 0.0.
   c. For each Gaussian $j \in \{1, 2, 3\}$:
      i. Calculate intensity: $I_j = \exp\left(-\left(\frac{t - level_j}{width_j}\right)^2\right)$.
      ii. Add color contribution: `rgba[i, 0:3]` += `colors[j-1]` $\times opacity_j \times I_j$.
      iii. Add opacity contribution: `rgba[i, 3]` += $opacity_j \times I_j$.
   d. Normalize RGB: If `rgba[i, 0:3].max() > 0`, divide `rgba[i, 0:3]` by `rgba[i, 0:3].max()`. [G] If max is 0, leave RGB as 0 to avoid NaN.
   e. Clamp: `rgba[i]` = clip(`rgba[i]`, 0, 1).

**Formulas:**
$$ I_j(t) = \exp\left(-\left(\frac{t - level_j}{width_j}\right)^2\right) $$
$$ \mathbf{C}_{raw}[i] = \sum_{j=1}^{3} \mathbf{c}_j \cdot opacity_j \cdot I_j(t) $$
$$ \alpha[i] = \sum_{j=1}^{3} opacity_j \cdot I_j(t) $$
$$ \mathbf{C}_{final}[i] = \text{clip}\left( \frac{\mathbf{C}_{raw}[i]}{\max(\mathbf{C}_{raw}[i])}, 0, 1 \right) $$

**Parameters:**
- `levels`: $[0.1, 0.5, 0.8]$ (Default)
- `opacities`: $[0.4, 0.1, 0.1]$ (Default)
- `widths`: $[0.1, 0.1, 0.1]$ (Default)

**Pitfalls:**
- Division by zero if all Gaussians contribute 0 intensity at a specific $t$. The source code clips to [0,1] after division, which may leave NaNs if not guarded. [G] A guard `if max_val > 0` is required.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def generate_gaussian_tf(levels, opacities, widths):
    N = 1024
    rgba = np.zeros((N, 4))
    colors = np.array([[1, 0, 0], [0, 1, 0], [0, 0, 1]])
    
    for i in range(N):
        t = i / 1023.0
        rgb_sum = np.zeros(3)
        alpha_sum = 0.0
        
        for j in range(3):
            level = levels[j]
            opacity = opacities[j]
            width = widths[j]
            
            # Gaussian intensity
            intensity = np.exp(-((t - level) / width) ** 2)
            
            # Accumulate color and opacity
            rgb_sum += colors[j] * opacity * intensity
            alpha_sum += opacity * intensity
            
        # Normalize RGB
        max_rgb = np.max(rgb_sum)
        if max_rgb > 0:
            rgb_sum /= max_rgb
            
        rgba[i, 0:3] = rgb_sum
        rgba[i, 3] = alpha_sum
        
    return np.clip(rgba, 0, 1)
```

## Subtopics

### [Explorable Image Representation](6507-explorable-image-representation.md)
Pre-renders scalar data into multi-layered 2D images for interactive transparency/color modification without re-rendering 3D. Use when computational resources are limited.

## Operating characteristics
- **Time Complexity:** $O(N)$ where $N=1024$ is the texture size. Constant factor is small (3 Gaussians per texel).
- **Memory:** $1024 \times 4 \times 4$ bytes $\approx 16$ KB for the texture.
- **Performance:** Fast generation; suitable for real-time UI updates.
- **Strengths:** Simple parameterization; intuitive control over distinct scalar ranges.
- **Limitations:** Fixed color assignment (Red/Green/Blue) limits aesthetic flexibility; normalization can distort color perception if one Gaussian dominates.

## Background
Originated in the `ipyvolume` library by Maarten Breddels [S1, unofficial]. It is implemented as `TransferFunctionWidget3` in the `transferfunction` module.

## Sources
[S1] https://github.com/maartenbreddels/ipyvolume/blob/master/ipyvolume/transferfunction.py, https://github.com/maartenbreddels/ipyvolume/blob/master/ipyvolume/transferfunction.py, fetched 2026-10-29 (unofficial)
