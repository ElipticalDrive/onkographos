# High Order Interpolation (Cubic/B-Spline)
Also called: Cubic Convolution
## Overview
High order interpolation methods, such as cubic convolution or B-spline interpolation, provide superior image quality compared to trilinear interpolation, particularly when rendering at large magnification factors where trilinear kernels may produce visible diamond structures or crosses [S1, unofficial]. These methods are chosen when visual fidelity is prioritized over performance, accepting a significant increase in computational cost and memory bandwidth usage [S1, unofficial].
## How it works
Inputs: A 3D volume grid $V$ of scalar or vector values, and a query point $P = (x, y, z)$ within the volume bounds.
Outputs: The interpolated value $I(P)$.

Data Structures:
- Volume Grid $V$: A 3D array of voxels.
- Neighborhood: A $4 \times 4 \times 4$ block of voxels centered around the integer coordinates of $P$. This requires accessing 64 voxels [S1, unofficial].

Steps:
1. Determine the integer grid coordinates $(i, j, k)$ such that $i \le x < i+1$, $j \le y < j+1$, $k \le z < k+1$.
2. Identify the 64 neighboring voxels required for the cubic kernel. For a uniform cubic B-spline or cubic convolution, this typically involves the 4 voxels along each axis surrounding the query point, forming a $4^3$ neighborhood [S1, unofficial].
3. Compute the fractional offsets $dx = x - i$, $dy = y - j$, $dz = z - k$.
4. Evaluate the 1D cubic basis function $W(t)$ for each dimension. For cubic B-splines, the basis function is defined piecewise. For cubic convolution, a specific kernel (e.g., Keys' cubic) is used. The sources do not provide the exact mathematical formula for the basis function or the convolution weights [S1, unofficial].
5. Compute the interpolated value by convolving the 64 voxel values with the 3D tensor product of the 1D basis functions:
   $$ I(P) = \sum_{a=0}^{3} \sum_{b=0}^{3} \sum_{c=0}^{3} V(i+a, j+b, k+c) \cdot W(dx-a) \cdot W(dy-b) \cdot W(dz-c) $$
   where $W$ is the 1D cubic interpolation kernel [S1, unofficial].
6. Return $I(P)$.

Parameters:
- Kernel type: Cubic B-spline or Cubic Convolution. Specific coefficients are not provided in the sources [S1, unofficial].
- Neighborhood size: 64 voxels ($4 \times 4 \times 4$) [S1, unofficial].

Edge cases and pitfalls:
- Boundary conditions: The $4 \times 4 \times 4$ neighborhood may extend outside the volume bounds. Handling requires clamping, wrapping, or extrapolation, which is not specified in the sources [S1, unofficial].
- Precision: Cubic interpolation involves more floating-point operations than trilinear, potentially amplifying precision errors in low-precision hardware [S1, unofficial].
- Aliasing: Interpolation of scalar values is prone to aliasing if high-frequency details are missed by the classification step, regardless of the interpolation order [S1, unofficial].

## Subtopics
## Operating characteristics
Time Complexity: $O(1)$ per voxel, but with a significantly higher constant factor than trilinear interpolation due to the 64-voxel neighborhood and cubic kernel evaluations [S1, unofficial].
Memory Bandwidth: Requires loading 64 voxels per query, leading to significantly higher memory bandwidth consumption compared to the 8 voxels required for trilinear interpolation [S1, unofficial].
Strengths: Higher image quality, especially at large magnification factors, avoiding the artifacts associated with trilinear interpolation [S1, unofficial].
Limitations: Significantly higher computational cost and memory bandwidth requirements [S1, unofficial].
When to use: When visual quality is critical and computational resources are sufficient [S1, unofficial].
When not to use: When performance or memory bandwidth is constrained, or when lower magnification factors make the quality difference negligible [S1, unofficial].
## Background
High order interpolation methods like cubic convolution and B-spline interpolation are established techniques in volume visualization, offering a trade-off between quality and performance [S1, unofficial]. They are used in applications where the artifacts of trilinear interpolation (such as diamond structures at large magnifications) are unacceptable [S1, unofficial].
## Sources
[S1] Volume Visualization and Volume Rendering Techniques, https://www.cs.unh.edu/~cs880/visualization/Meissner-VolRenderingEGTutorial-rdb.pdf, fetched 2026-10-27 (unofficial)
