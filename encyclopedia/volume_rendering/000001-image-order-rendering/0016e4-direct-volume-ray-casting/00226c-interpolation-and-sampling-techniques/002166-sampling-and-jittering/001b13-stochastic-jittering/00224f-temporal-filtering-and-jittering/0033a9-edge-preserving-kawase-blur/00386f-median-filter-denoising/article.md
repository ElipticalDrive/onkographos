# Median Filter Denoising

Also called: Postprocessing Shader Median Filter

## Overview
A post-processing technique used to reduce 'Salt and Pepper' noise caused by unpredictable precision errors in raycasting that result in wrong or missing pixel colors [S1, unofficial]. It sets apart from its sibling [Edge-Preserving Kawase Blur](7026-edge-preserving-kawase-blur.md) by operating on a fixed 3x3 neighborhood to remove isolated outliers (single-pixel errors) rather than blurring dither patterns, making it suitable for correcting discrete precision failures rather than high-frequency jitter [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: The rendered image buffer (float4 colors) from the raycasting step.
- Output: A denoised image buffer where isolated noisy pixels are replaced by the median of their neighbors.

**Data Structures:**
- `colorStack`: An array of 9 `float4` values, representing the current pixel and its 8 immediate neighbors.
- `GRAY_SCALE_FACTORS`: A constant vector (typically `[0.299, 0.587, 0.114]` for standard luminance, though the source does not specify the exact values, only its existence) used to convert `float4` colors to a scalar grayscale value for comparison [S1, unofficial].

**Steps:**
1. For each pixel $p_{i,j}$ in the output image:
2. Calculate texture lookup offsets:
   - $dx = 1.0 / \text{width}$
   - $dy = 1.0 / \text{height}$
3. Define the 9 relative offsets for the 3x3 neighborhood:
   - $(-dx, -dy), (0, -dy), (dx, -dy)$
   - $(-dx, 0), (0, 0), (dx, 0)$
   - $(-dx, dy), (0, dy), (dx, dy)$
4. Initialize an empty `colorStack` of size 9 and `currentIndex = 0`.
5. For each of the 9 offsets:
   - Sample the color $C$ from the input texture at $p_{i,j} + \text{offset}$.
   - Insert $C$ into `colorStack` at `currentIndex`.
   - Perform insertion sort on `colorStack` up to `currentIndex` to maintain sorted order.
   - Increment `currentIndex`.
6. The median is the element at index 4 (the 5th element) of the sorted `colorStack` [S1, unofficial].
   - Note: Arrays are zero-based; the 5th element is at index 4, not 5 [S1, unofficial].
7. Set the output pixel color to this median value.

**Formula:**
For an odd number of elements $n = 2m + 1$, the median is the element at index $m$. For even $n = 2m$, it is the average of elements at $m$ and $m+1$. In this 3x3 case ($n=9, m=4$), the median is the 5th element ($x_5$) [S1, unofficial].

**Comparison Function:**
Colors are compared based on their grayscale value using the dot product with `GRAY_SCALE_FACTORS`:
$$ \text{compare}(a, b) = (\text{GRAY\_SCALE\_FACTORS} \cdot a) > (\text{GRAY\_SCALE\_FACTORS} \cdot b) $$
[S1, unofficial]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def median_filter_denoise(input_texture, width, height):
    # Constants
    GRAY_SCALE_FACTORS = [0.299, 0.587, 0.114] # [G] Standard luminance weights; source does not specify exact values
    
    # Precompute offsets
    dx = 1.0 / width
    dy = 1.0 / height
    offsets = [
        (-dx, -dy), (0, -dy), (dx, -dy),
        (-dx, 0),   (0, 0),   (dx, 0),
        (-dx, dy),  (0, dy),  (dx, dy)
    ]
    
    output_image = empty_image_of_size(width, height)
    
    for y in range(height):
        for x in range(width):
            color_stack = []
            
            # Sample and insert-sort neighbors
            for i in range(9):
                offset_x, offset_y = offsets[i]
                sample_x = x + offset_x
                sample_y = y + offset_y
                
                # Clamp coordinates if necessary [G]
                sample_x = clamp(sample_x, 0, width - 1)
                sample_y = clamp(sample_y, 0, height - 1)
                
                current_color = input_texture.sample(sample_x, sample_y)
                
                # Insert into sorted stack
                # Find position to insert
                insert_pos = len(color_stack)
                for j in range(len(color_stack)):
                    if compare(current_color, color_stack[j]):
                        insert_pos = j
                        break
                
                # Insert
                color_stack.insert(insert_pos, current_color)
            
            # Median is the 5th element (index 4)
            median_color = color_stack[4]
            output_image.set_pixel(x, y, median_color)
            
    return output_image

def compare(color_a, color_b):
    # Compare based on grayscale value
    val_a = dot(GRAY_SCALE_FACTORS, color_a)
    val_b = dot(GRAY_SCALE_FACTORS, color_b)
    return val_a > val_b
```

**Parameters:**
- `GRAY_SCALE_FACTORS`: A constant vector used in the dot product to convert float4 colors to a scalar grayscale value for comparison [S1, unofficial]. Not given in the sources.

**Edge Cases and Pitfalls:**
- **Zero-based indexing:** Arrays are zero-based, so the 5th element (median of 9) is at index 4, not index 5 [S1, unofficial].
- **Boundary conditions:** The source does not specify how to handle pixels at the image boundary (e.g., clamping, wrapping, or ignoring). [G] Clamping is assumed.
- **Precision errors:** The filter specifically targets "Salt and Pepper" noise caused by unpredictable precision errors in raycasting [S1, unofficial].

## Subtopics

### [Forgetful Selection Median Filter](12105-forgetful-selection-median-filter.md)
A GPU-optimized median filter algorithm used to de-noise volumetric images, chosen because classic histogram-based approaches require too many registers per thread on CUDA hardware.

## Operating characteristics
- **Time Complexity:** O(1) because the loop always iterates through a fixed array of length 9, regardless of image size [S1, unofficial].
- **Memory Complexity:** O(1) per pixel, requiring storage for 9 colors in a stack [S1, unofficial].
- **Strengths:** Effective at removing isolated "Salt and Pepper" noise caused by precision errors in raycasting [S1, unofficial]. Simple to implement as a post-processing shader [S1, unofficial].
- **Limitations:** Only operates on a 3x3 neighborhood, so it may not remove larger noise clusters. It blurs edges slightly, though less than a standard box blur [S1, unofficial].
- **When to use:** When raycasting produces isolated incorrect or missing pixel colors due to precision errors [S1, unofficial].
- **When not to use:** When the noise is structured or high-frequency dithering (use [Edge-Preserving Kawase Blur](7026-edge-preserving-kawase-blur.md) instead) [S1, unofficial].

## Background
Originated in the context of raycasting-based voxel rendering, specifically as a post-processing step to handle noise from precision errors [S1, unofficial]. The method is described in a thesis by Enes Hergül (2022) [S1, unofficial].

## Sources
[S1] Raycasting-basiertes Voxelrendering, https://serwiss.bib.hs-hannover.de/frontdoor/index/index/docId/2250, fetched 2026-10-25 (unofficial)
