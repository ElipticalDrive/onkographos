# PRI Approximation Error Measurement

Also called: PRI Error Metric, PSNR calculation for PRI

## Overview
This method quantifies the geometric and scalar approximation error introduced when an unstructured grid volume is resampled into a Pixel Ray Image (PRI). It computes the Mean Square Error (MSE) between the original grid data and the data reconstructed from the PRI, serving as the basis for calculating Peak Signal-to-Noise Ratio (PSNR) to evaluate rendering fidelity [S1, S2, unofficial]. It differs from sibling methods like [PRI Texture Storage](2493-pri-texture-storage.md) by focusing on quantitative error analysis rather than data layout optimization.

## How it works
**Inputs:**
*   $V$: The original unstructured grid (composed of tetrahedra).
*   $P$: The PRI data structure.
*   $S(v)$: The scalar value at vertex $v$ in the original grid $V$.

**Outputs:**
*   $MSE(V, P)$: The weighted mean square error.
*   $\varepsilon_P(t)$: The maximal residual error for a specific tetrahedron $t$.

**Data Structures:**
*   The method operates on the logical representation of the grid $V$ and the reconstructed scalar field from $P$. No specific binary layout is required for the error calculation itself, as it relies on evaluating vertices.

**Steps:**
1.  Iterate over every tetrahedron $t$ in the original grid $V$.
2.  For each tetrahedron $t$, identify its four vertices $v \in t$.
3.  For each vertex $v$, retrieve the original scalar value $S(v)$ from the grid $V$.
4.  Reconstruct the scalar value $S_P(v)$ at the same spatial location using the PRI $P$. This is done via trilinear interpolation of the PRI data [S1, S2, unofficial].
5.  Compute the residual error for the vertex: $|S(v) - S_P(v)|$.
6.  Compute the maximal residual error for the tetrahedron $t$:
    $$ \varepsilon_P(t) = \max_{\forall v \in t} |S(v) - S_P(v)| $$
    This value represents the largest deviation between the original grid and the PRI approximation within that tetrahedron [S1, S2, unofficial].
7.  Calculate the volume $|t|$ of the tetrahedron.
8.  Accumulate the squared error weighted by volume to compute the MSE:
    $$ MSE(V, P) = \frac{\sum_{\forall t \in V} \varepsilon_P(t)^2 |t|}{\sum_{\forall t \in V} |t|} $$
9.  (Optional) Use the MSE to compute PSNR if the maximum possible scalar value range is known.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_pri_error(grid_V, pri_P):
    """
    Computes the MSE between unstructured grid V and PRI P.
    """
    total_weighted_error = 0.0
    total_volume = 0.0

    # Iterate over all tetrahedra in the original grid
    for tet in grid_V.tetrahedra:
        tet_volume = tet.volume()
        total_volume += tet_volume
        
        max_residual = 0.0
        
        # Evaluate error at all four vertices of the tetrahedron
        for vertex in tet.vertices:
            # Original scalar value from grid
            s_original = vertex.scalar_value
            
            # Reconstructed scalar value from PRI via trilinear interpolation
            # pri_P.interpolate() performs trilinear interpolation at vertex.position
            s_reconstructed = pri_P.interpolate(vertex.position)
            
            residual = abs(s_original - s_reconstructed)
            if residual > max_residual:
                max_residual = residual
        
        # Square the maximal residual for this tetrahedron
        total_weighted_error += (max_residual ** 2) * tet_volume

    if total_volume == 0:
        return 0.0

    mse = total_weighted_error / total_volume
    return mse
```

**Parameters:**
*   Error threshold: In practice, this metric is used to determine simplification thresholds. An error threshold of 0.0 results in the removal of superfluous knots (samples in constant or linearly changing regions) [S1, unofficial].

**Edge Cases:**
*   If the PRI cannot reconstruct a value (e.g., outside bounds), the interpolation behavior must be defined (e.g., clamping or zeroing), though standard trilinear interpolation is assumed [S1, S2, unofficial].
*   Precision issues may arise if $S(v)$ and $S_P(v)$ are very close; double precision is recommended for error accumulation.

## Operating characteristics
*   **Complexity:** $O(N_t)$, where $N_t$ is the number of tetrahedra in the grid. Each tetrahedron requires 4 interpolation queries.
*   **Performance:** The calculation is typically performed offline or during simplification steps, not per-frame. It is computationally inexpensive relative to the rendering cost but requires random access to the original grid data.
*   **Strengths:** Provides a physically meaningful error metric weighted by volume, accounting for the size of geometric elements.
*   **Limitations:** Requires access to the original unstructured grid $V$ and its vertex data, which may not be available in purely image-based pipelines where only the PRI is retained.

## Background
The method was introduced in the context of GPU-based unstructured grid volume rendering using an image-based modeling approach. It adapts techniques from prior work (cited as [CCM*00]) to measure sampling error in the specific context of PRI resampling [S1, S2, unofficial]. The metric is used to guide simplification algorithms that remove superfluous knots from the splines defining the PRI [S1, unofficial].

## Sources
[S1] An Image-Based Modelling Approach To GPU-based Unstructured Grid Volume Rendering, https://doi.org/10.2312/vg/vg06/031-038, fetched 2026-10-24 (unofficial)
[S2] PDF An Image-Based Modelling Approach To GPU-based Unstructured Grid Volume ..., https://www3.cs.stonybrook.edu/~mueller/papers/vg06_ibr.pdf, fetched 2026-10-27 (unofficial)
