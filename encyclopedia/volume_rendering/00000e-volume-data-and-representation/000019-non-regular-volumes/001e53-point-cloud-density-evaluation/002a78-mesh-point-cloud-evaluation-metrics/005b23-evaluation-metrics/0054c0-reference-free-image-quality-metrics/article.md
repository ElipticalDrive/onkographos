# Reference-free Image Quality Metrics

Also called: First evaluation method, Image Entropy, Image Mean, Image Quality Evaluation Metrics, Mean Gradient, Reference-free evaluation indices, Reference-free image quality evaluation, Standard Deviation

## Overview
Reference-free image quality metrics provide an objective assessment of rendered volume visualization quality when an ideal ground-truth reference image is unavailable [S1, S2]. Unlike [Evaluation Metrics](9899-evaluation-metrics.md) which rely on pixel-wise comparison with a reference, this method evaluates statistical properties of the rendered image itself, specifically targeting the clarity of microstructures and the reduction of visual occlusion [S1, S2]. It is distinguished from sibling metrics like [Multi-Scale Entropy for View Quality](17037-multi-scale-entropy-for-view-quality.md) by its reliance on simple histogram and gradient statistics rather than multi-resolution wavelet analysis [S1].

## How it works
The method computes four distinct indices from the rendered image to quantify visual quality: Image Entropy ($H$), Mean Gradient ($G$), Image Mean ($U$), and Standard Deviation ($Std$) [S1, S2].

**Inputs and Outputs:**
*   **Input:** A rendered image $I$ of dimensions $m \times n$ pixels.
*   **Input:** $L$, the maximum gray level (bit depth range, e.g., 255 for 8-bit).
*   **Output:** Four scalar values: $H, G, U, Std$.

**Data Structures:**
*   $I(i,j)$: The gray value of the pixel at row $i$ and column $j$, where $1 \le i \le m$ and $1 \le j \le n$.
*   $H_r$: The count of pixels in the image having gray level $r$, where $0 \le r < L$.

**Formulas and Definitions:**

1.  **Image Entropy ($H$):** Measures the "chaos" or randomness of the pixel gray level distribution [S1, S2].
    $$ H = - \sum_{r=0}^{L-1} \frac{H_r}{m \cdot n} \log \left( \frac{H_r}{m \cdot n} \right) $$
    *   $H_r$: Number of pixels with gray level $r$.
    *   $m \cdot n$: Total number of pixels.
    *   Note: If $H_r = 0$, the term $0 \log 0$ is treated as 0 [S2].

2.  **Mean Gradient ($G$):** Reflects the rate of contrast change of tiny details, characterizing microstructure clarity [S1, S2].
    $$ G = \frac{1}{(m-1)(n-1)} \sum_{i=1}^{m-1} \sum_{j=1}^{n-1} \sqrt{\Delta I_x^2 + \Delta I_y^2} $$
    *   $\Delta I_x$: First-order gradient in the x-direction.
    *   $\Delta I_y$: First-order gradient in the y-direction.
    *   Implementation detail: $\Delta I_x = I(i+1, j) - I(i, j)$ and $\Delta I_y = I(i, j+1) - I(i, j)$ [S2].

3.  **Image Mean ($U$):** The average pixel gray value [S1, S2].
    $$ U = \frac{1}{m \cdot n} \sum_{i=1}^{m} \sum_{j=1}^{n} I(i,j) $$

4.  **Standard Deviation ($Std$):** The dispersion of pixel gray values with respect to the mean [S1, S2].
    $$ Std = \sqrt{ \frac{1}{m \cdot n} \sum_{i=1}^{m} \sum_{j=1}^{n} (I(i,j) - U)^2 } $$

**Interpretation and Edge Cases:**
*   **General Content Richness:** In general image analysis, larger $H$ and $G$ indicate richer detail, while larger $U$ and $Std$ indicate higher contrast and brightness variation [S1].
*   **Explosion Field/Volume Visualization Context:** When evaluating specific volume rendering tasks like explosion fields where the goal is to eliminate visual occlusion and make specific phenomena (shock waves, jets) visible, the interpretation reverses for $U$ and $Std$. Smaller $U$ and $Std$ values indicate reduced visual chaos/occlusion and are considered better quality [S1, S2].
*   **Gradient Boundaries:** The summation for $G$ excludes the last row and column ($m-1, n-1$) to avoid out-of-bounds access for the forward difference operator [S2].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_reference_free_metrics(image: list[list[float]], max_gray_level: int) -> dict:
    """
    Computes H, G, U, and Std for a rendered image.
    
    Args:
        image: 2D list/array of pixel values I(i,j). Dimensions m x n.
        max_gray_level: L, the maximum possible gray value (e.g., 255).
        
    Returns:
        Dictionary with keys 'H', 'G', 'U', 'Std'.
    """
    m = len(image)
    n = len(image[0])
    
    # 1. Compute Image Mean (U)
    total_sum = 0.0
    for i in range(m):
        for j in range(n):
            total_sum += image[i][j]
    U = total_sum / (m * n)
    
    # 2. Compute Standard Deviation (Std)
    variance_sum = 0.0
    for i in range(m):
        for j in range(n):
            diff = image[i][j] - U
            variance_sum += diff * diff
    Std = (variance_sum / (m * n)) ** 0.5
    
    # 3. Compute Image Entropy (H)
    # Build histogram
    histogram = [0] * max_gray_level
    for i in range(m):
        for j in range(n):
            # Cast to int and clamp to valid range [0, max_gray_level-1]
            val = int(image[i][j])
            if 0 <= val < max_gray_level:
                histogram[val] += 1
    
    H = 0.0
    total_pixels = m * n
    for r in range(max_gray_level):
        count = histogram[r]
        if count > 0:
            prob = count / total_pixels
            # log base e or 2 is not specified in sources; natural log is standard in information theory unless specified.
            # Sources use "log" without base. We assume natural log or base 2. 
            # Given the context of "entropy", base 2 is common, but mathematical notation often implies natural.
            # We will use natural log as per standard calculus notation in [S2], 
            # but note that implementation choice affects the scale of H.
            H -= prob * (prob ** 0) # Placeholder for log logic below
            # Corrected line:
            import math
            H -= prob * math.log(prob)
            
    # 4. Compute Mean Gradient (G)
    gradient_sum = 0.0
    # Gradients are computed on (m-1) x (n-1) grid
    for i in range(m - 1):
        for j in range(n - 1):
            dx = image[i+1][j] - image[i][j]
            dy = image[i][j+1] - image[i][j]
            gradient_sum += (dx**2 + dy**2)**0.5
            
    G = gradient_sum / ((m - 1) * (n - 1))
    
    return {
        'H': H,
        'G': G,
        'U': U,
        'Std': Std
    }
```

## Subtopics

### [Multi-Scale Entropy for View Quality](17037-multi-scale-entropy-for-view-quality.md)
Multi-Scale Entropy for View Quality is a reference-free metric that quantifies the visual information content of a rendered volume image by computing the Shannon entropy of wavelet coefficients across multiple resolution levels. It addresses the limitation of standard histogram-based entropy, which ignores spatial pixel correlations, by capturing structural details and local correlations inherent in volume rendering. Unlike its sibling [Image Entropy (H)](15295-reference-free-image-quality-metrics.md), which relies solely on pixel value histograms, this method uses wavelet transforms to measure information at different scales, making it more sensitive to the structural complexity of the rendered view.

### Equivalent Number of Look (ENL)
ENL is a metric measuring the smoothness in homogeneous areas of a reconstruction [S3]. It is defined as:
$$ ENL = \frac{1}{N_r} \sum_{r=1}^{N_r} \frac{l_r^2}{r^2_r} $$
where $N_r$ is the number of regions selected to compute the ENL, and $l_r$ and $r^2_r$ refer to the mean and variance in the selected homogeneous regions, respectively [S3].

## Operating characteristics
*   **Complexity:** The computation is $O(m \cdot n)$ for all four metrics, as it requires a single pass (or two for Std) over the image pixels. Histogram computation is $O(m \cdot n)$ with $O(L)$ space. Gradient computation is $O(m \cdot n)$.
*   **Performance:** The sources do not provide specific timing measurements. However, the simplicity of the operations (sums, squares, square roots) makes it computationally inexpensive compared to structural similarity or deep learning-based metrics.
*   **Strengths:** No reference image required; effective for evaluating occlusion reduction and microstructure clarity in specific visualization contexts like explosion fields [S1, S2].
*   **Limitations:** Interpretation of $U$ and $Std$ is context-dependent (smaller is better for occlusion reduction, larger for general richness) [S1]. It does not account for spatial correlations as effectively as multi-scale methods [S1].
*   **Scaling:** Linear with image resolution.

## Background
The method originated in the context of evaluating volume rendering for explosion field data, where obtaining an ideal reference image is difficult [S1, S2]. It is also referred to as the "first evaluation method" [S1, S2]. The specific application in the sources involves analyzing shock waves and jets, where reducing visual chaos (lower $U$ and $Std$) is prioritized [S1]. The sources cite increases in Mean Gradient $G$ as indicators of improved microstructure visibility (e.g., Group C $G$ increased from 1.25 to 1.40, Group D from 0.37 to 0.53) [S1]. In explosion field visualization, the new method reduced Image Mean by up to 28.38% and Standard Deviation by 34.88% at the initial moment, indicating reduced occlusion [S1].

## Sources
[S1] A volume rendering visualization method oriented to explosion field features, https://doi.org/10.1063/5.0226283, fetched 2026-10-26
[S2] A Volume Rendering Visualization Method Oriented to Explosion Field ..., https://www.researchsquare.com/article/rs-4492344/v1, fetched 2026-10-29
[S3] Adaptive differentiable grids for cryo-electron tomography reconstruction and denoising, https://doi.org/10.1093/bioadv/vbad131, fetched 2026-10-27
