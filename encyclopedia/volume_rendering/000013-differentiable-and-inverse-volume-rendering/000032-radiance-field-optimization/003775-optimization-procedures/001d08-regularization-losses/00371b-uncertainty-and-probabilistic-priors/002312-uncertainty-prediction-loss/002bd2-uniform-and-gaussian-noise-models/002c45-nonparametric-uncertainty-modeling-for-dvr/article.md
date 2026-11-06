# Nonparametric Uncertainty Modeling for DVR

Also called: IQR Visualization, Quantile Mean, Quantile Range

## Overview
Nonparametric Uncertainty Modeling for DVR is a technique that represents volumetric uncertainty using empirical quantiles rather than parametric distributions, enabling accurate visualization of asymmetric or multimodal noise distributions [S1]. It differs from its parent method, Uniform and Gaussian Noise Models, by avoiding distributional assumptions, thereby capturing complex statistical structures in ensemble data that parametric models fail to represent [S1].

## How it works
**Inputs:** An ensemble of $N$ volumetric datasets (members) $\{V_1, V_2, \dots, V_N\}$ sampled from an uncertain scalar field, and a 2D Transfer Function (TF) mapping intensity and gradient magnitude to color/opacity [S1].
**Outputs:** A rendered image representing the expected visual appearance of the uncertain volume [S1].

**Data Structures:**
*   **Grid Vertex:** Stores $K$ quantile values for the scalar field at that location [S1].
*   **Quantile Representation:** For a vertex $v$, the uncertainty is stored as a set of $K$ values $\{q_1, q_2, \dots, q_K\}$ corresponding to specific cumulative probabilities [S1].
*   **Memory Layout:** Each quantile value is stored as a floating-point number [S1]. The total storage per vertex is $K$ times the size of a single scalar value [S1].

**Parameters:**
*   **Ensemble Size ($N$):** The number of members in the ensemble [S1]. Minimum 50 members are recommended for stable Kernel Density Estimation (KDE) and quantile interpolation [S1].
*   **Number of Quantiles ($K$):** Typically 8 for 'Quantile Mean' and 'Quantile Range' methods [S1]. Increasing $K$ from 2 to 4 decreases Root Mean Squared Error (RMSE); 8 provides stable accuracy for sufficient ensemble sizes [S1].
*   **Interpolation Weights:** Linear interpolation weights for neighboring voxels [S1].

**Steps:**

1.  **Ensemble Generation:** Generate an ensemble of $N$ volumetric datasets by injecting noise (e.g., bimodal distribution) into ground truth data [S1].
2.  **Empirical CDF Computation:** At each grid vertex $v$, compute the empirical Cumulative Distribution Function (CDF) from the $N$ ensemble samples $\{X_1, X_2, \dots, X_N\}$ [S1].
3.  **Quantile Selection:** Select $K$ specific quantiles from the empirical CDF to represent the distribution [S1]. For example, $K=8$ equidistant quantiles [S1].
4.  **Linear Interpolation Formulation:** For a query position, interpolate the intensity $X$ and gradient magnitude $|Y|$ as linear combinations of the $M$ neighboring voxel values $X_i$ [S1].
    *   Intensity: $X = \sum_{i=1}^{M} w_i X_i$
    *   Gradient Magnitude: $|Y| = \sum_{i=1}^{M} u_i X_i$
    *   Here, $w_i$ and $u_i$ are interpolation weights derived from the geometry and the gradient calculation [S1]. The gradient magnitude approximation uses the mean gradient $\mu_Y$, obtained by substituting $X_i$ with their mean values in the gradient formula [S1].
5.  **Joint PDF Construction:** The joint Probability Density Function (PDF) of the bivariate random variable $Z = [X, |Y|]^T$ is exactly described by a bivariate box spline [S1]. The parameters of this spline are defined by the weight vectors $[w_1, u_1]^T, \dots, [w_M, u_M]^T$ associated with the neighboring voxels [S1].
6.  **Rendering Integration:** Compute the expected color at position $v$ by integrating the 2D Transfer Function $TF(x, y)$ with the joint PDF $pdf_Z(x, y)$ [S1]:
    $$ E(TF(Z)) = \iint TF(x, y) pdf_Z(x, y) dx dy $$
    This integral is evaluated using the bivariate box spline representation [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_nonparametric_dvr(ensemble, tf, num_quantiles=8, step_size=0.01):
    """
    Renders a volume using nonparametric uncertainty modeling.
    
    Args:
        ensemble: List of N volumetric datasets (members).
        tf: 2D Transfer Function mapping (intensity, gradient) to color.
        num_quantiles: Number of quantiles K to store per vertex.
        step_size: Ray marching step size.
        
    Returns:
        Rendered image as a list of RGB lists.
    """
    N = len(ensemble)
    grid_shape = ensemble[0].shape
    K = num_quantiles
    quantile_grid = compute_quantile_grid(ensemble, K)
    height, width = grid_shape[1], grid_shape[2]
    image = [[0.0, 0.0, 0.0] for _ in range(height * width)]

    for z in range(grid_shape[0]):
        for y in range(height):
            for x in range(width):
                ray_origin = [float(x), float(y), float(z)]
                ray_direction = [0.0, 0.0, 1.0]
                color = [0.0, 0.0, 0.0]
                alpha = 0.0
                t = 0.0
                while t < grid_shape[0]:
                    pos = [ray_origin[i] + t * ray_direction[i] for i in range(3)]
                    X_val, Y_val = interpolate_linear_combination(pos, quantile_grid, K)
                    pdf_val = bivariate_box_spline_pdf(X_val, Y_val)
                    tf_color = tf(X_val, Y_val)
                    weight = pdf_val * step_size
                    color = [color[i] + tf_color[i] * weight * (1.0 - alpha) for i in range(3)]
                    alpha += weight * (1.0 - alpha)
                    t += step_size
                image[y * width + x] = color
    return image

def compute_quantile_grid(ensemble, K):
    """
    Computes K quantiles for each vertex across the ensemble.
    """
    N = len(ensemble)
    grid_shape = ensemble[0].shape
    grid = [[[0.0] * K for _ in range(grid_shape[2])] for _ in range(grid_shape[1])]
    for z in range(grid_shape[1]):
        for x in range(grid_shape[2]):
            values = [ensemble[m][z][x] for m in range(N)]
            values.sort()
            for k in range(K):
                p = (k + 1) / (K + 1)
                idx = int(p * (N - 1))
                idx = min(idx, N - 1)
                grid[z][x][k] = values[idx]
    return grid

def interpolate_linear_combination(pos, quantile_grid, K):
    """
    Computes X and |Y| as linear combinations of M neighboring voxels.
    Uses trilinear interpolation weights w_i and u_i.
    """
    x, y, z = pos
    # Nearest neighbor sampling for self-contained demonstration
    idx_z = int(z) % len(quantile_grid)
    idx_x = int(x) % len(quantile_grid[0])
    voxel_quantiles = quantile_grid[idx_z][idx_x]
    # X is the mean of quantiles, |Y| is a scaled difference (gradient proxy)
    X_val = sum(voxel_quantiles) / K
    Y_val = abs(voxel_quantiles[-1] - voxel_quantiles[0])
    return X_val, Y_val

def bivariate_box_spline_pdf(x, y):
    """
    Evaluates the bivariate box spline PDF at (x, y).
    Uses a standard tensor-product linear kernel [G].
    """
    return max(0.0, 1.0 - abs(x - 0.5)) * max(0.0, 1.0 - abs(y - 0.5))
```

**Edge Cases and Pitfalls:**
*   **Small Ensemble Size:** If the ensemble size is small (e.g., $N=5$), the average number of samples per quantile is low, leading to unreliable quantile interpolation and poor Kernel Density Estimation (KDE) at grid vertices [S1]. In such cases, using more quantiles (4 or 8) may result in higher RMSE than using fewer (2) [S1].
*   **Multimodal Distributions:** Parametric models (Gaussian/Uniform) fail to capture multimodal noise [S1]. Nonparametric models handle this naturally by storing multiple quantiles [S1].

## Operating characteristics
**Time and Memory Complexity:**
*   **Memory:** Memory usage scales linearly with the number of quantiles $K$ [S1].
    *   Mean-field: 1 unit per vertex [S1].
    *   Uniform/Gaussian: 2 units per vertex (mean + width/variance) [S1].
    *   Gaussian Mixture Model (GMM, 4 Gaussians): 12 units per vertex (mean, variance, weight per Gaussian) [S1].
    *   Nonparametric ($K=8$): 9 units per vertex [S1].
    *   IQR (single quantile, middle 50%): 2 units per vertex [S1].
*   **Time:** Rendering performance is comparable to parametric models but depends on the complexity of evaluating the bivariate box spline integral [S1]. The mean-field is rendered at 10 fps on a Quadro P6000; nonparametric models add overhead for quantile interpolation and PDF evaluation [S1].

**Measured Performance:**
*   On a synthetic tangle dataset with 50 ensemble members:
    *   RMSE for Mean-field: 0.0245 [S1].
    *   RMSE for Uniform: 0.0200 [S1].
    *   RMSE for Nonparametric (8 quantiles): Lower than mean/uniform/Gaussian [S1].
    *   RMSE for IQR (middle 50%): Lowest among all visualizations [S1].
*   Tested on real Red Sea eddy simulation ensemble (20 members, 500x500x50 resolution) [S1].

**Strengths and Limitations:**
*   **Strengths:** Accurately reconstructs asymmetric or multimodal noise distributions [S1]. Lower RMSE than parametric models for complex noise [S1]. Flexible memory usage based on required accuracy [S1].
*   **Limitations:** Requires larger ensemble sizes ($N \ge 50$) for stable results [S1]. Higher memory usage than mean-field or IQR [S1]. Sensitive to small sample sizes [S1].

**When to Use:**
*   Use when the underlying uncertainty distribution is known or suspected to be non-Gaussian, asymmetric, or multimodal [S1].
*   Use when high-fidelity uncertainty visualization is required and sufficient ensemble members are available [S1].

**When Not to Use:**
*   Avoid when ensemble size is very small ($N < 10$), as quantile interpolation becomes unreliable [S1].
*   Avoid when memory is extremely constrained and mean-field accuracy is sufficient [S1].

## Background
**Origin:** The method was introduced in the paper "Direct Volume Rendering with Nonparametric Models of Uncertainty" [S1].
**History:** It was developed to address the limitations of parametric statistical models (mean, uniform, Gaussian) in volume rendering, which assume specific distribution shapes that may not reflect real-world uncertainty [S1].
**Usage:** Implemented in the Voreen volume rendering engine for DVR of uncertain data [S1]. Tested on synthetic datasets (tangle, teardrop) and real-world data (Red Sea eddy simulation) [S1].

## Sources
[S1] Direct Volume Rendering with Nonparametric Models of Uncertainty, https://doi.org/10.1109/tvcg.2020.3030394, fetched 2026-10-26
