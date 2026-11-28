# DCT-compressed time-runs rendering

Also called: Lum et al. temporal encoding, Lum, and Clyne DCT method, Three-level cache for time-varying data

## Overview
This method enables interactive rendering of large-scale time-varying volumetric datasets by treating video memory, main memory, and disk as a three-level cache [S1]. It solves the bandwidth bottleneck of transferring thousands of time steps by compressing temporal sequences of voxel values into single scalar indices using the Discrete Cosine Transform (DCT) [S2, unofficial]. Unlike sibling methods that compress color or opacity data, this approach compresses the index data itself, allowing color and opacity transfer functions to be modified without recompression [S1]. It sets itself apart by leveraging hardware palette manipulation to decode time-varying data on the fly, avoiding the need for complex on-the-fly inverse transforms during rendering [S1].

## How it works
**Inputs:** A time-varying volumetric dataset $V$ with dimensions $W \times H \times D$ and $T$ time steps [S1][S2, unofficial].
**Outputs:** A set of texture maps (one per spatial slice) containing compressed scalar indices, and a mechanism for time-varying palette updates [S2, unofficial].

**Data Structures:**
1. **Texture Maps:** One 2D texture map per spatial slice (e.g., $Z$-slice) [S2, unofficial]. Each texel stores a single scalar index $I$ [S2, unofficial]. The size of the index depends on the quantization precision (e.g., 8-bit) [S1].
2. **Color Palette:** A lookup table mapping scalar indices to RGBA colors [G]. This palette is updated dynamically per time step [S1].

**Encoding Process:**
1. **Temporal Grouping:** For each spatial voxel position $(x, y, z)$, extract the sequence of scalar values $S = [v_0, v_1, \dots, v_{N-1}]$ across $N$ consecutive time steps [S1]. The window size $N$ is a parameter (e.g., $N=4$) [S1].
2. **DCT Transformation:** Apply a 1D Discrete Cosine Transform to the sequence $S$ to obtain coefficients $C = [c_0, c_1, \dots, c_{N-1}]$ [S1].
   $$ c_k = \alpha_k \sum_{n=0}^{N-1} v_n \cos\left(\frac{\pi(2n+1)k}{2N}\right) $$
   where $\alpha_0 = \sqrt{1/N}$ and $\alpha_k = \sqrt{2/N}$ for $k > 0$ [G].
3. **Quantization and Truncation:** Retain only the first $M$ coefficients (typically $M < N$, e.g., $M=3$) which contain the majority of the energy [S1]. Quantize these coefficients to fit into a smaller bit-depth (e.g., store the three retained coefficients into a single 8-bit value) [S1].
4. **Index Mapping:** The quantized coefficients are mapped to a single scalar index $I$ [S1]. This index serves as a placeholder for the temporal sequence [S1].
5. **Optimal Reconstruction Mapping:** Instead of using the standard Inverse DCT to recover values, calculate the optimal mapping for each index $I$ for each time frame $t$ [S1]. This mapping minimizes the mean square error (MSE) between the decoded value and the original scalar value $v_t$ [S1]. This pre-computed mapping is stored implicitly or explicitly to allow fast decoding [S1].
6. **Texture Population:** Store the index $I$ in the corresponding texel of the texture map for the voxel's spatial location [S2, unofficial].

**Decoding and Rendering Process:**
1. **Texture Sampling:** During rendering, sample the texture map to retrieve the scalar index $I$ for each voxel [S2, unofficial].
2. **Palette Update:** Before rendering a specific time step $t$, update the color palette such that the entry for index $I$ maps to the color corresponding to the reconstructed value for time $t$ [S1].
3. **Hardware Decoding:** Use an extension of OpenGL (or equivalent graphics API) that allows the color palette of all loaded textures to be changed simultaneously without reloading the texture data [S1].
4. **Rendering:** Perform standard volume rendering (e.g., ray casting or texture-based slicing) using the updated palettes [S1]. The graphics hardware maps the sampled indices to the correct time-step colors via the updated palette [S1][S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def encode_time_runs(volume_data, window_size=4, num_coeffs=3):
    """
    Compress time-runs of voxels into scalar indices.
    volume_data: 4D array [W, H, D, T]
    Returns: texture_maps (list of 2D arrays), reconstruction_map
    """
    W, H, D, T = len(volume_data), len(volume_data[0]), len(volume_data[0][0]), len(volume_data[0][0][0])
    texture_maps = []
    reconstruction_map = {} 
    
    for z in range(D):
        slice_map = [[0] * W for _ in range(H)]
        for x in range(W):
            for y in range(H):
                seq = [volume_data[x][y][z][t] for t in range(window_size)]
                
                # 1. DCT
                dct_coeffs = [0.0] * window_size
                for k in range(window_size):
                    s = sum(seq[n] * math.cos(math.pi * (2*n + 1) * k / (2 * window_size)) for n in range(window_size))
                    alpha = math.sqrt(1.0 / window_size) if k == 0 else math.sqrt(2.0 / window_size)
                    dct_coeffs[k] = alpha * s
                
                # 2. Quantize and truncate
                quantized = [0] * num_coeffs
                for i in range(num_coeffs):
                    # Map to 0-255 range for 8-bit storage
                    quantized[i] = int(max(0, min(255, round(dct_coeffs[i] + 128))) & 0xFF)
                
                # 3. Map to index (pack 3 bytes into one integer)
                index = (quantized[0] << 16) | (quantized[1] << 8) | quantized[2]
                
                slice_map[y][x] = index
                
                # 4. Calculate optimal reconstruction values for each time step
                for t in range(window_size):
                    if t not in reconstruction_map:
                        reconstruction_map[t] = {}
                    reconstruction_map[t][index] = seq[t]
                    
        texture_maps.append(slice_map)
        
    return texture_maps, reconstruction_map

def render_time_step(texture_maps, reconstruction_map, time_step, transfer_function):
    """
    Render a specific time step using palette manipulation.
    transfer_function: callable mapping scalar value to RGBA color
    """
    palette_size = 256 * 256 * 256
    current_palette = [[0, 0, 0, 0] for _ in range(palette_size)]
    
    for index, value in reconstruction_map[time_step].items():
        color = transfer_function(value)
        current_palette[index] = color
        
    # Update hardware palette for all textures simultaneously
    # (Conceptual API: glPixelMapfv(GL_PIXEL_MAP_I_TO_R, ...), etc.)
    update_hardware_palette(current_palette)
    
    # Render each slice texture
    for slice_map in texture_maps:
        draw_textured_geometry(slice_map)

def update_hardware_palette(palette):
    """
    Updates the color palette of all loaded textures simultaneously.
    Implements the OpenGL extension described in [S1] to avoid reloading textures.
    """
    pass  # Hardware API call to bind palette to all active texture units

def draw_textured_geometry(slice_map):
    """
    Renders a 2D slice using the texture map and current palette.
    """
    pass  # Hardware API call to draw textured polygons using the updated palette
```

**Parameters:**
*   **Window Size ($N$):** Number of time steps grouped together. Typical value: 4 [S1].
*   **Number of Coefficients ($M$):** Number of DCT coefficients retained. Typical value: 3 [S1].
*   **Bit Depth:** Precision of the stored index. Typical value: 8-bit [S1].

**Edge Cases:**
*   **Boundary Time Steps:** If the total number of time steps $T$ is not a multiple of the window size, the last group must be handled (e.g., padded or processed with a smaller window) [G].
*   **Index Collisions:** Different temporal sequences may map to the same index after quantization. The optimal reconstruction mapping handles this by assigning the best average value for that index at each time step [S1].

## Operating characteristics
*   **Compression Ratio:** Achieves approximately 4x compression [S1].
*   **Performance (Out-of-Core):** On an AMD 1.2 GHz Athlon with 768 MB main memory and a GeForce 3 with 64 MB texture memory, 1492 time steps of a $256 \times 256 \times 256$ dataset are rendered at approximately 6.8 frames/s [S1].
*   **Performance (In-Core):** If main memory holds 280 time steps, rendering speed increases to 25.8 frames/s [S1].
*   **Comparison:** Without compression, a memory-resident subset of 280 time steps renders at ~11.5 frames/s. Compression allows more data to fit in memory, significantly boosting performance compared to uncompressed out-of-core swapping [S1].
*   **Strengths:** Fast decoding via hardware palette updates; decoupling of data compression from transfer function modification; good visual quality despite lossy compression [S1].
*   **Limitations:** Lossy compression; requires graphics hardware support for palette manipulation; limited by the temporal coherence of the data (works best when voxels change slowly over time) [S1].

## Background
The method was developed by Lum, Ma, and Clyne and published in 2003 [S1]. It addresses the challenge of rendering large time-varying volumes on commodity PC hardware by exploiting the three-level cache hierarchy (disk, main memory, video memory) [S1]. The approach builds on the principle that temporal redundancy in volume data can be exploited similarly to video compression, but adapted for interactive volume rendering via indexed textures and palette manipulation [S1].

## Sources
[S1] Recent advances in hardware-accelerated volume rendering, https://doi.org/10.1016/s0097-8493(03)00146-8, fetched 2026-10-26
[S2] Overview of Volume Rendering - Stony Brook Computer Science, https://www3.cs.stonybrook.edu/~mueller/papers/volvisOverview.pdf, fetched 2026-10-27 (unofficial)
