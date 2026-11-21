# Progressive Downsampling


## Overview
Progressive downsampling is a rendering optimization technique that generates a hierarchy of data representations at decreasing spatial resolutions to accelerate image synthesis [S2, unofficial]. It solves the problem of high computational cost in volume rendering by allowing the system to produce a low-quality preview quickly, then progressively refining the resolution to the final target [S2, unofficial]. Unlike static mipmapping which pre-computes all levels, progressive methods often dynamically adjust resolution during optimization or rendering, such as starting with an 8x downsampled image and gradually increasing resolution [S1]. It sets itself apart from siblings like [VoxelGrid Down Sample](1014-voxelgrid-down-sample.md) by emphasizing the temporal progression of quality rather than just spatial subsampling.

## How it works
**Inputs and Outputs:**
- Input: Original high-resolution volume data or feature maps.
- Output: A sequence of images or volumes at increasing resolutions, culminating in the final high-resolution result.

**Data Structures:**
- **Pyramid Structure:** For an original dataset of $N^3$ samples, the structure consists of a sequence of $\log_2 N$ volumes [S2, unofficial].
  - Level 0: Original data set.
  - Level $k$: Created by averaging each $2 \times 2 \times 2$ group of samples from Level $k-1$ [S2, unofficial].
- **Bit-Packed Binary Representation:** For binary sample values, bits are packed into bytes. Each byte represents a $2 \times 2 \times 2$ portion of the data (8 bits) [S2, unofficial].

**Steps:**
1. **Initial Downsampling:** Downsample the input data by a factor of 8 initially [S1].
2. **Progressive Refinement:** Gradually increase the resolution with a logarithmic decay back to the original resolution [S1].
   - In specific implementations (e.g., Trick-GS), this progression continues until a specific iteration count (e.g., iteration 19500) [S1].
3. **Binary Optimization (Optional):** If data is binary:
   - Process data bit-by-bit to generate the full-resolution image.
   - Process data byte-by-byte to generate lower-resolution images [S2, unofficial].
   - Thresholding: If more than four bits of a byte are set, the byte represents an object element; otherwise, it represents background [S2, unofficial].
4. **Resolution Modulation (Optional):** For Gaussian Splatting, apply floor modulation to the continuous resolution downsampling factor $r(i)$ to accelerate primitive growth [S3].
   - Compute continuous factor $r(i)$ [S3].
   - Apply modulation: $r_{mod}(i) = \lfloor r(i) \rfloor$ [S3].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def progressive_downsample(volume, target_iterations=19500):
    """
    Generates a progressive resolution hierarchy for volume rendering.
    """
    # Step 1: Initial 8x downsampling
    current_resolution = volume.shape / 8
    current_volume = downsample_by_8(volume)
    
    # Step 2: Progressive refinement loop
    for iteration in range(target_iterations):
        # Calculate current target resolution based on logarithmic decay
        # Note: Exact decay function not specified in sources [G]
        progress = iteration / target_iterations
        scale_factor = 1.0 + (7.0 * (1.0 - progress)) # Linear approximation of decay from 8x to 1x
        
        # Interpolate/Up-sample to current scale
        next_volume = upsample(current_volume, scale_factor)
        
        # Render current resolution
        render(next_volume)
        
        # Update for next iteration
        current_volume = next_volume

def binary_byte_rendering(binary_data):
    """
    Renders binary volume data using byte-level processing for speed.
    """
    # Pack bits into bytes (8 bits per byte representing 2x2x2 voxel block)
    byte_data = pack_bits_to_bytes(binary_data)
    
    # Generate lower resolution by processing byte by byte
    for byte in byte_data:
        # Threshold: >4 bits set means object
        if count_set_bits(byte) > 4:
            render_object_element()
        else:
            render_background()
```

**Parameters:**
- Initial downsampling factor: 8x [S1].
- Binary threshold: 4 bits (out of 8) [S2, unofficial].
- Modulation function: Floor operation $\lfloor r(i) \rfloor$ [S3].

**Edge Cases and Pitfalls:**
- **Binary Thresholding:** The "more than four bits" rule is specific to the $2 \times 2 \times 2$ bit-packing scheme [S2, unofficial].
- **Insufficient Optimization:** Without floor modulation in Gaussian Splatting, primitives grow too slowly, leading to insufficient per-primitive optimization [S3].

## Subtopics

### CorrBlockFast1D
CorrBlockFast1D is a 1D correlation block implementation for stereo matching that computes correlations using grid sampling on downsampled feature maps, supporting a multi-level pyramid structure [S1]. It pre-computes a correlation pyramid using all-pairs correlation and bilinear sampling for efficient inference [S1]. Unlike standard correlation layers that compute pairwise similarities on the fly, this method constructs a pyramid of correlation volumes between two feature maps to enable efficient multi-scale 1D correlation sampling [S1].

### Bit-Packed Binary Rendering
An optimization for binary sample values where bits are packed into bytes to generate lower resolution images faster [S2, unofficial]. It processes data bit-by-bit for full resolution and byte-by-byte for lower resolution. If more than four bits of the byte are set, the byte represents an object element; otherwise, it represents background [S2, unofficial]. This produces an image with one-half the linear resolution in approximately one-eighth the time [S2, unofficial].

### Resolution Downsampling Modulation
A technique that applies a floor operation to a continuous resolution downsampling factor to accelerate Gaussian primitive growth and ensure sufficient per-primitive optimization [S3]. It computes the continuous factor $r(i)$ and applies $r(i) = \lfloor r(i) \rfloor$ [S3]. Without this modulation, Gaussian primitives grow relatively slower, leading to insufficient optimization per primitive [S3].

## Operating characteristics
- **Time Complexity:** Generating lower resolution images via byte processing takes approximately one-eighth the time of full resolution bit processing for binary data [S2, unofficial].
- **Performance:** DashGaussian, using resolution downsampling modulation, achieves optimization in 200 seconds, a 45.7% average speedup over other backbones [S3].
- **Scaling:** The pyramid structure scales with $\log N$ volumes for $N^3$ data samples [S2, unofficial].
- **Strengths:** Rapid generation of low-quality previews for interaction; accelerated optimization in Gaussian Splatting [S2, unofficial][S3].
- **Limitations:** Lower resolution images may not always be representative of the final image [S2, unofficial].

## Background
The concept of pyramid data structures for volume visualization was established in early volume rendering literature, where hierarchical sp… (text cut off) was proposed as an efficient implementation [S2, unofficial]. The specific application of progressive downsampling with logarithmic decay in optimization was detailed in Trick-GS [S1]. Resolution downsampling modulation was introduced in DashGaussian to accelerate 3D Gaussian Splatting optimization [S3].

## Sources
[S1] Trick-GS: A Balanced Bag of Tricks for Efficient Gaussian Splatting, https://doi.org/10.1109/icassp49660.2025.10889395, fetched 2026-10-30
[S2] PDF Volume Visualization: Principles and Advances, https://courses.cs.duke.edu/spring03/cps296.8/papers/KaufmanVolumeVisualization.pdf, fetched 2026-10-28 (unofficial)
[S3] DashGaussian: Optimizing 3D Gaussian Splatting in 200 Seconds, https://doi.org/10.1109/cvpr52734.2025.01041, fetched 2026-10-31
