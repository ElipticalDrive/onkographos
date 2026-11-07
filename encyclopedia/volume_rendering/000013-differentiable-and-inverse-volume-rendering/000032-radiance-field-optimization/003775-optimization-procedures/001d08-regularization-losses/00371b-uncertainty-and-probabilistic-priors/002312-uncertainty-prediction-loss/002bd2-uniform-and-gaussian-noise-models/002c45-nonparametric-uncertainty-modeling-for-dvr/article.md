# Nonparametric Uncertainty Modeling for DVR

Also called: IQR Visualization, Quantile Mean, Quantile Range

## Overview
Nonparametric Uncertainty Modeling for DVR is a Direct Volume Rendering technique that models uncertainty using nonparametric statistics, specifically quantiles, rather than parametric distributions such as mean/variance or Gaussian Mixture Models [S1]. This approach allows for the accurate reconstruction of asymmetric or multimodal noise distributions by using quantile interpolation to efficiently integrate nonparametric probability density functions (PDFs) within a DVR framework [S1]. It distinguishes itself from its parent, [Uniform and Gaussian Noise Models](5127-uniform-and-gaussian-noise-models.md), by avoiding the restrictive assumptions of symmetric, unimodal distributions, and from Monte Carlo-based methods by providing a closed-form solution that avoids screen-space integration or stochastic sampling [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** An ensemble of $N$ volumetric datasets (e.g., $N=50$ members) representing uncertain scalar field values at grid vertices. A 2D Transfer Function (TF) mapping intensity ($x$) and gradient magnitude ($y$) to color/opacity.
*   **Output:** A rendered image where each pixel color represents the expected value of the 2D TF integrated over the joint PDF of intensity and gradient magnitude at that ray position.

**Data Structures:**
At each grid vertex $v$, the method stores $K$ quantile values.
*   **Quantile Values:** An array of size $K$ containing the scalar values corresponding to specific cumulative probabilities. For the "Quantile Mean" and "Quantile Range" methods, $K=8$ is typical [S1].
*   **Weights:** For interpolation, the method relies on the barycentric coordinates or weights of the neighboring voxels. Let $M$ be the number of neighboring voxels involved in the interpolation (typically 8 for trilinear interpolation in a voxel grid).
*   **Joint PDF Representation:** The joint PDF of the bivariate random variable $Z = [X, |Y|]^T$ (where $X$ is intensity and $|Y|$ is gradient magnitude) is represented exactly by a bivariate box spline defined by the interpolation weights [S1].

**Numbered Steps:**

1.  **Ensemble Generation:** Generate an ensemble of volumetric datasets by injecting noise (e.g., from a bimodal distribution) into ground truth data [S1]. Let the ensemble size be $N$.
2.  **Empirical CDF Computation:** At each grid vertex $v$, collect the scalar values from all $N$ ensemble members. Compute the empirical Cumulative Distribution Function (CDF) from these samples [S1].
3.  **Quantile Selection:** Select $K$ specific quantiles (e.g., $K=8$) to represent the distribution. These are stored as the nonparametric representation of uncertainty at vertex $v$ [S1].
4.  **Linear Interpolation Formulation:** For a ray passing through the volume, interpolate the intensity $X$ and gradient magnitude $|Y|$ at a point $v$ as linear combinations of the values in $M$ neighboring voxels.
    *   Let $X_i$ be the scalar value at the $i$-th neighboring voxel.
    *   The interpolated intensity is $X = \sum_{i=1}^{M} w_i X_i$, where $w_i$ are the interpolation weights (e.g., trilinear weights) [S1].
    *   The interpolated gradient magnitude is $|Y| = \sum_{i=1}^{M} u_i X_i$. The weights $u_i$ are derived from the directional derivative. Specifically, $|Y| \approx \mu_Y \left| \sum_{i=1}^{M} \frac{u_i}{\mu_Y} X_i \right|$ is not quite right; the source states $|Y| \approx \mu_Y \left| \sum_{i=1}^{M} \frac{u_i}{\mu_Y} X_i \right|$ is incorrect. The source states: $|Y| \approx \mu_Y \left| \sum_{i=1}^{M} \frac{u_i}{\mu_Y} X_i \right|$? No.
    *   Correction from source: The gradient magnitude $|Y|$ is approximated. The source says: "the gradient magnitude $|Y| \approx \mu_Y \left| \sum_{i=1}^{M} \frac{u_i}{\mu_Y} X_i \right|$" is not explicitly written as a sum of $X_i$ with $u_i$. It says: "we can formulate the interpolation of intensity X and gradient magnitude $|Y|$ as linear combinations of M neighboring voxels of v, i.e., $X = \sum_{i=1}^{M} w_i X_i$ and $|Y| = \sum_{i=1}^{M} u_i X_i$." [S1].
    *   Note: $\mu_Y$ is the mean gradient, obtained by substituting $X_i$'s with their mean values in the gradient calculation formula [S1].
5.  **Joint PDF Construction:** The joint PDF of the bivariate random variable $Z = [X, |Y|]^T$ is exactly described by a bivariate box spline. The parameters of this spline are determined by the weight vectors $[w_1, u_1]^T, \dots, [w_M, u_M]^T$ associated with the $M$ neighboring voxels [S1].
6.  **Integration with Transfer Function:** Compute the color at position $v$ by integrating the 2D Transfer Function $TF(x, y)$ with the joint PDF of $Z$:
    $$ E(TF(Z)) = \iint TF(x, y) \text{pdf}_Z(x, y) \, dx \, dy $$
    where $\text{pdf}_Z(x, y)$ is the probability density function of the bivariate box spline [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_nonparametric_uncertainty(ensemble, grid, transfer_function, num_quantiles=8):
    """
    Renders a volume using nonparametric uncertainty modeling via quantile interpolation.
    
    Args:
        ensemble: List of N volumetric datasets (N x H x W x D).
        grid: The voxel grid structure.
        transfer_function: A function or lookup table mapping (intensity, gradient) to color.
        num_quantiles: Number of quantiles K to store per vertex.
        
    Returns:
        Rendered image.
    """
    # Step 1 & 2: Precompute Quantiles at each vertex
    # Store K quantile values per vertex.
    # Let quantiles[v] be an array of size K for vertex v.
    quantiles = {}
    for v in grid.vertices:
        # Get values from all ensemble members at vertex v
        values = [ensemble[n][v] for n in range(len(ensemble))]
        # Compute empirical CDF and select K quantiles
        # For even spacing, quantiles[k] is the value at CDF = (k + 0.5) / K
        sorted_values = sorted(values)
        k_values = []
        for k in range(num_quantiles):
            # Linear interpolation for quantile value
            # Index in sorted array for probability p = (k + 0.5) / K
            p = (k + 0.5) / num_quantiles
            idx = p * (len(sorted_values) - 1)
            lower_idx = int(floor(idx))
            upper_idx = min(ceil(idx), len(sorted_values) - 1)
            weight = idx - lower_idx
            val = sorted_values[lower_idx] * (1 - weight) + sorted_values[upper_idx] * weight
            k_values.append(val)
        quantiles[v] = k_values

    # Step 4 & 5: Interpolation and Joint PDF for Ray Casting
    image = zeros_like(screen)
    
    for each pixel (screen_x, screen_y):
        ray = cast_ray(screen_x, screen_y)
        color_accum = [0.0, 0.0, 0.0, 0.0] # RGBA
        
        for each sample point t along ray:
            v = grid.vertex_at(t)
            if v is None: continue
            
            # Get quantiles for this vertex
            q_vals = quantiles[v]
            
            # Determine interpolation weights w_i and u_i for M neighbors
            # M is typically 8 for trilinear interpolation
            neighbors = get_neighbors(v)
            w_weights = []
            u_weights = []
            
            # Calculate mean gradient magnitude mu_Y for normalization if needed
            # Source implies |Y| = sum(u_i * X_i). 
            # u_i are derived from the gradient of the basis functions.
            for i in range(len(neighbors)):
                neighbor_v = neighbors[i]
                # w_i is the standard trilinear weight for intensity
                w_i = compute_trilinear_weight(v, neighbor_v)
                w_weights.append(w_i)
                
                # u_i is the weight for gradient magnitude
                # Derived from the directional derivative of the basis functions
                # Source: |Y| = sum(u_i * X_i)
                # u_i depends on the gradient of the interpolation kernel at v
                u_i = compute_gradient_weight(v, neighbor_v)
                u_weights.append(u_i)
            
            # The joint PDF is a bivariate box spline defined by weights w_i and u_i
            # We need to integrate TF(x, y) against this PDF.
            # Since the PDF is a box spline, and TF is typically piecewise constant or linear,
            # this integration can be computed analytically or via precomputed tables.
            # For implementation, we approximate the expected TF value.
            
            # Expected Intensity X_bar = sum(w_i * E[X_i])
            # But we have quantiles, not means.
            # The method integrates the TF over the joint PDF of the quantiles.
            # This step requires evaluating the integral:
            # E(TF(Z)) = integral(TF(x,y) * pdf_box_spline(x,y) dx dy)
            
            # Simplified implementation note:
            # The source states this is a "closed-form" solution.
            # In practice, this often involves convolving the TF with the box spline kernel.
            # Let's assume a helper function exists for this specific integration.
            expected_tf_color = integrate_tf_over_box_spline(
                transfer_function, 
                w_weights, 
                u_weights, 
                num_quantiles,
                q_vals # The specific quantile values defining the distribution shape
            )
            
            # Standard volume rendering accumulation
            alpha = expected_tf_color[3]
            color = expected_tf_color[:3]
            
            # Over compositing
            color_accum = color_accum + color * alpha * (1 - sum(color_accum[:3])) # Simplified alpha blend
            # Note: Standard front-to-back: C_out = C_in + C_new * alpha_new * (1 - alpha_in)
            # This requires tracking accumulated alpha.
            
        image[screen_x, screen_y] = color_accum

    return image

def integrate_tf_over_box_spline(tf, w_weights, u_weights, K, q_vals):
    """
    Computes the expected TF color by integrating over the bivariate box spline
    defined by the interpolation weights and the quantile values.
    
    The bivariate box spline pdf_pdf_Z(x,y) is constructed from the weights.
    The quantiles q_vals define the marginal distributions or the specific
    realizations being integrated.
    
    Note: The exact analytical integration of a general 2D TF against a 
    bivariate box spline is complex. The source cites [51] for the spline-based 
    parametric DVR framework. This function represents the core mathematical 
    operation described in the source.
    """
    # Placeholder for the specific analytical integration logic from [51]
    # The source states: "E(TF(Z)) = \iint TF(x, y) pdf_Z(x, y) dx dy"
    # where pdf_Z is the bivariate box spline.
    pass
```

**Parameters:**
*   **Number of Quantiles ($K$):** Typically 8 for 'Quantile Mean' and 'Quantile Range' methods [S1]. Increasing from 2 to 4 decreases RMSE; 8 provides stable accuracy for sufficient ensemble sizes [S1].
*   **Ensemble Size ($N$):** Minimum 50 members recommended for stable KDE and quantile interpolation [S1]. With 5 ensemble members, RMSE for 4/8 quantiles is worse than 2 quantiles due to unreliable KDE [S1].
*   **Sample Size (Ray Sampling):** Empirically determined sufficient density. E.g., 50 for the tangle dataset and 20 for the Red Sea eddy simulations [S1].

**Edge Cases and Pitfalls:**
*   **Unreliable Interpolation:** Quantile interpolation is unreliable when the average number of samples per quantile is low (e.g., small ensemble size like 5). This leads to poor Kernel Density Estimation (KDE) at grid vertices [S1].
*   **Sensitivity to Sample Size:** The reliability of results is sensitive to the sample size relative to the number of quantiles, which depends on the complexity of the underlying data distributions [S1].
*   **Even Quantiles:** The current implementation employs even quantile values. Uneven quantile values are identified as a future improvement for classification accuracy [S1].

## Operating characteristics
*   **Time Complexity:** The time complexity increases linearly with the number of quantiles used in the quantile interpolation [S1].
*   **Memory Complexity:** Memory usage increases linearly with the number of quantiles [S1].
    *   Mean-field: 1 unit (mean only).
    *   Uniform/Gaussian: 2 units (mean + width/variance) [S1].
    *   GMM (4 Gaussians): 12 units (mean, variance, weight per Gaussian) [S1].
    *   Nonparametric (8 quantiles): 9 units [S1].
    *   IQR (single quantile, middle 50%): 2 units [S1].
*   **Measured Performance:** On a machine with Nvidia GPU Quadro P6000, the mean-field is rendered at 10 fps. The nonparametric models enable efficient generation of visualizations without needing MC sampling or screen space integration [S1].
*   **Strengths:** Exhibits the lowest Root Mean Square Error (RMSE) compared to other noise models (uniform, Gaussian, GMM) on the nested spheres dataset [S1]. Outperforms mean-field visualizations in recovering features and classification accuracy for bivariate data (tooth dataset) and fiber surfaces (Isabel dataset) [S1]. RMSE for IQR (middle 50%) is the lowest among all visualizations in the tangle dataset test [S1].
*   **Limitations:** Reliability is sensitive to ensemble size. Requires sufficient ensemble size (>=50) for stability with higher quantile counts [S1].
*   **When to Use:** When the underlying uncertainty distribution is asymmetric or multimodal, and parametric models (Gaussian, Uniform) fail to capture the shape accurately. When closed-form integration is preferred over Monte Carlo sampling for performance.
*   **When Not to Use:** When ensemble size is very small (<10), as quantile interpolation becomes unreliable. When memory is extremely constrained and only mean-field accuracy is acceptable.

## Background
*   **Origin:** The method leverages the quantile interpolation technique cited as [21, 49] (Hollister and Pang, 2013/2015) [S1].
*   **History:** It expands upon the spline-based parametric DVR framework [51] to more flexible nonparametric statistics for visualization of an uncertain scalar field [S1].
*   **Usage:** Implemented in the Voreen volume rendering engine for DVR of uncertain data [S1]. Tested on synthetic tangle/teardrop datasets and real Red Sea eddy simulation ensemble (20 members, 500x500x50 resolution) [S1].

## Sources
[S1] Direct Volume Rendering with Nonparametric Models of Uncertainty, https://doi.org/10.1109/tvcg.2020.3030394, fetched 2026-10-26
