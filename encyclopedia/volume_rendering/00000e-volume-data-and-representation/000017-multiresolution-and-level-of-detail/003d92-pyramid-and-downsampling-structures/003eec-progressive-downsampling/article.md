# Progressive Downsampling


## Overview
Progressive downsampling is a rendering technique that generates a sequence of images at increasing resolutions to provide immediate visual feedback while the final high-resolution image is computed [S2, unofficial]. It solves the latency problem in interactive volume visualization by allowing users to perceive the scene structure early, refining details only as computational budget permits [S2, unofficial]. Unlike static multi-resolution structures like Mipmaps which pre-compute all levels, progressive downsampling dynamically controls the resolution during the rendering pass itself, often using bit-packing optimizations for binary data or hierarchical pyramid structures for general data [S2, unofficial].

## How it works
The method operates by iteratively refining the rendered image resolution. Two primary implementation strategies exist: bit-packing for binary data and hierarchical pyramid traversal for general data.

### Inputs and Outputs
- **Input**: A volume dataset $V$ of size $N \times N \times N$ (or binary bit-field), and a target iteration count or time budget.
- **Output**: A sequence of images $I_0, I_1, \dots, I_k$ where resolution increases monotonically.

### Data Structures
1. **Binary Bit-Packed Volume**: For binary data, voxels are packed into bytes. Each byte represents a $2 \times 2 \times 2$ block of voxels (8 bits) [S2, unofficial].
2. **Pyramid Structure**: For general data, a hierarchy of volumes is pre-computed. The base level is the original $N^3$ data. Each subsequent level $L_{i+1}$ is derived from $L_i$ by averaging $2 \times 2 \times 2$ groups of samples, resulting in a volume of $1/8$ the resolution [S2, unofficial]. The pyramid contains $\log_2 N$ levels [S2, unofficial].

### Steps
1. **Initialization**: Determine the starting resolution. For binary data, start with full resolution bit-by-bit processing. For pyramid data, start with the lowest resolution level (smallest volume) [S2, unofficial].
2. **Rendering Loop**:
   - Render the current resolution level.
   - If using bit-packing: Process bits sequentially. A byte is considered "object" if more than 4 bits are set; otherwise, it is "background" [S2, unofficial].
   - If using pyramid: Render the current pyramid level.
   - Increase resolution:
     - **Binary**: Move to the next bit or byte group.
     - **Pyramid**: Move to the next higher-resolution level in the pyramid.
     - **Logarithmic Decay**: In some implementations (e.g., Trick-GS), initial downsampling (e.g., 8x) is applied, then resolution increases logarithmically back to original over a fixed number of iterations (e.g., 19,500) [S1].
3. **Termination**: Stop when the target resolution is reached or the time budget expires.

### Formulas
- **Binary Thresholding**: For a byte $B$ representing 8 voxels, the voxel value $v$ is:
  $$
  v = \begin{cases} 
  \text{Object} & \text{if } \sum_{i=0}^{7} b_i > 4 \\
  \text{Background} & \text{otherwise}
  \end{cases}
  $$
  where $b_i$ are the bits in $B$ [S2, unofficial].
- **Pyramid Resolution**: If the original volume has $N$ samples per dimension, the $k$-th pyramid level has resolution $\frac{N}{2^k}$ [S2, unofficial].

### Parameters
- **Initial Downsample Factor**: 8x (common in iterative optimization schemes) [S1].
- **Threshold for Binary**: 4 bits out of 8 (50% density) [S2, unofficial].
- **Iterations**: 19,500 (for logarithmic decay schedule) [S1].

### Edge Cases and Pitfalls
- **Partial Image Representativeness**: Lower resolution images may not accurately represent the final image's features, potentially misleading the user [S2, unofficial].
- **Precision**: Bit-packing assumes binary data; applying it to scalar data requires thresholding, which loses information [S2, unofficial].
- **Boundary Conditions**: Pyramid construction must handle non-power-of-two dimensions by padding or truncation (Not given in the sources).

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def progressive_downsample(volume, max_iterations=19500, initial_downsample=8):
    """
    Renders volume progressively from low to high resolution.
    volume: 3D array or bit-packed binary structure
    """
    # Determine initial resolution
    current_res = volume.shape[0] // initial_downsample
    
    for iteration in range(max_iterations):
        # 1. Render current resolution
        image = render_at_resolution(volume, current_res)
        display(image)
        
        # 2. Increase resolution logarithmically
        # Linear interpolation of resolution from initial_downsample to 1
        progress = iteration / max_iterations
        # Logarithmic decay schedule: resolution increases faster at first
        scale_factor = initial_downsample * (1 - progress) 
        # Ensure scale_factor >= 1
        scale_factor = max(1, scale_factor)
        
        new_res = round(volume.shape[0] / scale_factor)
        
        # 3. Update resolution if changed
        if new_res != current_res:
            current_res = new_res
            
        # 4. Handle binary bit-packing specific logic if applicable
        if volume.is_binary_packed:
            # Process next byte/bit group
            pass
```

## Subtopics

### CorrBlockFast1D
CorrBlockFast1D is a 1D correlation block implementation for stereo matching that computes correlations using grid sampling on downsampled feature maps, supporting a multi-level pyramid structure [S1]. It pre-computes a correlation pyramid using all-pairs correlation and bilinear sampling for efficient inference [S1]. Unlike standard correlation layers that compute pairwise similarities on the fly, this method constructs a pyramid of correlation volumes between two feature maps to enable efficient multi-scale 1D correlation sampling [S1].

### Bit-Packed Binary Rendering
An optimization for binary sample values where bits are packed into bytes to generate lower resolution images faster [S2, unofficial]. It processes data bit by bit to generate full resolution images, but generates lower resolution images by processing data byte by byte [S2, unofficial]. If more than four bits of the byte are set, the byte represents an element of the object; otherwise, it represents background [S2, unofficial]. This produces an image with one-half the linear resolution in approximately one-eighth the time [S2, unofficial].

## Operating characteristics
- **Time Complexity**: For bit-packed binary data, rendering at half linear resolution takes approximately $1/8$ the time of full resolution [S2, unofficial]. For pyramid-based methods, complexity depends on the number of levels $\log N$ and the cost of splatting or ray-marching at each level [S2, unofficial].
- **Memory Complexity**: Pyramid structures require $O(N^3)$ total memory for the hierarchy (geometric series sum $\approx \frac{8}{7} N^3$) [S2, unofficial]. Bit-packing reduces memory by a factor of 8 for binary data [S2, unofficial].
- **Performance**: Provides immediate visual feedback, improving interactivity [S2, unofficial]. However, lower resolution images may not be representative of the final result [S2, unofficial].
- **Strengths**: Fast initial image generation; efficient use of hardware for binary data.
- **Limitations**: Requires pre-computation for pyramid structures; bit-packing is limited to binary data.

## Background
The concept of progressive downsampling originated in the context of interactive volume visualization, where users need rapid feedback on large datasets [S2, unofficial]. The bit-packing optimization for binary data was described in early volume visualization literature [S2, unofficial]. The hierarchical pyramid structure for general data is a common extension of this principle, allowing efficient multi-resolution rendering [S2, unofficial]. Recent applications include iterative optimization schemes in neural rendering (e.g., Trick-GS), where progressive resolution refinement is used to balance training speed and quality [S1].

## Sources
[S1] Trick-GS: A Balanced Bag of Tricks for Efficient Gaussian Splatting, https://doi.org/10.1109/icassp49660.2025.10889395, fetched 2026-10-30
[S2] PDF Volume Visualization: Principles and Advances, https://courses.cs.duke.edu/spring03/cps296.8/papers/KaufmanVolumeVisualization.pdf, fetched 2026-10-28 (unofficial)
