# Total Euler Characteristic (TEC)

Also called: Total Euler Characteristic as a Noise Measure [S1, unofficial]

## Overview
Total Euler Characteristic (TEC) is a noise level estimation metric for direct volume rendering transfer function design, defined as the L1-norm of the evolution curve of the Euler Characteristic across all possible isovalues [S1, unofficial]. It solves the problem of distinguishing true topological structures (islands and holes) from noise-induced speckles in rendered images, offering a more accurate noise assessment than gradient-based measures like Total Variation (TV) or Total Second Derivatives (TSD) [S1, unofficial]. Unlike its siblings, TEC approximates zero for vanishing noise even in scenes with significant true gradients and curvatures, making it superior for automated transfer function optimization [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** An integer-valued image (2D) or volume (3D), typically 8-bit or 16-bit [S1, unofficial].
*   **Output:** A scalar value $TEC$ representing the total topological complexity across all isovalues [S1, unofficial].

**Data Structures:**
*   **Histogram $H$:** An array indexed by intensity value $I$, with a bin width of one for each possible intensity value [S1, unofficial].
*   **Accumulated Histogram $H_{acc}$:** Stores the cumulative Euler characteristic for each isovalue [S1, unofficial].

**Method:**
The method computes the Euler Characteristic $\chi(T)$ for every possible isovalue $T$ simultaneously using a single raster scan and a histogram-based accumulation strategy [S1, unofficial]. The Euler Characteristic is defined as $\chi = \#vertices - \#edges + \#faces$ (in 2D) or extended to include cells in 3D, based on the Euler-Poincaré formula [S1, unofficial].

1.  **Initialize Histogram:** Create histogram $H$ with bins for all possible intensity values $I \in [0, I_{max}]$, initialized to zero [S1, unofficial].
2.  **Raster Scan:** Iterate over the image/volume using a sliding window [S1, unofficial].
    *   For 2D: Use a $2 \times 2$ neighborhood of pixels [S1, unofficial].
    *   For 3D: Use a $2 \times 2 \times 2$ neighborhood of voxels [S1, unofficial].
3.  **Update Histogram (Local Euler Contributions):** For each neighborhood, update the histogram bins corresponding to the topological elements becoming part of the super level set as the isovalue $T$ decreases from $I_{max}$ to $0$:
    *   **Vertices:** For each vertex (pixel/voxel) in the neighborhood with intensity $I_v$, increment $H[I_v]$ by 1. This represents the vertex entering the super level set when the isovalue reaches the intensity $I_v$ [S1, unofficial].
    *   **Edges (2D) / Faces (3D):** For each edge (pair of pixels) or face (quad of voxels) in the neighborhood, let $I_{min}$ be the minimum intensity of its constituent vertices. Decrement $H[I_{min}]$ by 1. This represents the edge/face entering the super level set when the isovalue becomes lower than this minimum [S1, unofficial].
    *   **Faces (2D) / Cells (3D):** For each face (quad of pixels) or cell (hexahedron of voxels) in the neighborhood, let $I_{min}$ be the minimum intensity of its constituent vertices. Increment $H[I_{min}]$ by 1. This represents the face/cell entering the super level set when the isovalue becomes lower than this minimum [S1, unofficial].
    *   In 3D, the algorithm is extended by decrementing the histogram for 8-voxel-cubes in a $2 \times 2 \times 2$ neighborhood to account for the next Betti number [S1, unofficial].
4.  **Accumulate:** After the raster scan, compute the accumulated histogram $H_{acc}$ by summing $H$ from the highest bin downwards [S1, unofficial].
    $$ H_{acc}[T] = \sum_{i=T}^{I_{max}} H[i] $$
    $H_{acc}[T]$ now contains the Euler Characteristic $\chi(T)$ of the super level set at isovalue $T$ [S1, unofficial].
5.  **Compute TEC:** Sum the absolute values of the accumulated histogram bins [S1, unofficial].
    $$ TEC = \sum_{T} |\chi(T)| $$
    This is the L1-norm of the Euler Characteristic evolution curve [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_tec(data):
    """
    Computes the Total Euler Characteristic for an integer-valued image or volume.
    data: 2D list/tuple (H, W) or 3D list/tuple (D, H, W) with integer values.
    Returns: float, the TEC value.
    """
    # Determine dimensions and extract data
    if len(data) == 0 or len(data[0]) == 0:
        return 0.0
        
    if isinstance(data[0][0], list):
        is_3d = True
        D, H_dim, W = len(data), len(data[0]), len(data[0][0])
    else:
        is_3d = False
        H_dim, W = len(data), len(data[0])

    # Find intensity range
    min_val = min(min(row) for row in data)
    max_val = max(max(row) for row in data)
    hist_size = max_val - min_val + 1
    
    # Initialize histogram
    H = [0] * hist_size
    
    # Helper to map intensity to histogram index
    def get_idx(val):
        return val - min_val

    # 2D Raster Scan
    if not is_3d:
        for h in range(H_dim - 1):
            for w in range(W - 1):
                # Extract 2x2 block
                v00 = data[h][w]
                v01 = data[h][w+1]
                v10 = data[h+1][w]
                v11 = data[h+1][w+1]
                
                # Vertices (4)
                for val in (v00, v01, v10, v11):
                    H[get_idx(val)] += 1
                
                # Edges (4)
                H[get_idx(min(v00, v01))] -= 1
                H[get_idx(min(v10, v11))] -= 1
                H[get_idx(min(v00, v10))] -= 1
                H[get_idx(min(v01, v11))] -= 1
                
                # Face (1)
                H[get_idx(min(v00, v01, v10, v11))] += 1

    # 3D Raster Scan
    else:
        for d in range(D - 1):
            for h in range(H_dim - 1):
                for w in range(W - 1):
                    # Extract 2x2x2 block values
                    vals = []
                    for z in range(2):
                        for y in range(2):
                            for x in range(2):
                                vals.append(data[d+z][h+y][w+x])
                    
                    # Vertices (8)
                    for val in vals:
                        H[get_idx(val)] += 1
                    
                    # Edges (12)
                    # Along x-axis (4 edges)
                    for z in range(2):
                        for y in range(2):
                            H[get_idx(min(data[d+z][h+y][w], data[d+z][h+y][w+1]))] -= 1
                    # Along y-axis (4 edges)
                    for z in range(2):
                        for x in range(2):
                            H[get_idx(min(data[d+z][h][w+x], data[d+z][h+1][w+x]))] -= 1
                    # Along z-axis (4 edges)
                    for y in range(2):
                        for x in range(2):
                            H[get_idx(min(data[d][h+y][w+x], data[d+1][h+y][w+x]))] -= 1

                    # Faces (6)
                    # XY planes (z=0, z=1)
                    for z in range(2):
                        H[get_idx(min(data[d+z][h][w], data[d+z][h][w+1], data[d+z][h+1][w], data[d+z][h+1][w+1]))] += 1
                    # XZ planes (y=0, y=1)
                    for y in range(2):
                        H[get_idx(min(data[d][h+y][w], data[d][h+y][w+1], data[d+1][h+y][w], data[d+1][h+y][w+1]))] += 1
                    # YZ planes (x=0, x=1)
                    for x in range(2):
                        H[get_idx(min(data[d][h][w+x], data[d][h+1][w+x], data[d+1][h][w+x], data[d+1][h+1][w+x]))] += 1

                    # Cell (1)
                    H[get_idx(min(vals))] -= 1

    # Accumulate histogram from highest bin downwards
    current_sum = 0
    H_acc = [0] * hist_size
    for i in range(hist_size - 1, -1, -1):
        current_sum += H[i]
        H_acc[i] = current_sum

    # Compute TEC: Sum of absolute values of accumulated histogram
    tec = sum(abs(val) for val in H_acc)
    
    return tec
```

**Parameters:**
*   **Data Type:** Integer-valued images (e.g., 8-bit or 16-bit) [S1, unofficial].
*   **Bin Width:** 1 for integer data [S1, unofficial].
*   **Dimensionality:** Can be computed in 2D (image-centric) or 3D (data-centric) [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Numerical Inaccuracies:** At thin crests of phantoms (e.g., Marschner-Lobb), numerical inaccuracies can produce speckles, causing TEC to not respond with zero even in the absence of added noise [S1, unofficial].
*   **Floating Point Data:** Extending the algorithm to floating-point data is straightforward, but the approximation error must be analyzed with respect to the histogram bin width [S1, unofficial].
*   **View Dependency:** Image-centric (2D) TEC is dependent on the view angle, whereas data-centric (3D) TEC is not [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Comparable to Total Variation (TV) [S1, unofficial]. In 2D, it samples four intensity points per pixel; in 3D, eight points per voxel [S1, unofficial]. This is slightly higher than TV (3 samples) and TSD (5 samples) but remains linear with respect to the number of pixels/voxels [S1, unofficial].
*   **Memory Complexity:** Requires a histogram of size proportional to the intensity range ($I_{max} - I_{min}$) [S1, unofficial]. This is independent of the image/volume size [S1, unofficial].
*   **Strengths:** Superior to TV and TSD for scenes with many true gradients and curvatures because it better approximates zero for vanishing noise [S1, unofficial]. It is more reflective of true noise levels and human perception of noise [S1, unofficial].
*   **Limitations:** Not perfectly zero for noise-free data due to numerical inaccuracies at thin crests [S1, unofficial]. Requires integer data for exact computation; floating-point extension requires bin width analysis [S1, unofficial].
*   **When to Use:** Automated evaluation of direct volume renderings, controlling transfer function parameters, or smoothing operations, especially when true gradients are present [S1, unofficial].
*   **When Not to Use:** When view-independence is required (image-centric TEC has view angle dependency) or when data is floating-point and bin-width approximation errors are unacceptable without further analysis [S1, unofficial].

## Background
The Total Euler Characteristic method was presented by R. Wiemker in 2013, copyrighted by The Eurographics Association [S1, unofficial]. It builds on the Euler-Poincaré formula and the additivity of the Euler Characteristic, allowing for efficient computation via sliding windows [S1, unofficial]. The method is used for automated evaluation of direct volume renderings and may be added to the Contour Spectrum [S1, unofficial]. It addresses limitations of Total Variation (TV) and Total Second Derivatives (TSD), which over-estimate noise by interpreting true object gradients as noise [S1, unofficial].

## Sources
[S1] Total Euler Characteristic as a Noise Measure to aid Transfer Function Design, https://doi.org/10.2312/pe.eurovisshort.eurovisshort2013.097-101, fetched 2026-10-29 (unofficial)
