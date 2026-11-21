# PRI Approximation Error Measurement

Also called: PRI Error Metric, PSNR calculation for PRI

## Overview
PRI Approximation Error Measurement quantifies the sampling error of a Pixel Ray Image (PRI) $P$ relative to its source unstructured grid $V$ by evaluating the grid's vertices within the PRI structure [S2, unofficial]. It verifies that the resampling process (converting an unstructured grid to a semi-regular PRI structure) preserves the original scalar field's accuracy [S2, unofficial]. The method builds on [Pixel Ray Image (PRI)](2467-pixel-ray-image-pri.md) and computes a weighted Mean Square Error (MSE) over the tetrahedral elements of the original grid [S1, S2, unofficial].

## How it works
The method measures the approximation error of a PRI $P$ by evaluating the scalar values of the original grid vertices within the PRI structure [S2, unofficial].

**Inputs and Outputs:**
*   **Input:** The original unstructured grid $V$ (composed of tetrahedra $t$) and the constructed PRI $P$ [S1, S2, unofficial].
*   **Output:** A Mean Square Error (MSE) value, which can be converted to Peak Signal-to-Noise Ratio (PSNR) [S1, S2, unofficial].

**Data Structures:**
*   **Grid $V$:** A collection of tetrahedra, where each tetrahedron $t$ has four vertices $v$ and a volume $|t|$ [S1, S2, unofficial].
*   **PRI $P$:** A data structure allowing trilinear interpolation of scalar values at arbitrary 3D coordinates [S1, unofficial].

**Steps:**
1. Iterate over every tetrahedron $t$ in the original grid $V$.
2. For each tetrahedron $t$, iterate over its four vertices $v$.
3. Retrieve the original scalar value $S(v)$ from the grid $V$ at vertex $v$.
4. Reconstruct the scalar value $S_P(v)$ by performing trilinear interpolation on the PRI $P$ at the coordinates of vertex $v$ [S1, unofficial].
5. Compute the residual error for each vertex: $|S(v) - S_P(v)|$.
6. Determine the maximal residual error for the tetrahedron $t$:
   $$ \varepsilon_P(t) = \max_{\forall v \in t} |S(v) - S_P(v)| $$
   where $\varepsilon_P(t)$ is the maximal residual error on tetrahedron $t$ [S1, S2, unofficial].
7. Compute the weighted Mean Square Error (MSE) using the volume of each tetrahedron as the weight:
   $$ MSE(V, P) = \frac{\sum_{\forall t \in V} \varepsilon_P(t)^2 |t|}{\sum_{\forall t \in V} |t|} $$
   where $|t|$ is the volume of tetrahedron $t$ [S1, S2, unofficial].
8. (Optional) Convert MSE to PSNR using standard logarithmic conversion [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_pri_error(grid_V, pri_P):
    """
    Computes the MSE approximation error of PRI P relative to unstructured grid V.
    """
    total_weighted_error = 0.0
    total_volume = 0.0

    for tetrahedron in grid_V.tetrahedra:
        volume = tetrahedron.volume()
        total_volume += volume

        max_residual = 0.0
        for vertex in tetrahedron.vertices:
            s_original = vertex.scalar_value
            s_reconstructed = pri_P.trilinear_interpolate(vertex.position)
            residual = abs(s_original - s_reconstructed)
            if residual > max_residual:
                max_residual = residual

        total_weighted_error += (max_residual ** 2) * volume

    if total_volume == 0:
        return 0.0
    mse = total_weighted_error / total_volume
    return mse
```

**Parameters:**
*   **Error Threshold:** In the context of the simplification algorithm that generates the PRI, an error threshold determines when knots are removed; an error threshold of 0.0 results in the removal of superfluous knots representing samples in constant or linearly changing regions [S1, unofficial].

**Edge Cases and Pitfalls:**
*   Not given in the sources.

## Subtopics

### Radius Error from Vertex
A quality metric measuring the error committed when approximating a sphere, calculated as the distance from the sphere center to the vertices of each triangle [S3, unofficial]. It is used to compare isosurface extraction algorithms [S3, unofficial].
*   **Definition:** Measures the deviation of triangle vertices from the ideal spherical radius [S3, unofficial].
*   **Formula:** $\text{Total Error} = \sum_{i=1}^n \frac{R_i - R_r}{R_r}$, where $n$ is the number of triangles multiplied by 3, $R_i$ is the radius of the sphere, and $R_r$ is the distance from the center $(c_x, c_y, c_z)$ to a vertex $(v_x, v_y, v_z)$ [S3, unofficial].
*   **Vertex Distance:** $r_i = \sqrt{(v_x - c_x)^2 + (v_y - c_y)^2 + (v_z - c_z)^2}$ [S3, unofficial].
*   **Usage:** Lower radius error indicates a better geometric approximation of the sphere [S3, unofficial].

## Operating characteristics
Not given in the sources.

## Background
The method originates from research on GPU-based unstructured grid volume rendering, specifically the Image-Based Modelling approach [S1, S2, unofficial]. It was introduced to evaluate the accuracy of the Pixel Ray Image (PRI) representation, which replaces irregular unstructured grids with a semi-regular structure suitable for GPU texture storage [S1, S2, unofficial]. The approach follows a method similar to [CCM*00] for measuring sampling error in resampled data [S1, S2, unofficial]. The metric allows researchers to quantify the trade-off between data reduction (knot removal) and accuracy in the PRI simplification process [S1, unofficial].

## Sources
[S1] An Image-Based Modelling Approach To GPU-based Unstructured Grid Volume Rendering, https://doi.org/10.2312/vg/vg06/031-038, fetched 2026-10-24 (unofficial)
[S2] PDF An Image-Based Modelling Approach To GPU-based Unstructured Grid Volume ..., https://www3.cs.stonybrook.edu/~mueller/papers/vg06_ibr.pdf, fetched 2026-10-27 (unofficial)
[S3] Two Methods for Iso-Surface Extraction from Volumetric Data and Their Comparison, https://arxiv.org/abs/2201.03446v1, fetched 2026-10-26 (unofficial)
