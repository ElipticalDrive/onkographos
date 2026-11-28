# Color Quantization


## Overview
Color Quantization reduces the number of unique colors in a voxel grid to facilitate greedy meshing by snapping RGB values to discrete levels [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A voxel color tuple $(r, g, b)$ where each channel is an integer in $[0, 255]$, and an integer parameter $L$ representing the number of quantization levels per channel [S1, unofficial].
- Output: A quantized color tuple $(r', g', b')$ with the same range and type [S1, unofficial].

**Data Structures:**
- The input is a 3-tuple of integers. No complex data structures are required; the operation is element-wise [G].

**Steps:**
1. Calculate the step size for quantization:
   $$ \text{step} = 256 \div L $$
   where $L$ is the number of levels [S1, unofficial].
2. For each color channel $c \in \{r, g, b\}$:
   a. Divide the channel value by the step size using integer division:
      $$ q = \lfloor c \div \text{step} \rfloor $$
   b. Multiply by the step size to snap to the lower bound of the bin:
      $$ \text{snap} = q \times \text{step} $$
   c. Add half the step size to center the value within the bin:
      $$ \text{centered} = \text{snap} + \lfloor \text{step} \div 2 \rfloor $$
   d. Clip the result to the valid range $[0, 255]$:
      $$ c' = \max(0, \min(255, \text{centered})) $$
3. Return the tuple $(r', g', b')$ [S1, unofficial].

**Parameters:**
- $L$ (levels): Number of quantization levels per channel. Typical values are 8, 16, or 32 [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def color_quantize(color: tuple[int, int, int], levels: int) -> tuple[int, int, int]:
    step = 256 // levels
    quantized = []
    for c in color:
        val = (c // step) * step + (step // 2)
        val = max(0, min(255, val))
        quantized.append(val)
    return tuple(quantized)
```

**Edge Cases and Pitfalls:**
- If $L=0$, division by zero occurs. The sources do not specify a guard for this case [G].
- Integer division truncates towards zero, which is standard for positive RGB values [G].

## Subtopics

### [Color Quantization via Histogram & Manhattan Distance Mapping](12827-color-quantization-via-histogram-manhattan-distance-mapping.md)
Selects a palette based on frequency and maps colors using Manhattan distance. Chose when palette size is strictly limited and frequency matters more than perceptual uniformity.

## Operating characteristics
- **Time Complexity:** $O(1)$ per voxel. The operation involves a constant number of arithmetic operations per channel [G].
- **Memory Complexity:** $O(1)$ auxiliary space. No additional data structures are allocated per voxel [G].
- **Strengths:** Extremely fast, deterministic, and simple to implement [G].
- **Limitations:** Introduces color banding artifacts if $L$ is too small [G].
- **When to use:** Real-time voxelization pipelines where performance is critical and slight color inaccuracies are acceptable [S1, unofficial].

## Background
Originated in the context of optimizing voxel-based rendering pipelines, specifically for converting 3D Gaussian Splatting (3DGS) data into OpenUSD formats for game engines like Unreal Engine 5 [S1, unofficial].

## Sources
[S1] omolism/3DGS-to-Voxel-OpenUSD, https://github.com/omolism/3DGS-to-Voxel-OpenUSD, fetched 2026-10-27 (unofficial)
