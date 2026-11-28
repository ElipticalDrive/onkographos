# GPU 3D Mip Chain Generation
Also called: depthSlice render passes, generateMipChain3d

## Overview
`generateMipChain3d` is a GPU primitive that fills a 3D texture's mip chain by executing per-(level, z-slice) render passes, targeting specific Z-slices via `GPURenderPassColorAttachment.depthSlice` [S1, unofficial]. It mirrors the 2D `generateMipChain` approach but extends it to 3D volumes, enabling efficient downsampling for both display (box filter) and acceleration (max-value pyramid) purposes [S1, unofficial]. Unlike standard mipmaps which store averaged or filtered values for display, this method supports building a per-field max pyramid in deviation space to enable transfer-function-adaptive skipping during volume ray marching, allowing dynamic region skipping based on live contrast settings without rebuilding the structure [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: Source 3D texture, target mip level index $L$, target Z-slice index $Z_{dst}$ [S1, unofficial].
- Output: Destination 3D texture mip level $L$, specifically the slice at $Z_{dst}$ [S1, unofficial].

**Data Structures:**
The uniform buffer `MipBlit3dUniforms` contains:
- `boxZ`: float, Z-coordinate of the source slice [S1, unofficial].
- `dstZ`: float, Z-coordinate of the destination slice [S1, unofficial].
- `center`: vec2<f32>, center of the sampling window in normalized texture coordinates [S1, unofficial].
- `halfRange`: vec2<f32>, half-width of the sampling window in normalized texture coordinates [S1, unofficial].
Total size: 16 bytes, unpadded [S1, unofficial].

**Algorithm Steps:**
1. Iterate through each mip level $L$ from 1 to $N-1$ (where $N$ is the total number of mip levels) [S1, unofficial].
2. For each level $L$, determine the destination dimensions $D'_x, D'_y, D'_z$ (typically $\lfloor D_x/2 \rfloor, \lfloor D_y/2 \rfloor, \lfloor D_z/2 \rfloor$) [S1, unofficial].
3. Iterate through each destination Z-slice $Z_{dst}$ from 0 to $D'_z - 1$ [S1, unofficial].
4. For each $(L, Z_{dst})$, execute a render pass:
   - Set `GPURenderPassColorAttachment.depthSlice` to $Z_{dst}$ for the destination texture [S1, unofficial].
   - Bind the source texture's mip level $L-1$ [S1, unofficial].
   - Execute a fragment shader that performs the downsampling (box or max) over the source volume [S1, unofficial].
   - The shader uses `tapRange(d, parentDim)` to determine the source sampling footprint for each axis $d \in \{x, y, z\}$ [S1, unofficial].

**Footprint Calculation (`tapRange`):**
To handle odd parent dimensions correctly, the source sampling range for an axis with parent dimension $P$ and destination index $d$ is calculated as:
$$ \text{start} = 2d $$
$$ \text{end} = \min(2d + 2, P - 1) $$
This ensures a 3-wide footprint $(2d, 2d+1, P-1)$ is used at the last destination index on an odd axis, preventing the silent drop of the trailing slice/row/column that occurred with the old fixed 2-wide footprint and `min(2d+1, dim-1)` clamp [S1, unofficial]. For example, on dimensions $178 \times 300 \times 182$, the old method dropped $z=90$ and $x=88$ on level 1->2 [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def tapRange(d, parentDim):
    # Returns inclusive start and end indices for the source footprint
    start = 2 * d
    end = min(2 * d + 2, parentDim - 1)
    return start, end

def generate_mip_chain_3d(src_data, dst_data, src_dims, dst_dims, num_levels, filter_type='box'):
    # src_data and dst_data are lists of 3D arrays representing mip levels
    # src_dims and dst_dims are lists of (x, y, z) tuples for each mip level
    # num_levels is the total number of mip levels (including level 0)
    
    for level in range(1, num_levels):
        s_dim = src_dims[level - 1]
        d_dim = dst_dims[level]
        
        # Iterate over destination Z-slices, executing a render pass per slice
        for dst_z in range(d_dim[2]):
            # Render pass targets dst_data at mip level 'level', depth slice 'dst_z'
            # Binds src_data at mip level 'level - 1'
            # Uniforms: boxZ, dstZ, center, halfRange (16 bytes total)
            
            # Rasterizer handles dst_x and dst_y loops; shader computes footprint
            for dst_y in range(d_dim[1]):
                for dst_x in range(d_dim[0]):
                    # Calculate source footprint for each axis using tapRange
                    src_x_start, src_x_end = tapRange(dst_x, s_dim[0])
                    src_y_start, src_y_end = tapRange(dst_y, s_dim[1])
                    src_z_start, src_z_end = tapRange(dst_z, s_dim[2])
                    
                    value = 0.0
                    count = 0
                    for src_z in range(src_z_start, src_z_end + 1):
                        for src_y in range(src_y_start, src_y_end + 1):
                            for src_x in range(src_x_start, src_x_end + 1):
                                sample = src_data[level - 1][src_z][src_y][src_x]
                                if filter_type == 'box':
                                    value += sample
                                    count += 1
                                elif filter_type == 'max':
                                    value = max(value, sample)
                    
                    if filter_type == 'box' and count > 0:
                        value /= count
                    
                    dst_data[level][dst_z][dst_y][dst_x] = value
```

**Edge Cases and Pitfalls:**
- **Odd Dimensions:** A fixed 2-wide footprint in the max-reduction shader (`fs_max`) silently dropped the last parent slice/row/column when a parent dimension was odd. This was fixed by using `tapRange(d, parentDim)` [S1, unofficial].
- **Precision:** Not given in the sources [S1, unofficial].
- **Boundaries:** The `tapRange` function handles boundaries by clamping to `0` and `parentDim - 1` [S1, unofficial].

## Subtopics

### [Per-Field Max Pyramid in Deviation Space](5311-per-field-max-pyramid-in-deviation-space.md)
A max-value pyramid stored in deviation space (normalized distance from contrast center) used for transfer-function-adaptive skipping during volume ray marching. It is built from the raw volume cube, not the display mip chain, to avoid hiding thin bright filaments that might be smoothed out by box filtering. Unlike standard mipmaps which store averaged or filtered values for display, this structure stores the maximum possible signal intensity relative to a threshold, enabling the renderer to skip empty or low-intensity regions dynamically based on live contrast settings.

## Operating characteristics
- **Time Complexity:** Not given in the sources [S1, unofficial].
- **Memory Complexity:** Not given in the sources [S1, unofficial].
- **Measured Performance:** Not given in the sources [S1, unofficial].
- **Scaling:** The method executes per-(level, z-slice) render passes to process the volume [S1, unofficial].
- **Strengths:** Efficiently generates 3D mip chains on the GPU. Supports both box filtering for display and max filtering for acceleration [S1, unofficial]. Handles odd dimensions correctly [S1, unofficial].
- **Limitations:** Requires multiple render passes (one per Z-slice per mip level) [S1, unofficial].
- **When to use:** When generating mip chains for 3D textures on the GPU, especially for volume rendering where both display and acceleration pyramids are needed [S1, unofficial].
- **When not to use:** Not given in the sources [S1, unofficial].

## Background
`generateMipChain3d` was introduced as part of a GPU primitive for filling 3D texture mip chains via per-(level, z-slice) render passes, mirroring the 2D `generateMipChain` approach [S1, unofficial]. It was developed to support volume ray marching acceleration, specifically for building per-field max pyramids in deviation space for transfer-function-adaptive skipping [S1, unofficial]. The method was refined to fix issues with odd dimensions in the max-reduction shader [S1, unofficial].

## Sources
[S1] feat(volumes): raymarch acceleration — GPU pyramids, TF-adaptive skipping, cone LOD, https://github.com/rulkens/skymap/pull/556, fetched 2026-10-28 (unofficial)
