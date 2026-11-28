# GPU Volume Slicing (Advanced/Transform)

## Overview
This method renders a 2D slice from a 3D volume by applying an arbitrary 4x4 transformation matrix to screen coordinates, followed by scaling and centering, before sampling the volume texture. It differs from basic slice extraction by supporting complex spatial transformations (rotation, translation, non-uniform scaling) directly in the pixel shader, rather than relying on pre-computed slice geometries or fixed-axis alignment. It builds on [2D Slice Coordinate Mapping](21041-2d-slice-coordinate-mapping.md) by generalizing the coordinate mapping to include full affine transformations.

## How it works
**Inputs:**
- `texVolume`: 3D texture containing the volume data.
- `trans`: 16-element array representing a 4x4 transformation matrix.
- `scale`: 4-component vector (`scale.x`, `scale.y`, `scale.z`, `scale.w`) used for scaling and centering.
- `bp`: Buffer parameters including `width` and `height`.
- `buffer`: Output buffer for the 2D slice.

**Outputs:**
- `buffer`: 2D array of floating-point values representing the sliced volume.

**Data Structures:**
- `trans`: `float[16]` or equivalent, stored in column-major or row-major order consistent with the matrix multiplication implementation. The source uses a linear index mapping: `trans[0..3]` for the first row, `trans[4..7]` for the second, etc.
- `scale`: `float4` where `x`, `y`, `z` are scaling factors and `w` is unused (set to 0.f in centering).
- `buffer`: 1D array of size `width * height`, indexed by `offset = j * width + i`.

**Steps:**
1. **Launch Configuration:** The kernel is launched with a grid of `dim3(dim/16, dim/16)` blocks, where `dim = 1024`. Each block has `dim3 threads(16, 16)` threads.
2. **Thread Indexing:** Each thread calculates its pixel coordinates `i` (x) and `j` (y) based on its global thread ID. The linear offset into the output buffer is calculated as `offset = j * bp.width + i`.
3. **Boundary Check:** If `j >= bp.height` or `i >= bp.width`, the thread exits immediately.
4. **Screen Coordinate Construction:** A `float4 raw` vector is constructed:
   - `raw.x = i / bp.width`
   - `raw.y = j / bp.height`
   - `raw.z = 0.5`
   - `raw.w = 1.0`
5. **Matrix Transformation:** The transformation matrix `trans` is applied to `raw` to produce `pos`:
   - `pos.x = trans[0]*raw.x + trans[1]*raw.y + trans[2]*raw.z + trans[3]*raw.w`
   - `pos.y = trans[4]*raw.x + trans[5]*raw.y + trans[6]*raw.z + trans[7]*raw.w`
   - `pos.z = trans[8]*raw.x + trans[9]*raw.y + trans[10]*raw.z + trans[11]*raw.w`
   - Note: `pos.w` is not used in subsequent steps.
6. **Scaling:** The `pos` components are divided by the corresponding components of `scale`:
   - `pos.x /= scale.x`
   - `pos.y /= scale.y`
   - `pos.z /= scale.z`
7. **Centering:** The position is centered using the formula:
   - `pos = (pos - 0.5) / float4(scale.x, scale.y, scale.z, 0.0) + 0.5`
   - Note: The division by `scale.w` (0.0) is avoided by setting `scale.w` to 0.0 in the centering vector, effectively ignoring the w-component of `pos` in this step.
8. **Texture Sampling:** The `pos` vector is cast to `float3`. If the coordinates are within the valid range [0, 1] for all components, the volume is sampled using `tex3D(texVolume, pos.x, pos.y, pos.z)`. Otherwise, the sample value is 0.0.
9. **Output:** The sampled value is written to `buffer[offset]`.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def advanced_slice_kernel(texVolume, trans, scale, bp, buffer):
    # Calculate thread indices
    i = blockIdx.x * blockDim.x + threadIdx.x
    j = blockIdx.y * blockDim.y + threadIdx.y

    # Boundary check
    if j >= bp.height or i >= bp.width:
        return

    # Calculate linear offset
    offset = j * bp.width + i

    # Screen coordinate construction
    raw_x = i / bp.width
    raw_y = j / bp.height
    raw_z = 0.5
    raw_w = 1.0

    # Matrix transformation
    pos_x = trans[0]*raw_x + trans[1]*raw_y + trans[2]*raw_z + trans[3]*raw_w
    pos_y = trans[4]*raw_x + trans[5]*raw_y + trans[6]*raw_z + trans[7]*raw_w
    pos_z = trans[8]*raw_x + trans[9]*raw_y + trans[10]*raw_z + trans[11]*raw_w

    # Scaling
    pos_x /= scale.x
    pos_y /= scale.y
    pos_z /= scale.z

    # Centering
    # Note: scale.w is 0.0, so we divide by scale.x, scale.y, scale.z only
    pos_x = (pos_x - 0.5) / scale.x + 0.5
    pos_y = (pos_y - 0.5) / scale.y + 0.5
    pos_z = (pos_z - 0.5) / scale.z + 0.5

    # Texture sampling
    if 0.0 <= pos_x <= 1.0 and 0.0 <= pos_y <= 1.0 and 0.0 <= pos_z <= 1.0:
        sample = tex3D(texVolume, pos_x, pos_y, pos_z)
    else:
        sample = 0.0

    # Output
    buffer[offset] = sample
```

**Parameters:**
- `dim`: 1024 (grid dimension)
- `blockDim`: 16x16 (thread block dimension)
- `resolutionScale`: Used for display resolution, not directly in the kernel.

**Edge Cases and Pitfalls:**
- **Division by Zero:** If any component of `scale` is zero, division by zero occurs. Ensure `scale.x`, `scale.y`, `scale.z` are non-zero.
- **Out-of-Bounds Sampling:** The bounds check ensures that only valid texture coordinates are sampled. Values outside [0, 1] result in 0.0.
- **Precision:** Floating-point precision issues may arise with large transformation matrices or extreme scaling factors.

## Operating characteristics
- **Time Complexity:** O(1) per pixel, as each thread processes one pixel independently.
- **Memory Complexity:** O(width * height) for the output buffer.
- **Performance:** Performance depends on the resolution and the complexity of the transformation matrix. The use of a fixed grid size (1024x1024) may lead to inefficiencies for lower resolutions.
- **Strengths:** Supports arbitrary transformations, allowing for flexible slice orientations.
- **Limitations:** Fixed grid size may not be optimal for all resolutions. Requires careful handling of scaling and centering to avoid artifacts.

## Background
This method is derived from the `volume-viz` project by Jacob Stern [S1, unofficial]. It is part of a broader effort to visualize 3D volumes on the GPU using CUDA. The approach is similar to other GPU-based volume rendering techniques but focuses on the specific case of slice extraction with arbitrary transformations.

## Sources
[S1] jacobstern/volume-viz, https://github.com/jacobstern/volume-viz, fetched 2026-10-27 (unofficial)
