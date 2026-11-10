# Brushing and Linking with Hilbert Codes

Also called: Brushing and Linking for DVL and 3D Views, Hilbert code selection

## Overview
This method connects 1D visualization views (such as Dynamic Volume Lines) with 3D volume rendering by using Hilbert space-filling curve codes to define selection ranges. When a user selects a Region of Interest (ROI) in the 1D plot, the corresponding cells in the 3D view are highlighted. It differs from standard spatial selection by mapping the selection criterion to a 1D range of Hilbert codes rather than a 3D bounding box, allowing for compact representation of non-contiguous spatial regions [S1].

## How it works
**Inputs:**
- A set of ROIs defined in the 1D view, each specified by a start and end Hilbert code ($H_{start}, H_{end}$).
- A 3D volume dataset (structured-regular or AMR).
- A transfer function $T$ mapping scalar values to color and opacity.

**Outputs:**
- Highlighted cells in the 3D view corresponding to the selected ROIs.

**Data Structures:**
- **Hilbert Code:** A 64-bit integer representing the position of a cell in the Hilbert curve ordering [S1].
- **ROI Range:** A pair of integers $(H_{start}, H_{end})$ defining the selection interval [S1].

**Steps:**
1. **User Selection:** The user selects one or more ROIs in the 1D view. The system records the first and last Hilbert codes of each selected ROI [S1].
2. **Shader Initialization:** The 3D rendering shader receives the list of ROI ranges $(H_{start}, H_{end})$ as uniform variables [S1].
3. **Ray Casting:** For each pixel, a ray is cast through the volume. At each sample point $p$ along the ray:
    a. **Coordinate to Hilbert Conversion:** The 3D coordinates of the sample point $p$ are transformed into its corresponding Hilbert code $H(p)$ [S1].
    b. **Containment Test:** The shader checks if $H(p)$ falls within any of the active ROI ranges:
       $$ \exists k : H_{start}^{(k)} \le H(p) \le H_{end}^{(k)} $$
    c. **Value Lookup:** If contained, the scalar value at $p$ is retrieved and passed to the transfer function $T$ to determine color and opacity [S1].
4. **Compositing:** Standard front-to-back or back-to-front compositing is performed using the resulting color and opacity values [S1].

**Formulas:**
- **ROI Containment:** A sample at position $p$ is part of ROI $k$ if:
  $$ H_{start}^{(k)} \le H(p) \le H_{end}^{(k)} $$
  where $H(p)$ is the Hilbert code of the cell containing $p$ [S1].
- **Alpha Approximation (for AMR):** For AMR data, the alpha value for a ROI is approximated by:
  $$ V_\alpha = \max_{m \in M} \left( \max_{a \in [i, j]} A(m, a) - \min_{a \in [i, j]} A(m, a) \right) $$
  where $A$ is the transfer function lookup, $[i, j]$ is the Hilbert code range, and $M$ is the set of fields [S1].

**Parameters:**
- **Hilbert Order:** Determined by the resolution of the volume grid. Not given in the sources.
- **Bit Width:** Typically 64-bit integers for Hilbert codes [S1].

**Edge Cases and Pitfalls:**
- **Structured-Regular Volumes:** Cell bounds can be derived implicitly from the Hilbert code because cells are uniform in size [S1].
- **AMR Data:** The Hilbert code of the cell centroid is insufficient to determine if a sample falls inside an ROI because cell sizes vary. The shader must explicitly locate cells to know the ROI's exact size when checking containment [S1].
- **Precision:** Transforming 3D coordinates to Hilbert space requires precise mapping to avoid boundary errors [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode for 3D Shader Ray Casting Step
def ray_cast_pixel(ray_origin, ray_direction, roi_ranges, volume_data, transfer_function):
    color = (0.0, 0.0, 0.0, 0.0)
    alpha = 0.0
    
    # Sample along the ray
    for t in range(0, max_depth, step_size):
        p = ray_origin + t * ray_direction
        h_code = compute_hilbert_code(p) # [G] Helper to map 3D coord to Hilbert int
        
        is_selected = False
        for (h_start, h_end) in roi_ranges:
            if h_start <= h_code <= h_end:
                is_selected = True
                break
        
        if is_selected:
            scalar_value = volume_data.sample(p)
            rgba = transfer_function(scalar_value)
            
            # Front-to-back compositing
            color = color + rgba.color * rgba.alpha * (1.0 - alpha)
            alpha = alpha + rgba.alpha * (1.0 - alpha)
            
            if alpha > 0.99:
                break
                
    return color
```

## Subtopics

### Interactive Volume Lines (IVL)
The GPU-accelerated implementation of Dynamic Volume Lines that uses this brushing and linking technique. It allows for real-time interaction with large AMR data [S1].

### Hilbert Code Generation
The process of mapping 3D voxel coordinates to 1D Hilbert codes. This is a prerequisite for the selection method. Not given in the sources.

## Operating characteristics
- **Time Complexity:** The containment test is $O(1)$ per sample per ROI range. The overall complexity is dominated by the ray casting steps.
- **Memory Complexity:** The ROI ranges are stored as small lists of integers, making the memory footprint for selection data minimal [S1].
- **Performance:** The method is efficient for GPU implementation using CUDA. The compact representation of ROIs as Hilbert codes reduces memory bandwidth usage compared to storing lists of cell indices [S1].
- **Strengths:** Compact representation of complex, non-contiguous ROIs; efficient GPU implementation; seamless integration between 1D and 3D views [S1].
- **Limitations:** Requires pre-computation of Hilbert codes; AMR data requires explicit cell location for accurate containment testing, which may be more expensive than structured grids [S1].

## Background
This method was introduced in the context of visual analysis of large multi-field AMR data on GPUs using Interactive Volume Lines. It was presented at IEEE VIS 2023 [S1]. The technique builds on the concept of Dynamic Volume Lines (DVL) and extends it to support 3D view interaction through Hilbert space-filling curves [S1].

## Sources
[S1] Visual Analysis of Large Multi-Field AMR Data on GPUs Using Interactive Volume Lines, https://doi.org/10.1109/vis54172.2023.00020, fetched 2026-10-26
[S2] PDF Visual Analsyis of Large Multi-Field AMR Data on GPUs Using Interactive ..., https://www.cs.bilkent.edu.tr/~gudukbay/publications/papers/conf_papers/Zellmann_Et_Al_IEEE_VIS_2023.pdf, fetched 2026-10-27 (unofficial)
