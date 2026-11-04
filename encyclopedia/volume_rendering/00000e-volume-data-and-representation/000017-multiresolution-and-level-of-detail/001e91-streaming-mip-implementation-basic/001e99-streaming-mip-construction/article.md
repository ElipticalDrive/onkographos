# Streaming MIP Construction

Also called: Error-sorted Streaming MIP Construction, L1-error sorted Streaming MIP, MMIP, Multiresolution MIP, Streaming MIP

## Overview
Streaming MIP is a multiresolution volume rendering technique that constructs a progressive coefficient stream from a morphological pyramid to allow interactive frame rates on large datasets that do not fit in GPU memory [S1, unofficial]. It solves the problem of discrete quality jumps in level-by-level rendering by resorting detail coefficients from all pyramid levels simultaneously based on decreasing magnitude of an error measure, providing near-continuous error control [S1, unofficial]. It sets itself apart from basic texture-based MIP by storing only necessary voxels and from level-by-level MMIP by offering finer granularity in approximation error control [S1, unofficial].

## How it works
**Inputs:** A volumetric dataset $V$ of dimensions $W \times H \times D$.
**Outputs:** A MIP image $M$ and a performance/quality trade-off determined by the number of projected coefficients.

**Data Structures:**
1.  **Morphological Pyramid:** Constructed from $V$. The paper uses a three-level pyramid consisting of one approximation level $A_0$ and two detail levels $D_1, D_2$ [S1, unofficial].
    *   $A_0$: Approximation of the original data.
    *   $D_k$: Detail coefficients at level $k$.
2.  **Coefficient Stream:** A sorted list of detail coefficients $\{d_j\}$ from all levels $D_1, D_2$.
3.  **Auxiliary Coefficients:** An intermediate set used for sorting, discarded after the stream is constructed.
4.  **Render Target:** A framebuffer or image buffer initialized to the minimum intensity value (or background) for MIP accumulation.

**Parameters:**
*   **Pyramid Levels:** 3 levels (2 detail, 1 approximation) are used [S1, unofficial].
*   **Error Measure:** L1 error. The relative L1-approximation error is defined as:
    $$ \varepsilon^{(j)} = \frac{\| \mathbf{M}^{(0)} - \mathbf{M}^{(j)} \|_1}{\| \mathbf{M}^{(0)} \|_1} $$
    where $\mathbf{M}^{(0)}$ is the original MIP image and $\mathbf{M}^{(j)}$ is the approximation after projecting $j$ coefficients [S1, unofficial].
*   **Projection Type:** Parallel projection. Non-axis-aligned projections require post-processing [S1, unofficial].

**Steps:**

1.  **Construct Morphological Pyramid:**
    *   Apply morphological dilation and erosion operations to generate the pyramid levels.
    *   Compute detail coefficients $D_k$ by subtracting the upsampled approximation of the next level from the current level (or via the specific morphological difference definition used in the source).
    *   *Note:* The exact morphological pyramid construction algorithm is referenced to [GH00] and [Roe05] but not detailed in the provided text [S1, unofficial].

2.  **Construct Coefficient Stream:**
    *   Join all detail coefficients from all levels into a single set.
    *   Create an **auxiliary set** of detail coefficients.
    *   Sort the auxiliary set by decreasing magnitude (absolute value) of the coefficients.
    *   **Resort** the original detail coefficients $\{d_j\}$ by assigning them the same order as the sorted auxiliary coefficients.
    *   Discard the auxiliary coefficients.
    *   *Rationale:* Directly sorting detail coefficients by magnitude is suboptimal because coefficients are not small in regions where the structuring element fits approximately but not exactly. The auxiliary sorting ensures that each successive coefficient maximally reduces the L1-error between the partial reconstruction and the original data [S1, unofficial].

3.  **Rendering (Point-based Projection):**
    *   Initialize the render target.
    *   Iterate through the sorted coefficient stream $\{d_j\}$.
    *   For each coefficient $d_j$ at position $(x, y, z)$ with value $val$:
        *   Project the voxel at $(x, y, z)$ onto the image plane.
        *   Update the pixel value at the projected location: $pixel = \max(pixel, val)$.
        *   *Implicit Empty Space Skipping:* Only non-empty voxels (those with non-zero detail coefficients or significant approximation values) are rendered [S1, unofficial].
    *   Stop when a desired accuracy is reached or a maximum number of coefficients is projected.

4.  **Post-processing (Optional):**
    *   For non-axis-aligned parallel projections, the voxel-based method may yield pixel-sized holes.
    *   Apply a morphological closing operation using the structuring element $\tilde{A}$ used for synthesis to fill these holes [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def streaming_mip_construction(volume, pyramid_levels=3):
    """
    Constructs the coefficient stream for Streaming MIP.
    volume: 3D array of voxel intensities.
    pyramid_levels: Number of levels in the morphological pyramid.
    Returns: Sorted list of detail coefficients (d_j, x, y, z).
    """
    # 1. Construct Morphological Pyramid
    # Note: Exact morphological pyramid algorithm is not provided in source.
    # Assumes a function morphological_pyramid(volume, levels) exists.
    pyramid = morphological_pyramid(volume, pyramid_levels)
    
    # Extract detail coefficients from all levels except the approximation level
    detail_coefficients = []
    for level in range(1, pyramid_levels):
        coeffs = pyramid[level] # Assuming pyramid[level] contains detail coefficients
        for z in range(coeffs.shape[0]):
            for y in range(coeffs.shape[1]):
                for x in range(coeffs.shape[2]):
                    val = coeffs[z, y, x]
                    if val != 0: # Implicit empty space skipping
                        detail_coefficients.append({'val': val, 'x': x, 'y': y, 'z': z})
    
    # 2. Construct Coefficient Stream
    # Create auxiliary coefficients for sorting
    # The source states: "define an auxiliary set of detail coefficients which is sorted with respect to decreasing magnitude"
    # It does not specify if the auxiliary set is different from the original set other than in purpose.
    # We assume the auxiliary set is a copy of the detail coefficients.
    auxiliary_coefficients = list(detail_coefficients)
    
    # Sort auxiliary coefficients by decreasing magnitude (absolute value)
    auxiliary_coefficients.sort(key=lambda c: abs(c['val']), reverse=True)
    
    # Resort original detail coefficients to match the order of auxiliary coefficients
    # Since the values are the same, we just sort the original list by magnitude
    # The source implies a mapping: "original detail coefficients {dj} are resorted by giving them 
    # the same order as the resorted auxiliary coefficients"
    # This effectively means sorting the detail coefficients by magnitude.
    sorted_coefficients = sorted(detail_coefficients, key=lambda c: abs(c['val']), reverse=True)
    
    return sorted_coefficients

def streaming_mip_render(sorted_coefficients, image_width, image_height, projection_matrix):
    """
    Renders the MIP image by projecting sorted coefficients.
    sorted_coefficients: List of coefficients sorted by L1-error reduction.
    image_width, image_height: Dimensions of the output image.
    projection_matrix: Matrix to project 3D coordinates to 2D.
    Returns: MIP image (2D array).
    """
    # Initialize image with minimum value (e.g., 0 or background)
    image = np.zeros((image_height, image_width))
    
    for coeff in sorted_coefficients:
        x, y, z = coeff['x'], coeff['y'], coeff['z']
        val = coeff['val']
        
        # Project 3D coordinate to 2D
        # Note: Exact projection formula depends on projection_matrix
        # Assuming a standard parallel projection for simplicity
        u, v = project_3d_to_2d(x, y, z, projection_matrix, image_width, image_height)
        
        # Check if projection is within bounds
        if 0 <= u < image_width and 0 <= v < image_height:
            # Update pixel with maximum intensity
            if val > image[v, u]:
                image[v, u] = val
                
    return image

def post_process_morphological_closing(image, structuring_element):
    """
    Fills pixel-sized holes in non-axis-aligned projections.
    image: MIP image.
    structuring_element: Structuring element A used for synthesis.
    Returns: Closed image.
    """
    # Apply morphological closing (dilation followed by erosion)
    # Note: Exact implementation depends on the library, but the operation is standard.
    dilated = morphological_dilation(image, structuring_element)
    closed = morphological_erosion(dilated, structuring_element)
    return closed
```

**Edge Cases and Pitfalls:**
*   **Non-axis-aligned projections:** Yield pixel-sized holes. Requires post-processing morphological closing [S1, unofficial].
*   **Pyramid Levels:** Using more than three levels decreases performance due to additional render, upsample, and dilation steps, and higher levels have little non-zero coefficients [S1, unofficial].
*   **Precision:** The method trades image quality for performance. Median error of 7 grey levels is acceptable for preview mode [S1, unofficial].

## Subtopics

### Streaming MIP Implementation (Basic)
The basic implementation of the streaming MIP algorithm, focusing on the core rendering loop and data structures. [Streaming MIP Implementation (Basic)](1803-streaming-mip-implementation-basic.md)

### Morphological Pyramid Construction
The method for constructing the morphological pyramid from the volume data. [Morphological Pyramid Construction](52-morphological-pyramid-construction.md)

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of retained detail coefficients. Frame rate approximately doubles each time the number of detail coefficients is halved [S1, unofficial].
*   **Memory Complexity:** Cache-efficient, addressing GPU memory in a linear way. Only necessary voxels are stored [S1, unofficial].
*   **Performance:**
    *   **VisibleWoman (512x512x1734, 210M coeffs):**
        *   100% coeffs: 14 FPS, Median Error 0.
        *   1% coeffs: 51 FPS, Median Error 7 grey levels [S1, unofficial].
    *   **XMasTree (512x512x512, 105M coeffs):**
        *   100% coeffs: 80 FPS.
        *   1% coeffs: 100 FPS, Median Error 25 grey levels [S1, unofficial].
    *   **Comparison:**
        *   Streaming MIP (GPU) with L1 error 0.07 (median 4) achieves 18.4 FPS, compared to 1.1 FPS for Streaming MIP (GPU) with 0.0 error [S1, unofficial].
        *   Streaming MIP (GPU) is about six times faster than the optimized software implementation given the same dataset and error bound [S1, unofficial].
        *   Using two GeForce 7900GTX cards achieves a speed-up of almost a factor of two over a single card [S1, unofficial].
*   **Strengths:**
    *   Interactive frame rates on large datasets.
    *   Near-continuous error control.
    *   Cache-efficient memory access.
    *   Scalable to multiple GPUs [S1, unofficial].
*   **Limitations:**
    *   Requires post-processing for non-axis-aligned projections.
    *   Performance bound by vertex units on older GPU hardware (unified shaders improve this) [S1, unofficial].
    *   Does not remove view-independent hidden voxels (potential for 2x speedup if added) [S1, unofficial].

## Background
The method is based on morphological pyramids. Reference [GH00] discusses multiresolution signal decomposition schemes including linear and morphological pyramids [S1, unofficial]. The streaming MIP algorithm was introduced in [Roe05] to overcome the discrete quality jumps of level-by-level rendering [S1, unofficial]. The GPU implementation and comparison with existing CPU and GPU techniques were presented in [S1, unofficial].

## Sources
[S1] Multiresolution MIP Rendering of Large Volumetric Data Accelerated on Graphics Hardware, https://doi.org/10.2312/vissym/eurovis07/243-250, fetched 2026-10-26 (unofficial)
