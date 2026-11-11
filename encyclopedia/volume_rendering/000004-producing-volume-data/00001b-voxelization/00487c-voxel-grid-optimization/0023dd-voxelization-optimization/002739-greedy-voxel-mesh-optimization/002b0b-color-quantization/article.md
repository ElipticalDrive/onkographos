# Color Quantization

Also called: RGB level snapping, discrete color mapping

## Overview
Color Quantization reduces the number of unique colors in a voxel grid to facilitate greedy meshing by snapping RGB values to discrete levels [S1, unofficial]. It differs from its sibling [Disjoint Set Visible Plane Optimization](4045-disjoint-set-visible-plane-optimization.md) by operating on color attributes rather than geometric topology, ensuring that adjacent voxels with visually similar colors are treated as identical for the purpose of quad merging [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A voxel color tuple $(r, g, b)$ where each channel is an integer in $[0, 255]$, and an integer parameter $L$ representing the number of quantization levels per channel.
- Output: A quantized color tuple $(r', g', b')$ with the same range and type.

**Data Structures:**
- The input is a 3-tuple of integers. No complex data structures are required; the operation is element-wise.

**Steps:**
1. Calculate the step size for quantization:
   $$ \text{step} = \lfloor 256 / L \rfloor $$
   where $L$ is the number of levels.
2. For each color channel $c \in \{r, g, b\}$:
   a. Divide the channel value by the step size using integer division:
      $$ q = \lfloor c / \text{step} \rfloor $$
   b. Multiply by the step size to snap to the lower bound of the bin:
      $$ \text{snap} = q \times \text{step} $$
   c. Add half the step size to center the value within the bin:
      $$ \text{centered} = \text{snap} + \lfloor \text{step} / 2 \rfloor $$
   d. Clip the result to the valid range $[0, 255]$:
      $$ c' = \max(0, \min(255, \text{centered})) $$
3. Return the tuple $(r', g', b')$.

**Parameters:**
- $L$ (levels): Number of quantization levels per channel. Typical values are 8, 16, or 32 [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def color_quantize(color: tuple[int, int, int], levels: int) -> tuple[int, int, int]:
    """
    Quantize an RGB color tuple to discrete levels.
    
    Args:
        color: Tuple of (r, g, b) integers in [0, 255].
        levels: Number of quantization levels per channel.
        
    Returns:
        Quantized tuple of (r, g, b) integers.
    """
    step = 256 // levels
    
    quantized = []
    for c in color:
        # Snap to bin and center
        val = (c // step) * step + (step // 2)
        # Clip to valid range
        val = max(0, min(255, val))
        quantized.append(val)
        
    return tuple(quantized)
```

**Edge Cases and Pitfalls:**
- If $L=1$, step is 256. The formula results in 128 for all inputs, collapsing all colors to gray [S1, unofficial].
- Integer division truncates towards zero. For negative inputs (not applicable to standard RGB), behavior would differ, but RGB is unsigned.
- Precision loss is inherent; colors are mapped to the nearest bin center.

## Subtopics

### [Color Quantization via Histogram & Manhattan Distance Mapping](12827-color-quantization-via-histogram-manhattan-distance-mapping.md)
Selects a palette based on frequency and maps colors using Manhattan distance. Chose when palette size is strictly limited and frequency matters more than perceptual uniformity.

## Operating characteristics
- **Time Complexity:** $O(1)$ per voxel. The operation involves a constant number of arithmetic operations per channel.
- **Memory Complexity:** $O(1)$ auxiliary space. No additional data structures are allocated per voxel.
- **Strengths:** Extremely fast, deterministic, and simple to implement. Reduces draw calls by merging adjacent voxels with quantized colors.
- **Limitations:** Introduces color banding artifacts if $L$ is too small. Does not consider perceptual color differences (e.g., green vs. blue sensitivity).
- **When to use:** Real-time voxelization pipelines where performance is critical and slight color inaccuracies are acceptable [S1, unofficial].
- **When not to use:** High-fidelity rendering requiring perceptual color accuracy or when the number of unique colors must be optimized based on scene statistics rather than fixed bins.

## Background
Originated in the context of optimizing voxel-based rendering pipelines, specifically for converting 3D Gaussian Splatting (3DGS) data into OpenUSD formats for game engines like Unreal Engine 5 [S1, unofficial]. It serves as a preprocessing step to reduce the complexity of greedy meshing algorithms by limiting the color space.

## Sources
[S1] omolism/3DGS-to-Voxel-OpenUSD, https://github.com/omolism/3DGS-to-Voxel-OpenUSD, fetched 2026-10-27 (unofficial)
