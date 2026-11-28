# Hierarchical Viewing Cube Sampling

Also called: Adaptive cube placement, Quadtree cube sampling

## Overview
Hierarchical Viewing Cube Sampling is a method for computing scattering convolutions in volume rendering by adaptively placing viewing cubes along rays to balance accuracy and performance [S1, unofficial]. It solves the problem of efficiently evaluating high-dimensional scattering integrals by exploiting the spatial coherence of scattering convolutions: higher-order terms vary slowly and can be interpolated, while lower-order terms require precise sampling near the forward scattering peak [S1, unofficial]. It sets itself apart from uniform sampling by using a quadtree hierarchy to determine where new cubes are needed, and from pure interpolation by computing exact values at sparse vertices to guide the interpolation [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Inputs**: A set of low-density ray segments $S_{jkl}$ indexed by segment $j$ and pixel coordinates $(k, l)$, and a viewing cube geometry defined by rear five faces ($ABCD, AEFB, BFGC, CGHD, DHEA$) and a front face ($EFGH$) [S1, unofficial].
*   **Outputs**: Scattering convolutions $K_{ijkl}$ for each segment and pixel, probabilities $q_{ijkl}$, and the final radiance contribution $E_j$ for each segment [S1, unofficial].

**Data Structures**
*   **Convolutions $K_{ijkl}$**: Stored for each segment $j$ and pixel $(k,l)$ [S1, unofficial].
*   **Probabilities $q_{ijkl}$**: Computed for every pixel, even when $K$ is interpolated [S1, unofficial].
*   **Quadtree Vertices**: Used to hierarchically determine cube placement [S1, unofficial].
*   **Memory Buffer**: Stores only the right-hand column of the previous 9x9 block and one full image row (bottom row of the previous sweep) to enable horizontal sweep computation [S1, unofficial].

**Formulas**
The radiance contribution $E_j$ of segment $S_j$ to a viewing ray through a pixel is:
$$E_j = \sum_i q_{ij} K_{ij} \quad (9)$$
where $i$ indexes the scattering order/direction, $q_{ij}$ is the probability, and $K_{ij}$ is the scattering convolution [S1, unofficial].

**Parameters**
*   **Sampling Pattern Codes**:
    *   `'4'`: Place a new viewing cube at pixels where $x$ and $y$ are divisible by 4.
    *   `'2'`: Place a new viewing cube at pixels where $x$ and $y$ are divisible by 2.
    *   `'1'`: Place a new viewing cube at pixels where $x$ or $y$ is odd.
    *   `blank`: No new cube; indicates the viewing ray had no low-density segments [S1, unofficial].

**Steps**
1.  **Rear Face Sampling**: Sample the rear five faces ($ABCD, AEFB, BFGC, CGHD, DHEA$) initially along rays at every 8 output image pixels in $x$ and $y$ [S1, unofficial].
2.  **Front Face Sampling (Coarse)**: Sample the front face $EFGH$ initially every 4 pixels in $x$ and $y$, with doubled resolution to provide more accurate interpolated convolutions near the forward scattering peak [S1, unofficial].
3.  **Front Face Sampling (Fine)**: Sample only the center quarter $IJKL$ of the front face every 2 pixels in $x$ and $y$, disregarded when computing the whole front face convolution to ensure every direction is counted exactly once [S1, unofficial].
4.  **Hierarchical Cube Placement**: Compute convolutions hierarchically at the vertices of a quadtree to find values satisfactory for interpolation in sub-squares [S1, unofficial].
5.  **Interpolation & Placement**: Bilinearly interpolate $K_{ijkl}$ to positions indicated by `'-'` if distances along the ray are close enough; otherwise place a new viewing cube at the appropriate distance and calculate new $K_{ijkl}$ [S1, unofficial].
6.  **Accuracy Adaptation**: For higher-order (large $i$) convolutions, which are blurred and change slowly with pixel index $(k,l)$, rely on interpolation from sparser sampling and compute $q_{ijkl}$ at every pixel [S1, unofficial]. For smaller $i$, ensure the front face of the cube containing the viewing direction is accurate due to mainly forward scattering [S1, unofficial]. For even smaller $i$, only a central section of the front face is required, saving rendering time via octree clipping that eliminates large cells [S1, unofficial].
7.  **Sweep Computation**: Compute convolutions on 9x9 pixel blocks in horizontal sweeps. Save only the right-hand column of the previous block and one full image row to save memory. The left-hand column of the current block is known from the previous block, and the top row is known from the previous sweep [S1, unofficial].
8.  **Final Contribution**: Multiply $K_{ijkl}$ by $q_{ijkl}$ and sum over $i$ to get $E_j$ [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_hierarchical_viewing_cubes(image_width, image_height, segments, num_scattering_orders):
    K = {}  # K[j][k][l] stores convolution for segment j at pixel (k,l)
    q = {}  # q[j][k][l] stores probability for segment j at pixel (k,l)
    prev_block_right_col = []
    prev_sweep_bottom_row = []

    def compute_probability(seg, x, y):
        # Evaluate phase function/probability for segment at pixel [G]
        return 1.0

    def compute_cube_convolutions(seg, x, y, pattern):
        # Evaluate scattering integrals on the viewing cube faces [G]
        return [0.0] * num_scattering_orders

    def is_distance_close(seg, x, y):
        # Check if ray distances to existing cubes are within threshold [G]
        return True

    def bilinear_interpolate(K, seg, x, y):
        # Interpolate K from nearest computed pixels [G]
        return [0.0] * num_scattering_orders

    for block_y in range(0, image_height, 9):
        for block_x in range(0, image_width, 9):
            y_start, y_end = block_y, min(block_y + 9, image_height)
            x_start, x_end = block_x, min(block_x + 9, image_width)
            current_block_right_col = []

            for y in range(y_start, y_end):
                row_data = []
                for x in range(x_start, x_end):
                    if x % 4 == 0 and y % 4 == 0:
                        pattern = '4'
                    elif x % 2 == 0 and y % 2 == 0:
                        pattern = '2'
                    elif x % 2 != 0 or y % 2 != 0:
                        pattern = '1'
                    else:
                        pattern = '-'

                    for j in segments:
                        q[j][x][y] = compute_probability(j, x, y)

                        if pattern in ('4', '2', '1'):
                            K[j][x][y] = compute_cube_convolutions(j, x, y, pattern)
                            current_block_right_col.append(K[j][x][y])
                        else:
                            if is_distance_close(j, x, y):
                                K[j][x][y] = bilinear_interpolate(K, j, x, y)
                            else:
                                K[j][x][y] = compute_cube_convolutions(j, x, y, pattern)
                            current_block_right_col.append(K[j][x][y])
                    row_data.append(K[j][x][y])
                if y == y_end - 1:
                    prev_sweep_bottom_row = row_data
            prev_block_right_col = current_block_right_col

    for j in segments:
        for k in range(image_width):
            for l in range(image_height):
                E_j = sum(q[j][k][l][i] * K[j][k][l][i] for i in range(num_scattering_orders))
```

**Edge Cases and Pitfalls**
*   **Precision**: Interpolation is only valid if distances along the ray are "close enough." If not, a new cube must be placed to avoid artifacts [S1, unofficial].
*   **Boundaries**: The sweep computation requires careful handling of image boundaries to ensure the left column and top row are correctly initialized for the first block and sweep [S1, unofficial].
*   **Degenerate Input**: If a viewing ray has no low-density segments, the pattern code is blank, and no cube is placed [S1, unofficial].

## Operating characteristics
*   **Time Complexity**: Reduced compared to uniform sampling due to adaptive placement and interpolation [S1, unofficial]. Octree clipping eliminates large cells during traversal, saving rendering time [S1, unofficial].
*   **Memory Complexity**: Optimized by storing only the right-hand column of the previous 9x9 block and one full image row, significantly reducing memory footprint for large images [S1, unofficial].
*   **Strengths**: Efficiently handles forward scattering by focusing sampling on the front face and center [S1, unofficial]. Hierarchical placement reduces redundant calculations [S1, unofficial].
*   **Limitations**: Requires careful tuning of interpolation distance thresholds [S1, unofficial].
*   **When to Use**: Suitable for volume rendering with significant forward scattering, such as clouds or fog, where high-order scattering terms vary slowly [S1, unofficial].
*   **When Not to Use**: Not ideal for scenes with highly isotropic scattering or where rapid spatial changes in scattering properties occur, as interpolation may introduce errors [S1, unofficial].

## Background
Hierarchical Viewing Cube Sampling was introduced in the context of global illumination rendering for clouds and participating media [S1, unofficial]. It builds on the principles of adaptive sampling and quadtree subdivision to optimize the computation of scattering convolutions [S1, unofficial]. The method is detailed in the work on diffusion and multiple anisotropic scattering for global illumination in clouds [S1, unofficial].

## Sources
[S1] Diffusion and multiple anisotropic scattering for global illumination in clouds, https://digital.library.unt.edu/ark:/67531/metadc1410406/?q=multiple scattering rendering, fetched 2026-10-29 (unofficial)
