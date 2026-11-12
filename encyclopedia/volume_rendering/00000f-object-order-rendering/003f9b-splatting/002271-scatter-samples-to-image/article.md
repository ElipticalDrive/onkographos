# Scatter Samples to Image

Also called: Sample Scattering to Image, ScatterSamplesToImage

## Overview
Scatter Samples to Image is a post-processing step that scatters computed sample values from arbitrary screen-space positions into a regular image grid [S1, unofficial]. It solves the problem of mapping non-uniformly distributed samples (e.g., from adaptive sampling or particle systems) to a standard raster image for visualization. Unlike [Splatting](23-splatting.md), which uses geometric kernels to blur and accumulate contributions, this method performs a direct, unfiltered assignment of values to pixel coordinates, acting as a raw data transfer mechanism rather than a rendering primitive [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   `sample_positions`: A tensor of shape $2 \times N$, where $N$ is the number of samples. Each column represents the $(x, y)$ coordinates of a sample in screen space [S1, unofficial].
*   `samples`: A tensor of shape $C \times N$, where $C$ is the number of channels. Each column contains the values for the $N$ samples [S1, unofficial].
*   `image_out`: A tensor of shape $C \times H \times W$, representing the output image [S1, unofficial].
*   `sample_mask_out`: A tensor of shape $1 \times H \times W$, containing `1` where samples were written and `0` elsewhere [S1, unofficial].
*   `default_values`: A vector of $C$ doubles providing default values for channels, used in specific interfaces like PyTorch [S1, unofficial].

**Data Structures:**
The input positions are typically floating-point coordinates. The output image is a dense grid. The mask is a binary indicator grid.

**Steps:**
1.  Initialize `image_out` with `default_values` (or leave untouched if no defaults are provided, as pixels receiving no samples are left untouched) [S1, unofficial].
2.  Initialize `sample_mask_out` to all zeros [S1, unofficial].
3.  For each sample index $k$ from $0$ to $N-1$:
    a.  Retrieve position $(x_k, y_k)$ from `sample_positions`[:, k] [S1, unofficial].
    b.  Retrieve values $v_{k,1}, \dots, v_{k,C}$ from `samples`[:, k] [S1, unofficial].
    c.  Convert $(x_k, y_k)$ to integer pixel coordinates $(i, j)$ by rounding or flooring (implementation dependent, but must be consistent with screen space mapping) [G].
    d.  Check if $(i, j)$ is within the image boundaries $[0, W-1] \times [0, H-1]$. If outside, ignore this sample [S1, unofficial].
    e.  For each channel $c$ from $1$ to $C$:
        i.  Write $v_{k,c}$ to `image_out`[c, j, i] [S1, unofficial].
    f.  Set `sample_mask_out`[0, j, i] = 1 [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def scatter_samples_to_image(sample_positions, samples, image_shape, default_values=None):
    """
    Scatters sample values to an image grid.
    
    Args:
        sample_positions: Tensor of shape (2, N), (x, y) coordinates.
        samples: Tensor of shape (C, N), values for each sample.
        image_shape: Tuple (H, W).
        default_values: Optional list/array of length C.
        
    Returns:
        image_out: Tensor of shape (C, H, W).
        sample_mask_out: Tensor of shape (1, H, W).
    """
    C, N = samples.shape
    H, W = image_shape
    
    # Initialize output tensors
    if default_values is not None:
        image_out = torch.full((C, H, W), default_values, dtype=samples.dtype)
    else:
        # If no defaults, initialize with zeros or leave undefined? 
        # Sources say "left untouched", implying we might not overwrite. 
        # For a clean implementation, we initialize to 0 or a background color.
        # We will initialize to 0 for safety, but note the "untouched" behavior.
        image_out = torch.zeros((C, H, W), dtype=samples.dtype)
        
    sample_mask_out = torch.zeros((1, H, W), dtype=torch.float32)
    
    # Iterate over samples
    for k in range(N):
        x = sample_positions[0, k]
        y = sample_positions[1, k]
        
        # Convert to integer pixel coordinates
        # Assuming standard screen space: x in [0, W), y in [0, H)
        # Using floor for consistency with typical rasterization, or round.
        # The source doesn't specify rounding mode, so we assume standard integer conversion.
        col = int(x)
        row = int(y)
        
        # Boundary check
        if 0 <= col < W and 0 <= row < H:
            # Write values
            for c in range(C):
                image_out[c, row, col] = samples[c, k]
            # Update mask
            sample_mask_out[0, row, col] = 1.0
            
    return image_out, sample_mask_out
```

**Parameters:**
*   `default_values`: A vector of doubles providing default values for the 8 channels, used in the PyTorch interface `ScatterSamplesToImage` [S1, unofficial].

**Edge Cases and Pitfalls:**
*   Samples outside image boundaries are ignored [S1, unofficial].
*   Pixels receiving no samples are left untouched (no default value overwriting) unless `default_values` are explicitly provided and applied during initialization [S1, unofficial].
*   Multiple samples may map to the same pixel; the last sample processed overwrites previous values [G].

## Subtopics

### 2D Trace Heatmap Rendering
A variant that uses scatter-like logic to render a 2D heatmap from trace data. It initializes a canvas of width `traceCount * 2` and height `sampleCount` [S2, unofficial]. It computes min and max values across the 2D array `samples[traceCount][sampleCount]`, calculating range as `max - min` (defaulting to 1 if 0 to avoid division by zero) [S2, unofficial]. For each pixel $(i, j)$, it calculates normalized value $v = (samples[i][j] - min) / range$, computes color intensity $color = \lfloor v \times 255 \rfloor$, and sets RGBA to $(color, color, color, 255)$ [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N)$, where $N$ is the number of samples, as each sample is processed once [G].
*   **Memory Complexity:** $O(C \times H \times W)$ for the output image and mask [G].
*   **Strengths:** Extremely fast for direct mapping; no kernel evaluation or sorting required [G].
*   **Limitations:** No anti-aliasing or smoothing; artifacts appear if samples are sparse or misaligned with pixels [G].
*   **When to use:** When samples are already on a grid or when a raw, unfiltered representation is desired [G].
*   **When not to use:** When smooth visualization or blending is required; use [Splatting](23-splatting.md) instead [G].

## Background
The method is described in the context of adaptive sampling and reconstruction for volume visualization, specifically within the `AdaptiveSampling` project [S1, unofficial]. It serves as a bridge between adaptive sampling algorithms and standard image formats.

## Sources
[S1] Learning adaptive sampling and reconstruction for Volume Visualization, https://github.com/shamanDevel/AdaptiveSampling, fetched 2026-10-27 (unofficial)
[S2] SeismicDataCompressor, https://github.com/rubensrudio/SeismicDataCompressor, fetched 2026-10-28 (unofficial)
