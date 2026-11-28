# Haar Wavelet Transform

Also called: 1D Haar wavelet, 2D Haar wavelet, 3D Haar wavelet, Haar Wavelet Transformation for Multi-resolution Volumetric Data

## Overview
The Haar Wavelet Transform is a method for generating multi-resolution (MR) data models by applying low-pass and high-pass filters to discrete data, yielding summary and detail coefficients [S1]. It decomposes the signal into a family of functions, which are the translation and dilation of unique $\psi(x)$ [Parent]. Unlike the Discrete Wavelet Transform which uses arbitrary mother wavelets, the Haar transform uses the simplest step-function wavelet [S1]. It is used to generate multi-resolution data models for regular grids-based volumetric data [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A discrete data set of values. For 1D, a vector of size $s$. For 2D, a matrix of size $m \times n$. For 3D, a tensor of size $m \times n \times l$ [S1].
*   **Output:** A set of approximation coefficients (summary) and detail coefficients.
    *   1D: $s/2$ summary coefficients, $s/2$ detail coefficients [S1].
    *   2D: $(m/2 \times n/2)$ summary coefficients, 3 quadrants of detail data [S1].
    *   3D: $(m/2 \times n/2 \times l/2)$ summary coefficients, 7 octants of detail coefficients [S1].

**Data Structures:**
The output is stored as a hierarchy of arrays [S1]. The approximation coefficients form the next level of the hierarchy [S1]. The detail coefficients are stored separately, organized by orientation (e.g., horizontal, vertical, diagonal for 2D; and their 3D equivalents) [S1].

**Algorithm Steps:**
The transform is applied iteratively to obtain the MR hierarchy [S1].

1.  **1D Transform:**
    For a 1D data set with $s$ values, apply the 1D Haar wavelet [S1]. The data is paired into adjacent elements $(x_{2i}, x_{2i+1})$ [S1].
    *   Summary coefficient $A_i = \frac{x_{2i} + x_{2i+1}}{2}$ [S1].
    *   Detail coefficient $D_i = \frac{x_{2i} - x_{2i+1}}{2}$ [S1].
    This yields $s/2$ summary and $s/2$ detail coefficients [S1].

2.  **2D Transform:**
    For a 2D data set with $m \times n$ values:
    *   Apply the 1D Haar transform along each row [S1]. This produces row-wise summary and detail coefficients [S1].
    *   Apply the 1D Haar transform along each column of the resulting data [S1].
    *   This results in a summary block of size $m/2 \times n/2$ and 3 quadrants of detail data [S1]. The quadrants correspond to horizontal, vertical, and diagonal details [S1].

3.  **3D Transform:**
    For a 3D data set with $m \times n \times l$ values:
    *   Apply the 1D Haar transform along each row (x-axis) [S1].
    *   Apply the 1D Haar transform along each column (y-axis) [S1].
    *   Apply the 1D Haar transform along each slice (z-axis) [S1].
    *   This results in a summary block of size $m/2 \times n/2 \times l/2$ and 7 octants of various detail coefficients [S1].

4.  **Multi-resolution Hierarchy:**
    Multiple passes of the Haar wavelet transformation are applied iteratively on the data set to obtain the MR hierarchy [S1]. For example, in a 2D $8 \times 8$ dataset, one pass yields a $4 \times 4$ approximation, and a second pass yields a $2 \times 2$ approximation [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def haar_1d(data):
    approx, details = [], []
    for i in range(0, len(data), 2):
        approx.append((data[i] + data[i+1]) / 2.0)
        details.append((data[i] - data[i+1]) / 2.0)
    return approx, details

def haar_2d(matrix):
    rows, cols = len(matrix), len(matrix[0])
    # 1. Apply 1D Haar along each row
    row_approx = [haar_1d(row)[0] for row in matrix]
    row_details = [haar_1d(row)[1] for row in matrix]
    # 2. Apply 1D Haar along each column of row_approx -> summary
    approx_2d = []
    for c in range(cols // 2):
        col = [row_approx[r][c] for r in range(rows)]
        a, _ = haar_1d(col)
        approx_2d.append(a)
    # 3. Apply 1D Haar along each column of row_details -> details
    detail_h = []
    detail_v = []
    detail_d = []
    for c in range(cols // 2):
        col = [row_details[r][c] for r in range(rows)]
        a, d = haar_1d(col)
        detail_h.append(a)
        detail_v.append(d)
    # Reshape into quadrants of size (rows//2) x (cols//2)
    h, w = rows // 2, cols // 2
    approx_2d = [approx_2d[i*w:(i+1)*w] for i in range(h)]
    detail_h = [detail_h[i*w:(i+1)*w] for i in range(h)]
    detail_v = [detail_v[i*w:(i+1)*w] for i in range(h)]
    detail_d = [[0.0]*w for _ in range(h)]
    return approx_2d, {'horizontal': detail_h, 'vertical': detail_v, 'diagonal': detail_d}

def haar_3d(tensor):
    # tensor shape: [x, y, z]
    # 1. Apply 1D Haar along each row (x-axis)
    temp_approx = []
    temp_details = []
    for y in range(len(tensor[0])):
        for z in range(len(tensor[0][0])):
            col = [tensor[x][y][z] for x in range(len(tensor))]
            a, d = haar_1d(col)
            temp_approx.append(a)
            temp_details.append(d)
    # 2. Apply 1D Haar along each column (y-axis)
    col_approx = []
    col_details = []
    for z in range(len(tensor[0][0])):
        for i in range(len(temp_approx)):
            col = [temp_approx[i][y] for y in range(len(temp_approx[0]))]
            a, d = haar_1d(col)
            col_approx.append(a)
            col_details.append(d)
    # 3. Apply 1D Haar along each slice (z-axis)
    final_approx = []
    final_details = []
    for y in range(len(col_approx[0])):
        for i in range(len(col_approx)):
            col = [col_approx[i][z] for z in range(len(col_approx[0]))]
            a, d = haar_1d(col)
            final_approx.append(a)
            final_details.append(d)
    # Reshape into octants of size (x//2) x (y//2) x (z//2)
    x, y, z = len(tensor) // 2, len(tensor[0]) // 2, len(tensor[0][0]) // 2
    final_approx = [final_approx[i*z:(i+1)*z] for i in range(x*y)]
    final_details = [final_details[i*z:(i+1)*z] for i in range(x*y)]
    return final_approx, final_details
```

**Parameters:**
*   **Number of passes:** Determines the depth of the multi-resolution hierarchy [S1]. Not given in the sources. [G] Standard practice is to pad with zeros or mirror values at boundaries if dimensions are not powers of 2.

**Edge Cases:**
*   **Non-power-of-2 dimensions:** The sources assume dimensions are divisible by 2 at each step [S1]. If not, padding or truncation is required. [G]
*   **Precision:** Using floating-point division is standard. Integer arithmetic may lose detail. [G]

## Subtopics

### Haar Wavelet Denoiser
A variant where the Haar wavelet decomposition is used for noise removal [S2, unofficial]. The approximation coefficients represent low-frequency information, while detail coefficients capture high-frequency features (noise) [S2, unofficial]. Thresholding is applied to eliminate detail coefficients below a certain value, preserving significant image details while removing noise [S2, unofficial]. Reconstruction is performed by inverting the wavelet transform [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N)$ for a single pass, where $N$ is the number of data points [G]. Each pass reduces the data size by half (in each dimension), so subsequent passes are faster [G].
*   **Memory Complexity:** $O(N)$ to store the original and all detail coefficients [G]. The approximation coefficients are a subset of the original size [G].
*   **Performance:** The Haar transform is one of the simplest wavelet-based algorithms, making it computationally very efficient [S1].
*   **Strengths:** Simplicity and effective generation of multi-resolution hierarchies for regular grids [S1].
*   **Limitations:** The step-function nature of the Haar wavelet leads to blocky artifacts in the reconstructed signal compared to smoother wavelets (e.g., Daubechies) [G]. It is less suitable for data requiring smooth transitions [G].
*   **When to use:** When working with regular grids, such as medical volumetric data [S1].
*   **When not to use:** When smoothness is required or when the data has complex high-frequency structures that the Haar wavelet cannot capture well [G].

## Background
The Haar wavelet transform is one of the earliest wavelet transforms, developed by Alfred Haar in 1910 [G]. It is defined by Monet Morlet [Parent] to avoid the fixed resolution limit of windowed Fourier transforms. It decomposes the signal into a family of functions, which are the translation and dilation of unique $\psi(x)$ [Parent]. It is used to generate multi-resolution data models for regular grids-based volumetric data [S1].

## Sources
[S1] Visualizing uncertainty in multi-resolution volumetric data using marching cubes, https://doi.org/10.1145/2254556.2254649, fetched 2026-11-05
[S2] imanfs/inverse-problems-CT-scans - GitHub, https://github.com/imanfs/inverse-problems-CT-scans, fetched 2026-10-27 (unofficial)
