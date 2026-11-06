# Area-Averaged Resampling

Also called: Area Weighted Average, Box Filter Resampling, GPU Box Downsampling, box filter downsampling, downSampleImageGPU

## Overview
Area-Averaged Resampling is a downsampling technique that computes the average color of an input image region mapped to a single output pixel, weighted by the area of overlap between the input pixel grid and the mapped output pixel region [S1, unofficial]. It reduces image resolution by a factor of $2^k$ using a box filter (averaging) over a square neighborhood centered at the scaled source coordinates [S2, unofficial]. It is chosen when a physically accurate, alias-free reduction of volumetric or image data is required, distinguishing it from nearest-neighbor or simple integer-stride subsampling by accounting for partial pixel coverage [S1, S2, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** A source image/volume represented as a 2D grid of pixels/voxels. Each element contains color components (Red, Green, Blue) and optionally a density/grayscale value. The source is stored in device memory (e.g., `src.redDevice`, `src.grayDevice`) [S2, unofficial].
*   **Output:** A destination image/volume with dimensions $W_{out} \times H_{out}$, where $W_{out} = W_{in} / 2^k$ and $H_{out} = H_{in} / 2^k$. The output is stored in device memory (e.g., `dest.redDevice`, `dest.grayDevice`) [S2, unofficial].
*   **Parameter:** `downSampleTimes` (int): The exponent for the downsampling factor. The actual reduction factor is $2^{\text{downSampleTimes}}$ [S2, unofficial].

**Data Structures**
The implementation assumes access to the source and destination arrays via linear indexing.
*   Source layout: 1D array representing a 2D grid. Index for $(r, c)$ is $r \times W_{in} + c$ [S2, unofficial].
*   Destination layout: 1D array representing a 2D grid. Index for $(r, c)$ is $r \times W_{out} + c$ [S2, unofficial].
*   The source may be grayscale (`src.grayImage == true`) or RGB (`src.grayImage == false`) [S2, unofficial].

**Algorithm Steps**
The method can be implemented in two ways depending on the hardware constraints and precision requirements: a GPU Compute Shader approach using continuous area weighting [S1, unofficial], or a CUDA kernel approach using discrete integer arithmetic [S2, unofficial].

### Method 1: Continuous Area-Weighted Average (Compute Shader)
This method calculates the exact overlap area between the output pixel's mapped region and the input pixels [S1, unofficial].

1.  **Calculate Ratio:** Compute the scaling ratio between input and output dimensions:
    $$ \text{ratio} = \frac{\text{inputSize}}{\text{outputSize}} $$
    Specifically, $\text{ratio}_x = W_{in} / W_{out}$ and $\text{ratio}_y = H_{in} / H_{out}$ [S1, unofficial].

2.  **Determine Input Region:** For a destination pixel at coordinates $(u, v)$, determine the bounding box in source coordinates:
    $$ \text{inputRegionMin} = (u \cdot \text{ratio}_x, v \cdot \text{ratio}_y) $$
    $$ \text{inputRegionMax} = ((u + 1) \cdot \text{ratio}_x, (v + 1) \cdot \text{ratio}_y) $$
    $$ \text{inputRegionSize} = \text{inputRegionMax} - \text{inputRegionMin} $$
    $$ \text{totalArea} = \text{inputRegionSize}_x \cdot \text{inputRegionSize}_y $$
    [S1, unofficial].

3.  **Iterate Over Intersecting Input Pixels:** Loop over all input pixel integer coordinates $(x, y)$ that intersect the input region:
    $$ x \in [\lfloor \text{inputRegionMin}_x \rfloor, \lceil \text{inputRegionMax}_x \rceil) $$
    $$ y \in [\lfloor \text{inputRegionMin}_y \rfloor, \lceil \text{inputRegionMax}_y \rceil) $$
    [S1, unofficial].

4.  **Calculate Overlap Area:** For each input pixel $(x, y)$, define its unit square boundaries:
    $$ \text{pixelMin} = (x, y) $$
    $$ \text{pixelMax} = (x + 1, y + 1) $$
    Calculate the intersection with the input region:
    $$ \text{overlapMin} = \max(\text{pixelMin}, \text{inputRegionMin}) $$
    $$ \text{overlapMax} = \min(\text{pixelMax}, \text{inputRegionMax}) $$
    $$ \text{overlapSize} = \max(\text{overlapMax} - \text{overlapMin}, 0.0) $$
    $$ \text{overlapArea} = \text{overlapSize}_x \cdot \text{overlapSize}_y $$
    [S1, unofficial].

5.  **Accumulate Weighted Sum:**
    $$ \text{areaWeightedSum} += \text{imageLoad}(\text{inputImage}, (x, y)) \cdot \text{overlapArea} $$
    [S1, unofficial].

6.  **Normalize and Store:**
    $$ \text{areaWeightedAvg} = \frac{\text{areaWeightedSum}}{\text{totalArea}} $$
    $$ \text{imageStore}(\text{outputImage}, (u, v), \text{areaWeightedAvg}) $$
    [S1, unofficial].

### Method 2: Discrete Box Filter (CUDA Kernel)
This method uses a fixed square neighborhood centered at the scaled source coordinates, handling boundaries by skipping out-of-bounds pixels and dividing by the actual count of valid samples [S2, unofficial].

1.  **Compute Global Thread Index:**
    $$ \text{pixelPos} = (\text{blockIdx}_y \cdot \text{gridDim}_x + \text{blockIdx}_x) \cdot \text{blockDim}_x \cdot \text{blockDim}_y + \text{threadIdx}_y \cdot \text{blockDim}_x + \text{threadIdx}_x $$
    [S2, unofficial].

2.  **Check Bounds:**
    If $\text{pixelPos} \ge W_{out} \cdot H_{out}$, return [S2, unofficial].

3.  **Calculate Destination Coordinates:**
    $$ \text{pixelRow} = \lfloor \text{pixelPos} / W_{out} \rfloor $$
    $$ \text{pixelCol} = \text{pixelPos} \pmod{W_{out}} $$
    [S2, unofficial].

4.  **Initialize Accumulators:**
    $$ \text{avgDensity} = 0, \quad \text{avgRed} = 0, \quad \text{avgGreen} = 0, \quad \text{avgBlue} = 0, \quad \text{count} = 0 $$
    [S2, unofficial].

5.  **Loop Over Neighborhood:**
    Let $k = \text{downSampleTimes}$. The neighborhood radius is $k$. The source coordinates are centered around $\text{pixelRow} \cdot 2^k$ and $\text{pixelCol} \cdot 2^k$.
    For $row$ from $-k$ to $k$:
        $$ \text{srcRow} = \text{pixelRow} \cdot 2^k + row $$
        For $col$ from $-k$ to $k$:
            $$ \text{srcCol} = \text{pixelCol} \cdot 2^k + col $$
            If $\text{srcRow} < 0$ or $\text{srcRow} \ge H_{in}$ or $\text{srcCol} < 0$ or $\text{srcCol} \ge W_{in}$:
                Continue (skip) [S2, unofficial].
            Increment $\text{count}$ [S2, unofficial].
            If `src.grayImage`:
                $$ \text{avgDensity} += \text{src.grayDevice}[\text{srcRow} \cdot W_{in} + \text{srcCol}] $$
            Else:
                $$ \text{avgRed} += \text{src.redDevice}[\text{srcRow} \cdot W_{in} + \text{srcCol}] $$
                $$ \text{avgGreen} += \text{src.greenDevice}[\text{srcRow} \cdot W_{in} + \text{srcCol}] $$
                $$ \text{avgBlue} += \text{src.blueDevice}[\text{srcRow} \cdot W_{in} + \text{srcCol}] $$
    [S2, unofficial].

6.  **Write Result:**
    If `src.grayImage`:
        $$ \text{dest.grayDevice}[\text{pixelRow} \cdot W_{out} + \text{pixelCol}] = \text{avgDensity} / \text{count} $$
    Else:
        $$ \text{dest.redDevice}[\text{pixelRow} \cdot W_{out} + \text{pixelCol}] = \text{avgRed} / \text{count} $$
        $$ \text{dest.greenDevice}[\text{pixelRow} \cdot W_{out} + \text{pixelCol}] = \text{avgGreen} / \text{count} $$
        $$ \text{dest.blueDevice}[\text{pixelRow} \cdot W_{out} + \text{pixelCol}] = \text{avgBlue} / \text{count} $$
    [S2, unofficial].

**Pseudocode (CUDA-style, Method 2)**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Assumes global variables: src, dest, downSampleTimes
# src: object with .grayImage, .redDevice, .greenDevice, .blueDevice, .grayDevice, .width, .height
# dest: object with .redDevice, .greenDevice, .blueDevice, .grayDevice, .width, .height

def area_averaged_resampling_cuda():
    # Compute global thread index
    pixel_pos = (blockIdx.y * gridDim.x + blockIdx.x) * blockDim.x * blockDim.y + threadIdx.y * blockDim.x + threadIdx.x
    
    # Check bounds
    if pixel_pos >= dest.width * dest.height:
        return

    # Calculate destination coordinates
    pixel_row = pixel_pos // dest.width
    pixel_col = pixel_pos % dest.width
    
    # Initialize accumulators
    average_value_density = 0
    average_value_red = 0
    average_value_green = 0
    average_value_blue = 0
    count = 0
    
    # Loop over neighborhood
    # Note: pow(2, downSampleTimes) is computed once in practice, but shown here for clarity
    scale = 2 ** downSampleTimes
    
    for row in range(-downSampleTimes, downSampleTimes + 1):
        src_row = pixel_row * scale + row
        for col in range(-downSampleTimes, downSampleTimes + 1):
            src_col = pixel_col * scale + col
            
            # Boundary check
            if src_row < 0 or src_row >= src.height or src_col < 0 or src_col >= src.width:
                continue
            
            count += 1
            
            if src.grayImage:
                average_value_density += src.grayDevice[src_row * src.width + src_col]
            else:
                average_value_red += src.redDevice[src_row * src.width + src_col]
                average_value_green += src.greenDevice[src_row * src.width + src_col]
                average_value_blue += src.blueDevice[src_row * src.width + src_col]
                
    # Write result
    dest_index = pixel_row * dest.width + pixel_col
    
    if src.grayImage:
        dest.grayDevice[dest_index] = average_value_density / count
    else:
        dest.redDevice[dest_index] = average_value_red / count
        dest.greenDevice[dest_index] = average_value_green / count
        dest.blueDevice[dest_index] = average_value_blue / count
```

**Pseudocode (GLSL Compute Shader, Method 1)**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```glsl
#version 450
layout(local_size_x = 8, local_size_y = 8, local_size_z = 1) in;

layout(rgba8, binding = 0) uniform image2D inputImage;
layout(rgba8, binding = 1) uniform image2D outputImage;

uniform vec2 inputSize;
uniform vec2 outputSize;

void main() {
    vec2 outputCoords = vec2(gl_GlobalInvocationID.xy);
    
    // Calculate ratio
    vec2 ratio = inputSize / outputSize;
    
    // Determine input region
    vec2 inputRegionMin = outputCoords * ratio;
    vec2 inputRegionMax = (outputCoords + 1.0) * ratio;
    vec2 inputRegionSize = inputRegionMax - inputRegionMin;
    float totalArea = inputRegionSize.x * inputRegionSize.y;
    
    vec4 areaWeightedSum = vec4(0.0);
    
    // Iterate over intersecting input pixels
    ivec2 minPixel = ivec2(floor(inputRegionMin));
    ivec2 maxPixel = ivec2(ceil(inputRegionMax));
    
    for (int y = minPixel.y; y < maxPixel.y; y++) {
        for (int x = minPixel.x; x < maxPixel.x; x++) {
            // Pixel boundaries
            vec2 pixelMin = vec2(float(x), float(y));
            vec2 pixelMax = vec2(float(x) + 1.0, float(y) + 1.0);
            
            // Overlap calculation
            vec2 overlapMin = max(pixelMin, inputRegionMin);
            vec2 overlapMax = min(pixelMax, inputRegionMax);
            vec2 overlapSize = max(overlapMax - overlapMin, vec2(0.0));
            float overlapArea = overlapSize.x * overlapSize.y;
            
            // Accumulate
            areaWeightedSum += imageLoad(inputImage, ivec2(x, y)) * overlapArea;
        }
    }
    
    // Normalize and store
    vec4 areaWeightedAvg = areaWeightedSum / totalArea;
    imageStore(outputImage, ivec2(outputCoords), areaWeightedAvg);
}
```

## Operating characteristics
*   **Time Complexity:** $O(W_{out} \cdot H_{out} \cdot R^2)$ for the discrete method, where $R = 2^{\text{downSampleTimes}}$ is the radius of the neighborhood [S2, unofficial]. For the continuous method, the complexity depends on the average number of input pixels intersecting the output region, which is also proportional to the downsampling factor squared [S1, unofficial].
*   **Memory Complexity:** $O(1)$ auxiliary memory per thread/pixel, as it only requires accumulators for the current output pixel [S1, S2, unofficial].
*   **Measured Performance:** The discrete CUDA implementation uses a local size of 8x8x1 and processes `rgba8` images [S1, unofficial]. Performance scales with the number of threads and the downsampling factor. The discrete method avoids floating-point overlap calculations, potentially offering higher throughput on GPUs where integer arithmetic is efficient, but requires variable division by `count` [S2, unofficial].
*   **Strengths:** Physically accurate averaging that preserves total energy (sum of values) better than nearest-neighbor. Handles arbitrary downsampling factors (not just powers of 2) correctly in the continuous version [S1, unofficial].
*   **Limitations:** The discrete method requires division by a variable `count` which can introduce slight inconsistencies if not handled carefully (though standard floating-point division is precise enough) [S2, unofficial]. The continuous method involves more complex arithmetic (floor, ceil, max, min) per pixel [S1, unofficial].
*   **When to use:** When downsampling volumetric data or images where aliasing artifacts from simple subsampling are unacceptable, and the downsampling factor is known [S2, unofficial].
*   **When not to use:** When real-time performance is critical and the downsampling factor is small (e.g., 2x), where the overhead of area calculation might outweigh the benefit, or when nearest-neighbor is sufficient for the application [S2, unofficial].

## Background
Area-Averaged Resampling is a standard technique in image processing and volume rendering for reducing resolution while minimizing aliasing. It is often referred to as a box filter or box filter downsampling [S2, unofficial]. The method is implemented in various open-source projects for volume rendering and neural radiance fields (NeRF) processing, such as voxel ray traversal and Clear-plenoxels [S1, S2, unofficial]. The discrete CUDA implementation is specifically noted in the context of optimizing downsampling for large-scale volumetric data [S2, unofficial].

## Sources
[S1] GitHub - DeadlockCode/voxel_ray_traversal, https://github.com/DeadlockCode/voxel_ray_traversal, fetched 2026-10-25 (unofficial)
[S2] nortonBryan/Clear-plenoxels, https://github.com/nortonBryan/Clear-plenoxels, fetched 2026-10-27 (unofficial)
