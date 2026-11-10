# Curvature-based Transfer Function

Also called: Curvature Based TF, Non-photorealistic Curvature TF, Shape Based TF (Curvature), cbTF

## Overview
The Curvature-based Transfer Function (cbTF) is a data feature-based classification method that uses the second derivative (curvature) of the volume data, in addition to the scalar value, to distinguish structural transitions and emphasize geometric contours [S1][S3, unofficial]. It is primarily used for non-photorealistic rendering to highlight ridges, valleys, and surface bumpiness, or to distinguish features of different sizes that share similar intensity values [S1][S2, S3, unofficial]. Unlike gradient-based methods which detect edges, cbTF detects the *shape* of the edge (convex vs. concave), allowing it to separate the borders of small and large features that have identical intensity gradients [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A scalar volume dataset $V$ and a camera position $\mathbf{c}$.
*   **Output:** Opacity $\alpha \in [0, 1]$ and color $\mathbf{C} = (r, g, b)$ for each sample point.

**Data Structures:**
The classification function is typically implemented as a 2D Lookup Table (LUT) [S1]. The axes of the LUT are:
1.  **Data Value:** The scalar intensity $v$ at the sample point.
2.  **Curvature:** The computed curvature metric $\kappa$ at the sample point.
*Note: Some implementations may incorporate camera position as a third dimension or use it to modulate the output, but the core cbTF domain is 2D [S1].*

**Curvature Computation:**
Curvature is derived from the local geometry of the isosurfaces. It is commonly calculated using the Hessian matrix $H$ of the scalar field or approximated via the Laplacian [S4, unofficial].
1.  Compute the Hessian matrix $H(\mathbf{x})$ at each voxel $\mathbf{x}$, which contains the second partial derivatives:
    $$
    H(\mathbf{x}) = \begin{bmatrix}
    f_{xx} & f_{xy} & f_{xz} \\
    f_{yx} & f_{yy} & f_{yz} \\
    f_{zx} & f_{zy} & f_{zz}
    \end{bmatrix}
    $$
2.  Compute the eigenvalues $\lambda_1, \lambda_2, \lambda_3$ of $H(\mathbf{x})$.
3.  Derive curvature metrics from the eigenvalues. Common metrics include:
    *   **Mean Curvature:** $H_{mean} = \frac{\lambda_1 + \lambda_2 + \lambda_3}{3}$
    *   **Gaussian Curvature:** $K = \lambda_1 \lambda_2 \lambda_3$
    *   Alternatively, maximum curvature $k_1$ and minimum curvature $k_2$ can be used [S4, unofficial].
*   **Neighborhood:** The computation of curvature requires a neighborhood of at least 26 points (the 3x3x3 cube excluding the center) to estimate the second derivatives accurately [S1].

**Classification Steps:**
1.  **Preprocessing (Optional):** To combat spurious curvature variations caused by noise (common in MRI data), the data values may be smoothed using Gaussian filters [S2, unofficial]. This sacrifices resolution and accuracy [S2, unofficial].
2.  **Feature Extraction:** For each sample point $\mathbf{x}$ along the ray:
    a. Retrieve the scalar value $v = V(\mathbf{x})$.
    b. Compute the curvature $\kappa$ at $\mathbf{x}$ using the Hessian/Laplacian method described above.
3.  **Lookup:** Query the 2D LUT using coordinates $(v, \kappa)$ to retrieve opacity $\alpha$ and color $\mathbf{C}$ [S1].
4.  **Rendering:** Composite the sample using standard volume rendering integration (e.g., front-to-back compositing) [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_curvature_hessian(volume, x, y, z):
    """
    Computes curvature metrics using the Hessian matrix at voxel (x,y,z).
    Requires a 3x3x3 neighborhood.
    """
    # Extract 3x3x3 neighborhood values
    # Note: Boundary checks required in implementation [G]
    neighborhood = get_3x3x3_patch(volume, x, y, z)
    
    # Compute second derivatives (Hessian elements)
    # Using central differences or convolution kernels on the neighborhood
    f_xx = compute_second_derivative(neighborhood, axis=0)
    f_yy = compute_second_derivative(neighborhood, axis=1)
    f_zz = compute_second_derivative(neighborhood, axis=2)
    f_xy = compute_cross_derivative(neighborhood, axis0=0, axis1=1)
    f_xz = compute_cross_derivative(neighborhood, axis0=0, axis1=2)
    f_yz = compute_cross_derivative(neighborhood, axis0=1, axis1=2)
    
    # Construct Hessian Matrix
    H = [
        [f_xx, f_xy, f_xz],
        [f_xy, f_yy, f_yz],
        [f_xz, f_yz, f_zz]
    ]
    
    # Compute eigenvalues of H
    eigenvalues = compute_eigenvalues(H)
    lambda1, lambda2, lambda3 = sorted(eigenvalues, reverse=True)
    
    # Return Mean and Gaussian Curvature
    mean_curvature = (lambda1 + lambda2 + lambda3) / 3.0
    gaussian_curvature = lambda1 * lambda2 * lambda3
    
    return mean_curvature, gaussian_curvature

def classify_cbtf(sample_x, volume, tf_lut_2d):
    """
    Classifies a sample point using Curvature-based Transfer Function.
    """
    # 1. Get scalar value
    v = volume[sample_x]
    
    # 2. Compute curvature
    # Requires precomputed curvature volume or on-the-fly computation
    # On-the-fly requires 26-point neighborhood [S1]
    k_mean, k_gauss = compute_curvature_hessian(volume, sample_x.x, sample_x.y, sample_x.z)
    
    # 3. Map to optical properties
    # Using Mean Curvature as the primary second dimension for LUT indexing
    alpha, color = tf_lut_2d.lookup(v, k_mean)
    
    return alpha, color
```

**Parameters:**
*   **Smoothing Kernel:** If used, the sigma of the Gaussian filter for noise reduction [S2, unofficial].
*   **Curvature Metric:** Choice between Mean, Gaussian, or Principal curvatures [S4, unofficial].

**Edge Cases and Pitfalls:**
*   **Noise Amplification:** Curvature is a second derivative and is highly sensitive to noise. In noisy datasets (e.g., MRI), spurious surface curvature variations occur, leading to incorrect classification [S1][S2, unofficial].
*   **Ambiguity:** Many different features share the same curvature value. It is difficult to emphasize a targeted feature without inadvertently emphasizing other features with similar curvature [S1].
*   **Resolution Loss:** Smoothing data to reduce noise sacrifices accuracy and resolution [S2, unofficial].

## Subtopics

### Curvature
A specific metric computed by signal convolution that describes local shape contours and measures how much a surface deviates from a straight aspect [S4, unofficial]. It is derived from the maximum curvature $k_1$ and minimum curvature $k_2$ [S4, unofficial]. Different types include mean curvature and Gaussian curvature [S4, unofficial]. It is commonly used for non-photorealistic visualizations [S4, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
*   **Memory:** Requires a 2D LUT for the transfer function [S1].
*   **Time:** Real-time capability is achievable on GPU [S1]. However, it is computationally expensive due to the need for second derivative estimation. A real-time GPU variant requires a minimum of 27 3D texture lookups per sample (to gather the 3x3x3 neighborhood for Hessian computation) [S1]. This is significantly higher than 1D TFs (1 lookup) or Gradient-based TFs (7 lookups) [S1].

**Strengths and Limitations:**
*   **Strengths:** Can distinguish between borders of features with different sizes (curvatures) even if they have the same intensity [S1]. Effective for non-photorealistic contouring and ridge/valley visualization [S1][S2, S3, unofficial].
*   **Limitations:** Highly sensitive to noise [S1][S2, unofficial]. Computationally intensive due to neighborhood requirements [S1]. Difficult to isolate specific features due to curvature ambiguity [S1].

**When to Use:**
*   Use when the goal is non-photorealistic rendering emphasizing surface shape, contours, or bumpiness [S1][S2, S3, unofficial].
*   Use when distinguishing features based on their geometric size/shape rather than intensity [S1].

**When Not to Use:**
*   Do not use for noisy datasets without significant preprocessing/smoothing, as noise amplifies curvature errors [S1][S2, unofficial].
*   Do not use if real-time performance is critical and hardware cannot support the high number of texture lookups required for Hessian estimation [S1].

## Background
The Curvature-based Transfer Function was introduced by Kindlmann et al. in 2003 [KWTM03], who used curvature as a second dimension in their transfer function domain to allow non-photorealistic renderings that highlight volume contours [S3, unofficial]. It is described in reference [16] of [S2, unofficial] and compared with other TF types in [S1]. Real-time GPU variants were later published in GPU Gems 2 [PF05] and [HSS∗05] for iso-surfaces [S1].

## Sources
[S1] A Survey of Transfer Functions Suitable for Volume Rendering, https://doi.org/10.2312/vg/vg10/077-083, fetched 2026-10-26
[S2] Context-Preserving Volume Image Exploration Using A 3D Painting Metaphor, https://doi.org/10.32920/ryerson.14646888, fetched 2026-10-26 (unofficial)
[S3] MULTIVARIATE TRANSFER FUNCTION DESIGN, https://www.sci.utah.edu/~lzhou/dissertation_final_zhoul.pdf, fetched 2026-10-27 (unofficial)
[S4] A volume exploration approach based on clustering and dimensional reduction to support the definition of multidimensional transfer functions, http://hdl.handle.net/11449/153618, fetched 2026-10-26 (unofficial)
