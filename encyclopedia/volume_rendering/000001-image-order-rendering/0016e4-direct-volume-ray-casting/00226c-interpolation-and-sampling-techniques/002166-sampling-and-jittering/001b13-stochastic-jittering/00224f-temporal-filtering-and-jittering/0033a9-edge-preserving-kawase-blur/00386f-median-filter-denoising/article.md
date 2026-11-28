# Median Filter Denoising

Also called: Postprocessing Shader Median Filter

## Overview
A post-processing technique used to reduce 'Salt and Pepper' noise caused by unpredictable precision errors in raycasting that result in wrong or missing pixel colors [S1, unofficial]. It operates on a fixed 3x3 neighborhood to remove extreme single-pixel outliers [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   Input: The rendered image buffer (texture) from the raycasting step [S1, unofficial].
*   Output: A denoised image buffer [S1, unofficial].

**Data Structures:**
*   `colorStack`: An array of 9 `float4` color values [S1, unofficial].
*   `lookups`: An array of 9 `float2` offset vectors representing the relative coordinates of the 3x3 neighborhood [S1, unofficial].
*   `GRAY_SCALE_FACTORS`: A constant vector used in a dot product to convert `float4` colors to a scalar grayscale value for comparison [S1, unofficial].

**Steps:**
1.  Calculate the texture coordinate step sizes: $dx = 1.0 / \text{width}$ and $dy = 1.0 / \text{height}$ [S1, unofficial].
2.  Define the 3x3 neighborhood offsets relative to the current pixel $(0,0)$:
    $$
    \begin{aligned}
    &(-dx, -dy), (0, -dy), (dx, -dy), \\
    &(-dx, 0), (0, 0), (dx, 0), \\
    &(-dx, dy), (0, dy), (dx, dy)
    \end{aligned}
    $$
    [S1, unofficial].
3.  For each pixel $p_{i,j}$:
    a. Retrieve the 8 surrounding pixels and the center pixel using the offsets defined in step 2 [S1, unofficial].
    b. Store these 9 colors in `colorStack` [S1, unofficial].
    c. Sort `colorStack` based on the grayscale luminance of the colors using the comparison function $C(a, b) = \text{dot}(\text{GRAY\_SCALE\_FACTORS}, a) > \text{dot}(\text{GRAY\_SCALE\_FACTORS}, b)$ [S1, unofficial].
    d. The median of 9 elements ($n=9, m=4$) is the element at index 4 (the 5th element) because arrays are zero-based [S1, unofficial].
    e. Set the output pixel color to `colorStack[4]` [S1, unofficial].

**Formula:**
For an odd number of elements $n = 2m + 1$, the median is the element at index $m$ [S1, unofficial]. For even $n = 2m$, it is the average of elements at $m$ and $m+1$ [S1, unofficial]. In this 3x3 case ($n=9, m=4$), the median is $x_5$ (stored at index 4) [S1, unofficial].

**Parameters:**
*   `GRAY_SCALE_FACTORS`: Luminance weights for color comparison. Not given in the sources [S1, unofficial].

**Pitfalls:**
*   **Indexing Error:** Arrays are zero-based, so the 5th element (median of 9) is at index 4, not index 5 [S1, unofficial].
*   **Boundary Conditions:** Not given in the sources [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Constants/Parameters (provided by the rendering context)
GRAY_SCALE_FACTORS = vec4(0.0, 0.0, 0.0, 0.0)  # Not given in the sources [S1, unofficial]
width = 1920  # Image width in pixels
height = 1080 # Image height in pixels

def luminance(color: vec4) -> float:
    return dot(color, GRAY_SCALE_FACTORS)

def compare_colors(a: vec4, b: vec4) -> bool:
    return luminance(a) > luminance(b)

def texture_fetch(uv: vec2) -> vec4:
    return vec4(0.0)  # [G: Standard texture lookup operation]

def median_filter_pixel(center_uv: vec2, dx: float, dy: float) -> vec4:
    offsets = [
        vec2(-dx, -dy), vec2(0, -dy), vec2(dx, -dy),
        vec2(-dx, 0),   vec2(0, 0),   vec2(dx, 0),
        vec2(-dx, dy),  vec2(0, dy),  vec2(dx, dy)
    ]
    
    color_stack = [vec4(0.0) for _ in range(9)]
    current_count = 0
    
    for k in range(9):
        current_color = texture_fetch(center_uv + offsets[k])
        
        idx = current_count
        while idx > 0 and compare_colors(color_stack[idx - 1], current_color):
            color_stack[idx] = color_stack[idx - 1]
            idx -= 1
        color_stack[idx] = current_color
        current_count += 1
        
    return color_stack[4]

def postprocessing_shader(input_uv: vec2) -> vec4:
    dx = 1.0 / width
    dy = 1.0 / height
    return median_filter_pixel(input_uv, dx, dy)
```

## Subtopics

### [Forgetful Selection Median Filter](12105-forgetful-selection-median-filter.md)
The Forgetful Selection Median Filter is a GPU-optimized algorithm for computing the median of a data set, specifically designed to overcome register pressure limitations on CUDA hardware. It solves the problem of high memory usage in classic histogram-based median filters, which require excessive registers per thread, thereby limiting parallelism on GPUs with limited low-latency memory (up to compute capability 3.5). It sets itself apart from histogram-based siblings by maintaining a small, fixed-size subset of values in registers rather than a full frequency distribution, allowing sorting operations to occur entirely within registers.

## Operating characteristics
**Time and Memory Complexity:**
*   Time Complexity: $O(1)$ because the loop always iterates through a fixed array of length 9, regardless of image size [S1, unofficial].
*   Memory Complexity: $O(1)$ per pixel, requiring storage for 9 colors and a few scalar variables [S1, unofficial].

**Measured Performance:**
*   Not given in the sources [S1, unofficial].

**Strengths and Limitations:**
*   **Strengths:** Effective at removing "Salt and Pepper" noise caused by precision errors in raycasting [S1, unofficial]. Simple to implement as a post-processing shader [S1, unofficial].
*   **Limitations:** Only considers a 3x3 neighborhood, which may not be sufficient for larger noise patterns [S1, unofficial].

**When to use it:**
*   Use when the primary noise source is single-pixel outliers (salt and pepper noise) resulting from raycasting precision issues [S1, unofficial].

**When not to use it:**
*   Not given in the sources [S1, unofficial].

## Background
Origin: Developed as part of a raycasting-based voxel rendering system to handle noise from precision errors [S1, unofficial]. The technique is described in a thesis/report by Enes Hergül (2022) [S1, unofficial].

## Sources
[S1] Raycasting-basiertes Voxelrendering, https://serwiss.bib.hs-hannover.de/frontdoor/index/index/docId/2250, fetched 2026-10-25 (unofficial)
