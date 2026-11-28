# High-Quality Wavelet Splatting

Also called: Volume wavelet rendering, Wavelet Projections for Volume Rendering, Wavelet Splats, Wavelet Splatting, Wavelet splatting with least-squares approximation, Wavelet-based splatting

## Overview
High-Quality Wavelet Splatting is a volume rendering technique that decomposes volume data into a wavelet pyramid representation in the spline domain and computes splats via an orthogonal projection operator using least-squares approximation on an adaptive multiresolution grid [S1]. It allows full control over approximation error and achieves the smallest possible error for a given sampling step size [S1]. Unlike [Wavelet Splatting Method 1](2848-wavelet-splatting-method-1.md), which decays complexity slowly ($2^{-j}$), this method decays complexity rapidly ($2^{-3j}$) at small scales, offering significant speedups for large volumes while maintaining higher quality, particularly for data with significant high-frequency content [S1].

## How it works
The method operates by projecting volume data onto a multiresolution grid, interpolating the projected data to full-screen resolution, and visualizing it in cardinal space.

### Inputs and Outputs
*   **Input:** Volume data of resolution $M$ per dimension.
*   **Input:** Rendering screen resolution $N$ per dimension.
*   **Input:** Scale parameter $j$.
*   **Input:** Oversampling factor $h$ (typically $h=1$ for complexity comparison) [S1].
*   **Input:** Filter half-width parameter $n$ [S1].
*   **Output:** Rendered image at resolution $N^2$ in cardinal space [S1].

### Data Structures
1.  **Wavelet Pyramid:** The volume data is decomposed into a wavelet pyramid representation in the spline domain [S1].
2.  **Multiresolution Grid:** An adaptive grid where the approximation error is minimized. The grid resolution adapts to the scale $j$, with the sampling step size defined as $h_j = 2^j \cdot h$ [S1].
3.  **Projected Data:** The volume data projected onto the multiresolution grid using an orthogonal projection operator [S1].

### Steps
1.  **Wavelet Decomposition:** Decompose the input volume data into a wavelet pyramid representation in the spline domain [S1].
2.  **Orthogonal Projection:** Compute splats via an orthogonal projection operator onto the multiresolution grid [S1].
3.  **Least-Squares Approximation:** Approximate the splats of the basis functions on the multiresolution grid using least-squares approximation to minimize error for the given sampling step size $h_j$ [S1]. The approximation error on the grid is derived as a function of the sampling step $h$ [S1].
4.  **Interpolation to Full Resolution:** Interpolate the projected data from the multiresolution grid to the full-screen resolution $N^2$. This is achieved in three sub-steps [S1]:
    1.  **Dual to Spline:** Change from dual space to spline space. This step is fast because the low-resolution data is smaller in size [S1].
    2.  **Upsampling:** Use spline space filters to reach the final full resolution. The spline space has the shortest upsampling interpolation filters [S1].
    3.  **Spline to Cardinal:** Change from spline space to cardinal space for visualization [S1].

### Formulas
The total number of operations $\text{Op}_2$ for this method is given by [S1]:

$$
\text{Op}_2 = k_3 \left( 2^{-j} M \right)^3 \cdot (3n+1)^2 h^{-2} + k_7 N^2 \left( 2^{-2j} h^{-2} + 2^{-1} \right) \cdot (2n+1)^2
$$

Which simplifies to [S1]:

$$
\text{Op}_2 \sim 2^{-3j} \cdot M^3 + k_7 N^2
$$

**Symbol Definitions:**
*   $j$: Scale parameter.
*   $M$: Volume data resolution per dimension.
*   $n$: Filter half-width parameter.
*   $h$: Oversampling factor.
*   $N$: Rendering screen resolution per dimension.
*   $k_3, k_7$: Constants [S1].

### Parameters and Typical Values
*   **$h$ (Oversampling factor):** Typical value used in complexity comparison is $h=1$ (no oversampling) [S1].
*   **$N$ (Screen resolution):** In comparisons, assumed equal to volume data resolution $M$ ($N=M$) [S1].
*   **$M$ (Volume resolution):** Typical values for large volume comparison are $M \in \{512, 1024, 2048\}$ [S1].
*   **$j$ (Scale parameter):** Typical values for significant speedup are $j \in \{4, 5\}$ [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def high_quality_wavelet_splatting(volume_data, M, N, j, h=1, n=1):
    """
    Performs high-quality wavelet splatting.
    
    Args:
        volume_data: 3D array of size M x M x M.
        M: Volume resolution per dimension.
        N: Screen resolution per dimension.
        j: Scale parameter.
        h: Oversampling factor (default 1).
        n: Filter half-width parameter (default 1).
        
    Returns:
        image: 2D array of size N x N.
    """
    # Step 1: Decompose volume into wavelet pyramid in spline domain
    # [G] Implementation of wavelet decomposition in spline domain is assumed.
    wavelet_pyramid = decompose_wavelet_pyramid(volume_data)
    
    # Step 2 & 3: Project and approximate on multiresolution grid
    # The grid resolution adapts to scale j: h_j = 2^j * h
    h_j = (2 ** j) * h
    
    # [G] The orthogonal projection operator and least-squares approximation
    # are applied to minimize error for the given sampling step size h_j.
    projected_data = orthogonal_projection_and_approximate(
        wavelet_pyramid, 
        grid_resolution=h_j, 
        filter_half_width=n
    )
    
    # Step 4: Interpolate to full-screen resolution N^2
    # Sub-step 1: Change from dual space to spline space
    spline_data = dual_to_spline_conversion(projected_data)
    
    # Sub-step 2: Use spline space filters to reach full resolution
    # [G] Shortest upsampling interpolation filters are used here.
    upscaled_data = spline_upsampling(spline_data, target_resolution=N)
    
    # Sub-step 3: Change from spline to cardinal space
    image = spline_to_cardinal_conversion(upscaled_data)
    
    return image
```

### Edge Cases and Pitfalls
*   **Full Resolution Complexity:** At full resolution (where $N=M$ and $j=0$), the complexity of this method is 50% higher than the direct wavelet splatting method [S1]. However, it yields higher quality results, especially for data with significant high-frequency content [S1].
*   **Precision:** The method relies on least-squares approximation. Precision depends on the sampling step size $h$ and the filter half-width $n$ [S1].

## Operating characteristics
*   **Time Complexity:** The complexity $\text{Op}_2$ decays with a factor $8^{-j}$ (or $2^{-3j}$) per scale $j$ at small scales, until it reaches a constant depending only on the rendering screen size $N^2$ [S1]. This is the theoretical limit as complexity cannot decrease faster than the number of voxels decrease per scale [S1].
*   **Comparison with Method 1:** Method 2 is significantly faster than Method 1 ($\text{Op}_1$) for scale $j > 1$ [S1]. Method 2 is up to two orders of magnitude faster at scales $j \in \{4, 5\}$ for large volumes ($M \in \{512, 1024, 2048\}$) [S1]. Method 2 complexity decays as $2^{-3j}$ per scale, whereas Method 1 decays only as $2^{-j}$ [S1].
*   **Strengths:** Allows full control over approximation error; achieves the smallest error for a given sampling step size; up to two orders of magnitude faster than standard methods; yields higher quality for a given resolution, especially for high-frequency data [S1].
*   **Limitations:** At full resolution, complexity is 50% higher than direct wavelet splatting [S1].
*   **Usage:** Suitable for interactive or progressive volume rendering where speedups are critical and high quality is maintained [S1].

## Background
*   **Origin:** Proposed by Stefan Horbelt, Michael Unser, and Martin Vetterli from the Swiss Federal Institute of Technology (EPFL) [S1].
*   **History:** The method extends Gross's method of volume wavelet rendering by computing splats via an orthogonal projection operator [S1].
*   **Sources:** [S1]

## Sources
[S1] Wavelet Projections for Volume Rendering, https://doi.org/10.2312/egs.19991026, fetched 2026-10-26
