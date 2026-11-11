# Multivariate Transfer Function Design based on User-Selected Samples

Also called: Interactive TF design with KDE, KDE-based Multivariate TF from Spatial Samples, Probing-based TF workflow, Sample-based HDTF generation, User-Selected Sample Based TF Design, User-Selected Sample TF Design

## Overview
This method automates the creation of high-dimensional transfer functions (HDTFs) and 2D Gaussian transfer functions from user-selected spatial samples in multivariate volume datasets [S1, unofficial]. It solves the problem of cumbersome manual parameter tuning in high-dimensional spaces by allowing users to probe features in the spatial domain (via slices) and automatically generating robust TFs using Kernel Density Estimation (KDE) [S1, unofficial]. Unlike direct manipulation widgets that require abstract parameter adjustment, this approach bridges the spatial and TF domains by deriving TFs directly from queried voxel data, discarding outliers robustly [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** Multivariate volume dataset (scalar and/or vector fields); user-selected spatial regions (via lasso or magic wand) on slice views [S1, unofficial].
*   **Output:** High-dimensional transfer functions (HDTFs) represented in Parallel Coordinate Plots (PCPs); 2D Gaussian TFs; rendered volume with directional occlusion shading [S1, unofficial].

**Data Structures**
*   **Spatial Samples:** A set of $N$ multivariate data vectors $\mathbf{v}_i = (v_{i,1}, v_{i,2}, \dots, v_{i,D})$ queried from voxels within the user-selected region, where $D$ is the number of data dimensions [S1, unofficial].
*   **HDTF Representation:** Parallel Coordinate Plots (PCPs) and pairwise scatter plots [S1, unofficial].
*   **2D Gaussian TF:** A Gaussian Mixture Model (GMM) in a dimensionality-reduced 2D space [S1, unofficial].

**Steps**
1.  **Data Probing:** The user examines multivariate data slices in a multipanel view [S1, unofficial].
2.  **Region Selection:** The user selects regions of interest (ROIs) using a lasso tool or a "magic wand" stroke [S1, unofficial].
3.  **Voxel Query:** A voxel query operation retrieves the multivariate data values for all voxels within the selected region, forming the sample set $\{\mathbf{v}_i\}_{i=1}^N$ [S1, unofficial].
4.  **HDTF Generation (KDE):** Kernel Density Estimation (KDE) is applied to the sample set to automatically generate HDTFs [S1, unofficial].
    *   The KDE process robustly discards outliers from the queried samples [S1, unofficial].
    *   The resulting density function defines the transfer function in the high-dimensional space [S1, unofficial].
5.  **2D Gaussian TF Generation (Optional):** Alternatively, a Gaussian Mixture Model (GMM) is applied to the samples to generate 2D Gaussian TFs in a dimensionality-reduced view [S1, unofficial].
6.  **Visualization:** The extracted features are rendered in the volume rendering view using directional occlusion shading to overcome artifacts from Phong shading in the multivariate case [S1, unofficial].
7.  **Refinement (Optional):** If features share similar data value ranges, the user applies direct volume selection tools (segmentation brushes or lassos) on the volume rendering view or panel view to refine the classification [S1, unofficial].
8.  **Iteration:** The user iterates through stages A-C (Probing, Analysis, Refinement) until satisfied, then outputs the labeled volume [S1, unofficial].

**Formulas**
*   **KDE for HDTF:** The transfer function $T(\mathbf{x})$ at a point $\mathbf{x}$ in the data space is estimated from the samples $\{\mathbf{v}_i\}$ using a kernel $K$ and bandwidth $h$:
    $$ T(\mathbf{x}) = \frac{1}{N h^D} \sum_{i=1}^{N} K\left(\frac{\mathbf{x} - \mathbf{v}_i}{h}\right) $$
    *   $N$: Number of queried samples [S1, unofficial].
    *   $D$: Number of data dimensions [S1, unofficial].
    *   $h$: Bandwidth parameter (Not given in the sources) [S1, unofficial].
    *   $K$: Kernel function (typically Gaussian, though not explicitly defined in the text) [S1, unofficial].
    *   The method uses KDE to "robustly discard outliers," implying a thresholding or trimming step on the density estimate or sample weights, but the exact algorithm for outlier removal is not specified [S1, unofficial].

*   **2D Gaussian TF:** Generated via a Gaussian Mixture Model (GMM) in a dimensionality-reduced space [S1, unofficial]. The specific reduction technique (e.g., PCA) and GMM parameters (number of components, covariance constraints) are not given in the sources [S1, unofficial].

**Parameters**
*   **KDE Bandwidth:** Not given in the sources [S1, unofficial].
*   **GMM Components:** Not given in the sources [S1, unofficial].
*   **Shading:** Directional occlusion shading is used instead of Phong shading [S1, unofficial].

**Edge Cases and Pitfalls**
*   **Phong Shading Artifacts:** In multivariate cases, standard Phong shading produces artifacts; directional occlusion shading is required [S1, unofficial].
*   **Similar Data Ranges:** Features with similar data value ranges may be difficult to separate; direct volume selection tools are needed for refinement [S1, unofficial].
*   **Outliers:** KDE is used to robustly discard outliers, but the mechanism for this robustness is not detailed [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def design_tf_multivariate(volume_data, user_selection_tool):
    # Stage A: Data Probing
    selected_indices = user_selection_tool(volume_data)
    if len(selected_indices) == 0:
        return None, None, None
    samples = volume_data[selected_indices]
    N, D = samples.shape

    # Stage B: Qualitative Analysis
    # HDTF Generation via KDE
    def compute_hdtf_density(x, samples, h):
        density = 0.0
        for v in samples:
            diff = [x[j] - v[j] for j in range(D)]
            dist_sq = sum(d**2 for d in diff)
            density += math.exp(-dist_sq / (2 * h**2))
        return density / N

    # 2D Gaussian TF via GMM on dimensionality-reduced space
    # Dimensionality reduction technique and GMM fitting algorithm are not specified in the sources [S1, unofficial]
    def compute_gaussian_tf_params(samples):
        # Standard GMM initialization and fitting [G]
        return {"n_components": 2, "weights": None, "means": None, "covariances": None}

    hdtf_func = lambda x: compute_hdtf_density(x, samples, h=1.0)
    gaussian_tf = compute_gaussian_tf_params(samples)

    # Render features using directional occlusion shading to overcome Phong shading artifacts [S1, unofficial]
    rendered_volume = volume_data # Rendering step handled by the visualization engine

    # Stage C: Optional Refinement
    # Direct volume selection tools (segmentation brushes or lassos) are applied iteratively by the user [S1, unofficial]
    return hdtf_func, gaussian_tf, rendered_volume
```

## Subtopics

### [Design Gallery Interface](9489-design-gallery-interface.md)
The Design Gallery Interface is a user interaction method for volume rendering that presents a gallery of predefined transfer functions, allowing users to select features of interest from the gallery and refine them. It sets itself apart from direct manipulation widgets by abstracting the transfer function space into a visual gallery of rendered thumbnails, reducing the need for users to understand the underlying parameter space.

## Operating characteristics
*   **Time Complexity:** Not given in the sources [S1, unofficial].
*   **Memory Complexity:** Not given in the sources [S1, unofficial].
*   **Measured Performance:** Not given in the sources [S1, unofficial].
*   **Strengths:** Automates HDTF generation from spatial samples; robustly discards outliers; intuitive for domain users who find manual PCP editing cumbersome; handles multivariate data better than scalar-only methods [S1, unofficial].
*   **Limitations:** Requires user interaction to select initial samples; refinement may be needed for features with similar data ranges; relies on KDE and GMM which may have parameter sensitivity (bandwidth, components) not specified in the sources [S1, unofficial].
*   **When to Use:** When exploring complex multivariate volume datasets (e.g., geophysics, seismic surveys) where manual TF design is difficult [S1, unofficial].
*   **When Not to Use:** When real-time performance is critical and KDE/GMM computation is too slow (not specified, but implied by the interactive nature); when the user prefers direct parameter manipulation over sample-based generation [S1, unofficial].

## Background
*   **Origin:** Proposed by Zhou et al. in a dissertation titled "Multivariate Transfer Function Design" [S1, unofficial].
*   **History:** The method addresses the limitations of existing state-of-the-art methods like QVAPOR, which require users to explore volumes using parameter views (PCPs, histograms) that can be cumbersome for domain users [S1, unofficial]. The work was developed in collaboration with geophysicists who requested more automated methods for feature extraction [S1, unofficial].
*   **Usage:** Successfully applied for simulation and complicated seismic datasets [S1, unofficial].

## Sources
[S1] MULTIVARIATE TRANSFER FUNCTION DESIGN, https://www.sci.utah.edu/~lzhou/dissertation_final_zhoul.pdf, fetched 2026-10-27 (unofficial)
