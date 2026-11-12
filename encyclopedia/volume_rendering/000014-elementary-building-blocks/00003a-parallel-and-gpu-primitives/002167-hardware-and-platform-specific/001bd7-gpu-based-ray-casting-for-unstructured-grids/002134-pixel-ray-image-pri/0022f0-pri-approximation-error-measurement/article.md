# PRI Approximation Error Measurement

Also called: PRI Error Metric, PSNR calculation for PRI

## Overview
PRI Approximation Error Measurement quantifies the fidelity of a Pixel Ray Image (PRI) representation relative to its source unstructured grid by evaluating the sampling error at the grid's vertices. It solves the problem of verifying that the resampling process (converting an unstructured grid to a semi-regular PRI structure) preserves the original scalar field's accuracy. It is distinguished from general rendering error metrics by focusing specifically on the reconstruction error of the underlying data structure rather than the visual output of a ray-caster.

## How it works
The method measures the approximation error of a PRI $P$ (which is a resampling of an unstructured grid $V$) by evaluating the scalar values of the original grid vertices within the PRI structure.

**Inputs and Outputs:**
*   **Input:** The original unstructured grid $V$ (composed of tetrahedra $t$) and the constructed PRI $P$.
*   **Output:** A Mean Square Error (MSE) value, which can be converted to Peak Signal-to-Noise Ratio (PSNR).

**Data Structures:**
*   **Grid $V$:** A collection of tetrahedra. Each tetrahedron $t$ has four vertices $v$ and a volume $|t|$.
*   **PRI $P$:** A data structure allowing trilinear interpolation of scalar values at arbitrary 3D coordinates.

**Steps:**
1.  Iterate over every tetrahedron $t$ in the original grid $V$.
2.  For each tetrahedron $t$, iterate over its four vertices $v$.
3.  Retrieve the original scalar value $S(v)$ from the grid $V$ at vertex $v$.
4.  Reconstruct the scalar value $S_P(v)$ by performing trilinear interpolation on the PRI $P$ at the coordinates of vertex $v$.
5.  Compute the residual error for each vertex: $|S(v) - S_P(v)|$.
6.  Determine the maximal residual error for the tetrahedron $t$:
    $$ \varepsilon_P(t) = \max_{\forall v \in t} |S(v) - S_P(v)| $$
    where $\varepsilon_P(t)$ is the maximal residual error on tetrahedron $t$ [S1, S2, unofficial].
7.  Compute the weighted Mean Square Error (MSE) using the volume of each tetrahedron as the weight:
    $$ MSE(V, P) = \frac{\sum_{\forall t \in V} \varepsilon_P(t)^2 |t|}{\sum_{\forall t \in V} |t|} $$
    where $|t|$ is the volume of tetrahedron $t$ [S1, S2, unofficial].
8.  (Optional) Convert MSE to PSNR using standard logarithmic conversion.

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
        # Calculate volume of the tetrahedron
        volume = tetrahedron.volume()
        total_volume += volume

        # Find max residual error among the 4 vertices of this tetrahedron
        max_residual = 0.0
        for vertex in tetrahedron.vertices:
            # S(v): Original scalar value at vertex
            s_original = vertex.scalar_value
            
            # S_P(v): Reconstructed scalar value via trilinear interpolation in PRI
            s_reconstructed = pri_P.trilinear_interpolate(vertex.position)
            
            # Residual error
            residual = abs(s_original - s_reconstructed)
            if residual > max_residual:
                max_residual = residual

        # Square the max residual and weight by volume
        total_weighted_error += (max_residual ** 2) * volume

    # Compute MSE
    if total_volume == 0:
        return 0.0
    mse = total_weighted_error / total_volume
    return mse
```

**Parameters:**
*   **Error Threshold:** In the context of the simplification algorithm that generates the PRI, an error threshold determines when knots are removed. An error threshold of 0.0 results in the removal of superfluous knots (samples lying in constant or linearly changing regions) [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** The calculation relies on the accuracy of the trilinear interpolation in the PRI. If the PRI resolution is too low, $S_P(v)$ may deviate significantly from $S(v)$, inflating the error metric [S1, unofficial].
*   **Boundary Conditions:** Vertices on the boundary of the grid must be correctly mapped to the PRI sampling plane. Incorrect mapping leads to invalid interpolation results [G].

## Subtopics

### Radius Error from Vertex
A quality metric measuring the error committed when approximating a sphere, calculated as the distance from the sphere center to the vertices of each triangle. It is used to compare isosurface extraction algorithms.
*   **Definition:** Measures the deviation of triangle vertices from the ideal spherical radius.
*   **Formula:**
    $$ \text{Total Error} = \sum_{i=1}^n \frac{R_i - R_r}{R_r} $$
    where $n$ is the number of triangles multiplied by 3 (total vertices), $R_i$ is the radius of the sphere (ideal), and $R_r$ is the distance from the center $(c_x, c_y, c_z)$ to a vertex $(v_x, v_y, v_z)$ [S3, unofficial].
*   **Vertex Distance:**
    $$ r_i = \sqrt{(v_x - c_x)^2 + (v_y - c_y)^2 + (v_z - c_z)^2} $$
    where $r_i$ is the actual distance from the center to vertex $i$ [S3, unofficial].
*   **Usage:** Lower radius error indicates a better geometric approximation of the sphere. This metric is distinct from the scalar field MSE used in PRI error measurement and is typically applied to mesh quality assessment [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N \cdot V)$, where $N$ is the number of tetrahedra in the grid and $V$ is the number of vertices per tetrahedron (4). The dominant cost is the trilinear interpolation for each vertex.
*   **Memory Complexity:** $O(1)$ auxiliary space, as the error is accumulated in a running sum. The PRI and Grid must be resident in memory.
*   **Strengths:** Provides a rigorous, quantitative measure of data fidelity independent of rendering parameters. It directly correlates with the simplification threshold used to generate the PRI.
*   **Limitations:** It is a pre-rendering metric. It does not account for visual artifacts introduced during the ray-casting process (e.g., aliasing, lighting errors). It assumes the PRI's trilinear interpolation is the ground truth for reconstruction, which may not match the actual ray-caster's sampling strategy.
*   **When to use:** When validating the quality of the PRI generation algorithm or comparing different simplification thresholds.
*   **When not to use:** When assessing the final visual quality of the rendered image; use PSNR/SSIM on pixel buffers instead.

## Background
The method originates from research on GPU-based unstructured grid volume rendering, specifically the Image-Based Modelling approach. It was introduced to evaluate the accuracy of the Pixel Ray Image (PRI) representation, which replaces irregular unstructured grids with a semi-regular structure suitable for GPU texture storage. The approach is similar to methods described in [CCM*00] for measuring sampling error in resampled data [S1, S2, unofficial]. The metric allows researchers to quantify the trade-off between data reduction (knot removal) and accuracy in the PRI simplification process [S1, unofficial].

## Sources
[S1] An Image-Based Modelling Approach To GPU-based Unstructured Grid Volume Rendering, https://doi.org/10.2312/vg/vg06/031-038, fetched 2026-10-24 (unofficial)
[S2] PDF An Image-Based Modelling Approach To GPU-based Unstructured Grid Volume ..., https://www3.cs.stonybrook.edu/~mueller/papers/vg06_ibr.pdf, fetched 2026-10-27 (unofficial)
[S3] Two Methods for Iso-Surface Extraction from Volumetric Data and Their Comparison, https://arxiv.org/abs/2201.03446v1, fetched 2026-10-26 (unofficial)
