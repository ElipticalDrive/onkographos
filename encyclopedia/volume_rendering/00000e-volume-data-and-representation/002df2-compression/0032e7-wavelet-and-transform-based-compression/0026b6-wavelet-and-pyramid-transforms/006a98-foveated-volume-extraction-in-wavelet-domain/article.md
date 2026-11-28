# Foveated Volume Extraction in Wavelet Domain

Also called: Derive Foveated Volume, ROI Extraction, Wavelet Foveation

## Overview
Foveated Volume Extraction in Wavelet Domain is a method for extracting a subset of wavelet coefficients required to reconstruct a specific Region of Interest (ROI) at various resolution layers, thereby reducing the data size for transmission and rendering [S1, unofficial]. It solves the problem of efficiently transmitting and rendering high-resolution 3D volume data over networks by focusing computational and bandwidth resources on the user's area of interest [S1, unofficial]. Unlike full-volume reconstruction, this method leverages the hierarchical structure of wavelet transforms to identify and extract only the coefficients necessary to reconstruct the ROI across all resolution layers, achieving significant compression ratios (e.g., a factor of 2,000) [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D volume $V$ of size $N^3$ (where $N$ is a power of 2) transformed into the wavelet domain $W$, and a user-defined ROI specified by a center $(x, y, z)$ and a region radius $r$ (denoted as $ROI\{(x, y, z), r^3\}$) [S1, unofficial].
*   **Output:** A subset of coefficients $W_f$ from $W$ that allows for the reconstruction of the ROI at all resolution layers [S1, unofficial].

**Data Structures:**
The wavelet decomposition creates a hierarchy of layers. Layer 0 is the lowest resolution (average image). Layer $i$ is obtained by applying the inverse wavelet transform to Layer $i-1$ using the detail coefficients from Layer $i$. The highest resolution is Layer $m_l$ [S1, unofficial].

**Parameters:**
*   $N$: Side length of the cubic volume (power of 2).
*   $m$: Side length of the average image (power of 2).
*   $m_l$: Number of layers, calculated as $m_l = \frac{\log_2 N}{m}$ [S1, unofficial].
*   $(x, y, z)$: Center coordinates of the fovea/ROI in the original volume [S1, unofficial].
*   $r$: Radius of the ROI [S1, unofficial].
*   $s$: ROI size parameter, typically related to $r$ (e.g., $s=2r+1$ or similar depending on implementation specifics, though the source uses $s$ in boundary formulas) [S1, unofficial].

**Steps:**

1.  **Define ROI:** Specify the center $(x, y, z)$ and radius $r$ of the region of interest [S1, unofficial].
2.  **Calculate Layers:** Determine the total number of layers $m_l$ using the formula:
    $$ m_l = \frac{\log_2 N}{m} $$
    where $N$ is the volume size and $m$ is the size of the lowest resolution layer side [S1, unofficial].
3.  **Iterate Through Layers:** For each layer $i$ from $0$ to $m_l$:
    a.  **Calculate ROI Center at Layer $i$:** The center of the ROI in the coordinate system of layer $i$ is calculated by downsampling the original center:
        $$ C_i = \left( \text{round}\left(\frac{x}{2^{(m_l-i)}}\right), \text{round}\left(\frac{y}{2^{(m_l-i)}}\right), \text{round}\left(\frac{z}{2^{(m_l-i)}}\right) \right) $$
        [S1, unofficial]
    b.  **Calculate Boundary Coordinates:** Determine the spatial boundaries of the ROI at layer $i$. For each axis $t \in \{x, y, z\}$:
        $$ \text{begin}_i.t = \text{round}\left(\frac{x}{2^{(m_l-i)}} - \frac{s}{2}\right) $$
        $$ \text{end}_i.t = \text{round}\left(\frac{x}{2^{(m_l-i)}} + \frac{s}{2}\right) - 1 $$
        [S1, unofficial]
    c.  **Apply Boundary Constraints:** Ensure the boundaries are within valid limits for the layer. The coordinates must satisfy:
        $$ \text{begin}_i.t, \text{end}_i.t \in [0, \text{minlen} \cdot 2^i - 1] $$
        where $\text{minlen}$ is a minimum length parameter (often 1 or related to block size) [S1, unofficial].
    d.  **Identify Coefficients:** Identify all coefficients in layer $i$ that fall within the calculated boundaries $[\text{begin}_i.t, \text{end}_i.t]$. These coefficients are necessary to reconstruct the ROI at layer $i$ [S1, unofficial].
4.  **Extract Coefficients:** Collect all identified coefficients from all layers $0$ to $m_l$ into the foveated volume set $W_f$ [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def extract_foveated_volume(volume_size_N, avg_layer_size_m, roi_center_xyz, roi_radius_r):
    """
    Extracts wavelet coefficients for a foveated volume.
    
    Args:
        volume_size_N (int): Side length of the cubic volume (power of 2).
        avg_layer_size_m (int): Side length of the lowest resolution layer (power of 2).
        roi_center_xyz (tuple): (x, y, z) center of the ROI in original volume coordinates.
        roi_radius_r (int): Radius of the ROI.
        
    Returns:
        list: A list of coefficients (or their indices) required for the foveated volume.
    """
    # Calculate number of layers
    # m_l = log2(N) / m
    # Note: In practice, ensure N is a power of 2 and m is a power of 2
    import math
    m_l = int(math.log2(volume_size_N) / avg_layer_size_m)
    
    # ROI size parameter s. 
    # The source uses 's' in boundary formulas. 
    # Assuming s corresponds to the diameter of the ROI, s = 2 * r + 1 or similar.
    # The source example uses ROI size r^3, and boundary formulas use s/2.
    # If ROI is r^3, the side length is r. So s should be r.
    # Let's assume s = roi_radius_r based on "size of ROI_i is r^3" and boundary being center +/- s/2.
    s = roi_radius_r
    
    foveated_coefficients = []
    
    # Iterate through each layer
    for i in range(m_l + 1):
        # 1. Calculate ROI center at layer i
        # Center is downsampled by 2^(m_l - i)
        scale = 2 ** (m_l - i)
        cx = round(roi_center_xyz[0] / scale)
        cy = round(roi_center_xyz[1] / scale)
        cz = round(roi_center_xyz[2] / scale)
        
        # 2. Calculate boundary coordinates for each axis
        # begin = round(center - s/2)
        # end = round(center + s/2) - 1
        begin_x = round(cx - s / 2)
        end_x = round(cx + s / 2) - 1
        
        begin_y = round(cy - s / 2)
        end_y = round(cy + s / 2) - 1
        
        begin_z = round(cz - s / 2)
        end_z = round(cz + s / 2) - 1
        
        # 3. Apply boundary constraints
        # The source states: begin_i.t, end_i.t in [0, minlen * 2^i - 1]
        # minlen is not explicitly defined in the notes, but typically relates to block size or minimum valid dimension.
        # For a standard grid, minlen is often 1.
        minlen = 1 
        max_coord = minlen * (2 ** i) - 1
        
        # Clamp boundaries to valid range [0, max_coord]
        begin_x = max(0, min(begin_x, max_coord))
        end_x = max(0, min(end_x, max_coord))
        begin_y = max(0, min(begin_y, max_coord))
        end_y = max(0, min(end_y, max_coord))
        begin_z = max(0, min(begin_z, max_coord))
        end_z = max(0, min(end_z, max_coord))
        
        # 4. Identify coefficients
        # In a wavelet transform, layer i contains detail coefficients.
        # To reconstruct ROI_i, we need the detail coefficients at layer i 
        # that cover the spatial region defined by the boundaries.
        # The source implies extracting coefficients within these boundaries.
        
        # Note: The exact mapping of coefficients to spatial coordinates depends on the specific wavelet implementation.
        # Generally, for a separable 3D wavelet, layer i coefficients correspond to specific subbands.
        # We assume a function `get_coefficients_at_layer(layer_index, x_range, y_range, z_range)` exists.
        # Since we must write self-contained code, we describe the selection logic.
        
        # For implementation, one would iterate through the coefficient structure of layer i
        # and select those whose spatial support overlaps with [begin, end] in all dimensions.
        
        # Placeholder for actual coefficient extraction logic based on wavelet structure
        # coefficients_i = extract_coefficients_from_layer_i(i, (begin_x, end_x), (begin_y, end_y), (begin_z, end_z))
        # foveated_coefficients.extend(coefficients_i)
        
        # Since the source does not provide the internal structure of the wavelet coefficient array,
        # we note that the coefficients are those located within the calculated spatial bounds at each layer.
        pass 

    return foveated_coefficients
```

**Edge Cases and Pitfalls:**
*   **Boundary Conditions:** The boundary coordinates must be clamped to the valid range of the layer $[0, \text{minlen} \cdot 2^i - 1]$ to avoid out-of-bounds access [S1, unofficial].
*   **Redundancy in Transmission:** While extraction focuses on coefficients, transmission strategies (see Subtopics) may send redundant data if atomic blocks are used [S1, unofficial].
*   **Precision:** Rounding operations in center and boundary calculations can introduce small errors, but are necessary for integer coordinate systems [S1, unofficial].

## Subtopics

### Region-Based Progressive Transmission
A transmission strategy where wavelet foveated volume data is sent progressively from the innermost layer (highest detail/ROI) to outer layers [S1, unofficial]. The priority of a coefficient is inversely proportional to its distance to the center fovea, with the center having the highest priority [S1, unofficial]. It starts by transmitting and rendering the ROI at the highest resolution, then iteratively expands the region layer by layer towards the periphery [S1, unofficial]. Coefficients in each layer are wrapped as atomic packages and sent sequentially from inner to outer layers [S1, unofficial].

### Coarse-to-Fine Progressive Transmission
A transmission strategy that sends data blocks required to construct the ROI iteratively from coarse (low resolution) to fine (high resolution), including parent blocks in the octree [S1, unofficial]. It first transmits and renders the volume at the lowest resolution, then continuously transmits detail coefficients to progressively refine the image based on distance to the ROI [S1, unofficial]. The algorithm initializes a queue, computes the boundary of the innermost ROI, adds data blocks and their parent blocks to the queue, and iterates outward [S1, unofficial]. A pitfall is redundant data transmission because atomic block units may send detail coordinates needed for later steps in earlier steps [S1, unofficial].

## Operating characteristics
*   **Compression Ratio:** Significant reduction in data size. For a $1024^3$ volume with a $48^3$ fovea, the foveated volume has approximately $24^3 \times 36 = 497,664$ coefficients, a reduction factor of 2,000 compared to the original size [S1, unofficial].
*   **Time Complexity:** The extraction process involves iterating through layers and calculating boundaries, which is linear with respect to the number of layers and the size of the ROI. The progressive transmission adds overhead for queue management and block packaging.
*   **Memory Complexity:** The method stores only the extracted coefficients $W_f$, which is significantly smaller than the full wavelet coefficient set [S1, unofficial].
*   **Strengths:** Efficient bandwidth usage for remote visualization; allows high-resolution rendering of specific areas of interest; supports progressive refinement.
*   **Limitations:** Requires user interaction to define ROI; transmission strategies may introduce redundancy (especially coarse-to-fine); reconstruction is limited to the foveated region.

## Background
The method was proposed by Chen C. and Huang in 2006 as part of a system for remote volume rendering over networks [S1, unofficial]. It addresses the challenges of data transmission and fast volume rendering for large datasets from CT and MRI scanners [S1, unofficial]. The approach builds on the principles of wavelet transform-based compression and progressive transmission [S1, unofficial].

## Sources
[S1] , https://www.scitepress.org/PublishedPapers/2006/13574/pdf/index.html, fetched 2026-10-27 (unofficial)
