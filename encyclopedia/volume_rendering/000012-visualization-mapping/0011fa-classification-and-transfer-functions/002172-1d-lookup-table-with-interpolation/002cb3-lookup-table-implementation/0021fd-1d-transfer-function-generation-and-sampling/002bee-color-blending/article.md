# Color Blending

Also called: Linear Color Interpolation, Linear RGB Blending

## Overview
Color Blending is the final compositing step in volume rendering that combines the color and opacity contributions of multiple samples along a ray into a single pixel color. It solves the problem of accumulating translucent volume data into a coherent 2D image by applying alpha compositing rules. It is distinguished from [Voxel Color Mapping](2970-voxel-color-mapping.md) (which maps scalar values to colors) and [1D Transfer Function Generation and Sampling](2689-1d-transfer-function-generation-and-sampling.md) (which generates the color map) by operating on the accumulated RGBA values rather than individual voxel properties.

## How it works
**Inputs and Outputs:**
- Input: A sequence of $N$ samples along a ray, where each sample $i$ provides a color $C_i = (r_i, g_i, b_i)$ and an opacity $\alpha_i$. The ray starts from the back (farthest) and moves to the front (closest to the camera).
- Output: A final pixel color $C_{final}$ and opacity $\alpha_{final}$.

**Data Structures:**
- Accumulators: Two variables, $C_{acc}$ (color) and $\alpha_{acc}$ (opacity), initialized to $(0, 0, 0, 0)$.

**Steps:**
1. Initialize $C_{acc} = (0, 0, 0, 0)$ and $\alpha_{acc} = 0$.
2. Iterate through samples $i = 0$ to $N-1$ (back to front):
   a. Let $C_i$ be the color and $\alpha_i$ be the opacity of the current sample.
   b. Calculate the transmittance of the current sample: $\tau_i = 1 - \alpha_i$.
   c. Update the accumulated color:
      $$C_{acc} = C_{acc} + C_i \cdot \alpha_i \cdot \tau_{acc}$$
      where $\tau_{acc} = 1 - \alpha_{acc}$ is the transmittance accumulated so far.
   d. Update the accumulated opacity:
      $$\alpha_{acc} = \alpha_{acc} + \alpha_i \cdot \tau_{acc}$$
   e. If $\alpha_{acc} \ge 1.0$, break the loop (ray is fully opaque).
3. The final color is $C_{final} = C_{acc}$ and final opacity is $\alpha_{final} = \alpha_{acc}$.

**Formulas:**
- Alpha Compositing (Over operator):
  $$C_{out} = C_{in} + C_{src} \cdot \alpha_{src} \cdot (1 - \alpha_{in})$$
  $$\alpha_{out} = \alpha_{in} + \alpha_{src} \cdot (1 - \alpha_{in})$$

**Parameters:**
- $\alpha_i$: Typically in range $[0, 1]$.
- $C_i$: Typically in range $[0, 1]$ per channel.

**Edge Cases and Pitfalls:**
- **Premultiplied Alpha:** If colors are stored as premultiplied alpha ($C' = C \cdot \alpha$), the formula simplifies to $C_{acc} = C_{acc} + C'_i \cdot \tau_{acc}$.
- **Ordering:** Samples must be processed from back to front for the standard "over" operator.
- **Precision:** Accumulating many small $\alpha_i$ values can lead to precision loss if not handled carefully. Using double precision for accumulators is recommended for high dynamic range scenes.
- **Early Exit:** If $\alpha_{acc}$ reaches 1.0, remaining samples do not contribute and can be skipped.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def color_blend(samples):
    """
    samples: list of tuples (r, g, b, alpha) from back to front.
    Returns: (r, g, b, alpha)
    """
    acc_r, acc_g, acc_b, acc_alpha = 0.0, 0.0, 0.0, 0.0
    
    for r, g, b, alpha in samples:
        if alpha <= 0.0:
            continue
            
        # Transmittance of the accumulated ray so far
        tau_acc = 1.0 - acc_alpha
        
        # Contribution of current sample
        contribution_r = r * alpha * tau_acc
        contribution_g = g * alpha * tau_acc
        contribution_b = b * alpha * tau_acc
        
        # Update accumulators
        acc_r += contribution_r
        acc_g += contribution_g
        acc_b += contribution_b
        acc_alpha += alpha * tau_acc
        
        # Early exit if fully opaque
        if acc_alpha >= 1.0:
            break
            
    # Clamp final alpha to 1.0
    acc_alpha = min(acc_alpha, 1.0)
    
    return (acc_r, acc_g, acc_b, acc_alpha)
```

## Subtopics

### [SVT Animation Interpolation](7953-svt-animation-interpolation.md)
SVT Animation Interpolation is a technique used to smooth the playback of animated Sparse Volume Textures (SVTs) by blending between two adjacent frames using a scalar alpha value. It addresses the visual discontinuity inherent in discrete frame-based animation by providing temporal continuity, distinguishing it from simple frame stepping or non-interpolated playback.

### Blur Group Colour Replacement
A mechanism to preserve sharpness when the average method causes object blur due to large differences between C1 and C2, by replacing the voxel's colour with that of a neighbouring voxel from a 'clear group' [S2].
- **Definition:** Used when the difference between color components $C1$ and $C2$ is large, causing blur [S2].
- **Steps:**
  1. Set defined colours into a blur group if the difference of $C1$ and $C2$ is more than a particular threshold [S2].
  2. Set defined colours into a clear group if the difference of $C1$ and $C2$ is less than or equal to the threshold [S2].
  3. Replace colour $C_{old}$ of the voxel in the blur group by colour of neighbouring voxel that has the closest value to $C_{old}$ and has to be in the clear group [S2].

## Operating characteristics
- **Time Complexity:** $O(N)$ per ray, where $N$ is the number of samples. Early exit can reduce average case complexity.
- **Memory Complexity:** $O(1)$ per ray, as only accumulators are needed.
- **Strengths:** Physically plausible compositing for translucent volumes; simple to implement; efficient with early exit.
- **Limitations:** Does not handle transparency order correctly for non-Lambertian surfaces without additional depth sorting; sensitive to sample density.
- **When to use:** Standard volume rendering where alpha compositing is appropriate.
- **When not to use:** When depth sorting is required for correct transparency rendering of opaque objects within the volume.

## Background
Color blending in volume rendering is based on the classic alpha compositing equation introduced by Porter and Duff [S1, unofficial]. It is a fundamental operation in ray casting and texture mapping based volume rendering techniques.

## Sources
[S1] Musta-Kissa/quadtree-raycast, https://github.com/Musta-Kissa/quadtree-raycast, fetched 2026-10-24 (unofficial)
[S2] Volumetric Representation for Sparse Multi-Views, https://doi.org/10.1109/icip.2006.312545, fetched 2026-10-26
