# Luminance Scaling

Also called: Image Data Type Casting, Luminance Mapping

## Overview
Luminance Scaling converts a multi-component unsigned char volume (typically RGB) into a single-component unsigned short volume by computing a weighted sum of the color channels to approximate human perceptual brightness. It solves the problem of reducing multi-channel data to a single scalar field for rendering pipelines that do not support independent component processing, while preserving the dynamic range by mapping the result to a 16-bit integer space. It differs from standard grayscale conversion by outputting a 16-bit value scaled by 255.0 rather than an 8-bit value, and from Luminance Conversion by not appending the result to the original data but replacing the volume structure.

## How it works
**Inputs:**
- A volume dataset with extent $(I, J, K)$ and data type `VTK_UNSIGNED_CHAR` (8-bit unsigned integer).
- The data consists of $N = I \times J \times K$ voxels, each containing 3 components ($C_0, C_1, C_2$) representing Red, Green, and Blue.

**Outputs:**
- A volume dataset with the same extent $(I, J, K)$.
- The data type is `VTK_UNSIGNED_SHORT` (16-bit unsigned integer).
- The data consists of $N$ voxels, each containing 1 component.

**Data Structures:**
- Source pointer: `unsigned char*` pointing to the contiguous array of 3-byte tuples.
- Destination pointer: `unsigned short*` pointing to the contiguous array of 2-byte values.

**Formula:**
The output value $V_{out}$ for each voxel is calculated as:
$$ V_{out} = (0.299 \cdot C_0 + 0.587 \cdot C_1 + 0.114 \cdot C_2) \times 255.0 $$
where $C_0, C_1, C_2$ are the input scalar components (Red, Green, Blue) treated as floating-point values in the range $[0, 255]$.

**Parameters:**
- Weights: $0.299$ (Red), $0.587$ (Green), $0.114$ (Blue). These are standard ITU-R BT.601 luminance coefficients.
- Scale factor: $255.0$.

**Steps:**
1. Retrieve the volume extent $(I, J, K)$.
2. Allocate a destination buffer of type `unsigned short` with size $I \times J \times K$.
3. Cast the source data pointer to `unsigned char*` and the destination pointer to `unsigned short*`.
4. Iterate over each voxel index $(i, j, k)$ from $0$ to $I \times J \times K - 1$:
   a. Read the three source components:
      $$ R = \text{src}[3 \cdot i] $$
      $$ G = \text{src}[3 \cdot i + 1] $$
      $$ B = \text{src}[3 \cdot i + 2] $$
   b. Compute the weighted sum:
      $$ \text{luminance} = 0.299 \cdot R + 0.587 \cdot G + 0.114 \cdot B $$
   c. Scale to the 16-bit range:
      $$ \text{temp} = \text{luminance} \times 255.0 $$
   d. Cast to `unsigned short` and assign to the destination:
      $$ \text{dst}[i] = \text{static\_cast<unsigned short>}(temp) $$
5. Return the destination volume.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def luminance_scaling(src_data: list[int], extent: tuple[int, int, int]) -> list[int]:
    """
    Converts a multi-component unsigned char volume to a single-component unsigned short volume.
    
    Args:
        src_data: List of integers (0-255) representing the RGB volume data.
                  Length must be 3 * I * J * K.
        extent: Tuple (I, J, K) representing the volume dimensions.
        
    Returns:
        List of integers (0-65535) representing the luminance volume data.
    """
    I, J, K = extent
    num_voxels = I * J * K
    dst_data = [0] * num_voxels
    
    # Cast source pointer to unsigned char* (handled by list of ints)
    # Cast destination pointer to unsigned short* (handled by list of ints)
    
    for i in range(num_voxels):
        # Read components
        r = src_data[3 * i]
        g = src_data[3 * i + 1]
        b = src_data[3 * i + 2]
        
        # Compute luminance
        luminance = 0.299 * r + 0.587 * g + 0.114 * b
        
        # Scale to 0-255 range and then to 16-bit
        # Note: The source formula multiplies by 255.0 after the weighted sum.
        # Since the weighted sum of 8-bit values is in [0, 255], multiplying by 255
        # maps it to [0, 65025].
        temp = luminance * 255.0
        
        # Assign to destination (cast to unsigned short)
        dst_data[i] = int(temp)
        
    return dst_data
```

**Edge Cases and Pitfalls:**
- **Precision:** The intermediate calculation uses floating-point arithmetic. The final cast to `unsigned short` truncates the decimal part.
- **Range:** The maximum possible output value is $255 \times 255.0 = 65025$, which fits within a 16-bit unsigned integer (max 65535).
- **Input Type:** The method assumes the input is `VTK_UNSIGNED_CHAR`. If the input is a different type, the behavior is undefined or requires prior casting.

## Subtopics

### Luminance Conversion
Converts a multi-scalar-component volume (e.g., RGB) into a single-channel representation by computing luminance and appending it to the original data. This is used when independent component rendering is disabled. The formula is $L = 0.299 \cdot C_0 + 0.587 \cdot C_1 + 0.114 \cdot C_2$. The parameter `independentcomponents` (bool) controls whether multi-component volumes are merged via luminance appending (false) or processed separately (true) [S1, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N)$, where $N$ is the number of voxels. Each voxel requires a constant number of arithmetic operations.
- **Memory Complexity:** $O(N)$ for the output buffer. The input buffer is read-only.
- **Performance:** The operation is memory-bandwidth bound due to the conversion from 3 bytes to 2 bytes per voxel. It is computationally lightweight.
- **Strengths:** Preserves perceptual brightness information; reduces data dimensionality for single-channel rendering; outputs 16-bit data for higher dynamic range than 8-bit grayscale.
- **Limitations:** Loses color information; assumes RGB input; not suitable for multi-component rendering where channels are independent.
- **When to use:** When rendering multi-component volumes in a pipeline that only supports single-channel scalar fields and independent component processing is disabled.
- **When not to use:** When color information is critical or when independent component rendering is enabled.

## Background
Luminance Scaling is implemented in the `wxMaracasSurfaceRenderingManager` class within the `QtVTKViewer` project. It is used to prepare multi-component volume data for surface rendering when the rendering engine expects a single-channel scalar field. The method leverages standard luminance coefficients to approximate human perception of brightness.

## Sources
[S1] juanprietob/QtVTKViewer, https://github.com/juanprietob/QtVTKViewer, fetched 2026-10-27 (unofficial)
