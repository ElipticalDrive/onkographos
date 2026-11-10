# Curvature-based Transfer Function

Also called: Curvature Based TF, Non-photorealistic Curvature TF, Shape Based TF (Curvature), cbTF

## Overview
The Curvature-based Transfer Function (cbTF) is a data feature-based classification method that maps the raw data value and the local curvature of the volume to opacity and color [S1][S3, unofficial]. It solves the problem of distinguishing structural boundaries and emphasizing geometric contours, particularly for non-photorealistic rendering (NPR) where ridge, valley, or bumpiness visualization is desired [S1][S2, unofficial]. Unlike gradient-based methods that highlight intensity changes, cbTF highlights the shape of isosurfaces, allowing it to distinguish features of different sizes that share similar intensity gradients [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A scalar volume dataset $V$, a camera position $\mathbf{p}_{cam}$, and a set of isosurfaces defined by threshold values [S1][S2, unofficial].
*   **Output:** Opacity $\alpha \in [0, 1]$ and color $\mathbf{c}$ for each voxel, typically via a 2D look-up table (LUT) indexed by data value and curvature [S1].

**Data Structures:**
*   **Curvature Volume:** A 3D volume storing the computed curvature metric for each voxel. The sources specify that curvature computation requires a neighborhood of at least 26 points (the 3x3x3 cube excluding the center) [S1].
*   **Transfer Function LUT:** A 2D texture or table where the axes are Data Value and Curvature [S1].

**Method Steps:**
1.  **Smoothing (Optional but Recommended):** To combat spurious curvature variations caused by noise (common in MRI data), the input volume is smoothed [S2, unofficial]. This step sacrifices accuracy and resolution [S2, unofficial].
2.  **Curvature Computation:** Compute the curvature $K$ for each voxel.
    *   The curvature is derived from the second derivatives of the intensity function [S4, unofficial].
    *   It is commonly calculated using the Hessian matrix $H$ of the volume data [S4, unofficial].
    *   The principal curvatures are the maximum ($k_1$) and minimum ($k_2$) eigenvalues of the Hessian [S4, unofficial].
    *   Specific curvature metrics can be derived from these, such as Mean Curvature or Gaussian Curvature [S4, unofficial].
    *   Alternatively, the Laplacian can approximate the second derivative, though it is less precise than the Hessian approach [S4, unofficial].
3.  **Feature Vector Construction:** Construct the feature vector $\mathbf{f}_i = [V_i, K_i]$ for each voxel $i$, where $V_i$ is the data value and $K_i$ is the computed curvature [S1].
4.  **Classification:** Map $\mathbf{f}_i$ to optical properties using the 2D cbTF [S1].
    *   If camera position is included, the mapping becomes $C(V_i, K_i, \mathbf{p}_{cam})$ to generate contour effects relative to the view [S1].
    *   The LUT is queried at $(V_i, K_i)$ to retrieve $\alpha$ and $\mathbf{c}$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_curvature_hessian(volume):
    """
    Computes curvature using the Hessian matrix approach.
    Requires a 26-point neighborhood (3x3x3).
    """
    # Initialize curvature volume
    curvature_vol = zeros_like(volume)
    
    # Compute second derivatives (Hessian components)
    # In practice, this involves convolution with derivative kernels
    # H_xx, H_xy, H_xz, H_yy, H_yz, H_zz
    H_xx = convolve(volume, kernel_d2x2)
    H_yy = convolve(volume, kernel_d2y2)
    H_zz = convolve(volume, kernel_d2z2)
    H_xy = convolve(volume, kernel_d2xy)
    H_xz = convolve(volume, kernel_d2xz)
    H_yz = convolve(volume, kernel_d2yz)
    
    # For each voxel, compute eigenvalues of the Hessian
    for voxel in volume:
        # Construct 3x3 Hessian matrix H
        H = [
            [H_xx[voxel], H_xy[voxel], H_xz[voxel]],
            [H_xy[voxel], H_yy[voxel], H_yz[voxel]],
            [H_xz[voxel], H_yz[voxel], H_zz[voxel]]
        ]
        
        # Compute eigenvalues k1, k2, k3
        eigenvalues = compute_eigenvalues(H)
        
        # Sort eigenvalues such that |k1| >= |k2| >= |k3|
        k1, k2, k3 = sort_abs(eigenvalues)
        
        # Compute curvature metric (e.g., Mean Curvature)
        # Mean Curvature H_mean = (k1 + k2 + k3) / 2
        # Or Gaussian Curvature K_gauss = k1 * k2 * k3
        # The specific metric depends on the TF design [S4, unofficial]
        curvature_vol[voxel] = mean_curvature(k1, k2, k3)
        
    return curvature_vol

def apply_cbtf(volume, curvature_vol, tf_lut, camera_position=None):
    """
    Applies the Curvature-based Transfer Function.
    """
    result_color = zeros_like(volume) # Assuming RGB output
    result_opacity = zeros_like(volume)
    
    for voxel in volume:
        v_val = volume[voxel]
        c_val = curvature_vol[voxel]
        
        # Query 2D LUT
        # If camera position is used, the LUT might be 3D or the lookup 
        # depends on the angle between the surface normal and camera vector.
        # Standard cbTF is 2D: (Data Value, Curvature) [S1]
        
        if camera_position is not None:
            # Advanced NPR: adjust color/opacity based on view-dependent contours
            # This requires surface normal estimation, which is not explicitly 
            # detailed in the core cbTF definition but implied by "contour around features" [S1]
            # [G] Estimate normal from gradient of volume for view-dependent shading
            grad = compute_gradient(volume, voxel)
            normal = normalize(grad)
            view_dir = normalize(camera_position - voxel)
            # Adjust TF lookup based on dot product or similar view factor
            # Note: Exact formula for view-dependent cbTF is not provided in sources
            # [G] Assume standard 2D lookup for base opacity/color
            pass
            
        alpha, color = tf_lut.lookup(v_val, c_val)
        
        result_opacity[voxel] = alpha
        result_color[voxel] = color
        
    return result_color, result_opacity
```

**Parameters:**
*   **Smoothing Kernel:** Used to reduce noise before curvature calculation. Trade-off: reduces spurious variations but sacrifices resolution [S2, unofficial].
*   **Curvature Metric:** Choice between Mean, Gaussian, or Principal curvatures [S4, unofficial].
*   **Isosurface Thresholds:** Used to define the structures of interest [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Noise Amplification:** Curvature is a second derivative, making it highly sensitive to noise. Spurious variations occur in noisy datasets (e.g., MRI) [S1][S2, unofficial].
*   **Ambiguity:** Many different features can share the same curvature value, making it difficult to emphasize a specific target feature without unintended emphasis on others [S1].
*   **Resolution Loss:** Smoothing required to mitigate noise reduces the effective resolution of the volume [S2, unofficial].

## Subtopics

### Curvature
A specific metric computed via signal convolution that describes local shape contours and measures deviation from a straight aspect [S4, unofficial]. It is derived from the maximum ($k_1$) and minimum ($k_2$) principal curvatures, which are eigenvalues of the Hessian matrix [S4, unofficial]. Common variants include Mean Curvature and Gaussian Curvature [S4, unofficial]. It is primarily used for non-photorealistic visualizations to highlight contours [S4, unofficial].

## Operating characteristics
*   **Time Complexity:** Real-time capability is achievable on GPU [S1].
*   **Memory/Performance:** Requires a minimum of 27 3D texture lookups per sample for the real-time GPU variant (1 for data, 26 for the neighborhood/curvature calculation context) [S1].
*   **Strengths:** Effective for non-photorealistic rendering, highlighting contours, ridges, and valleys [S1][S2, unofficial]. Can distinguish features of different sizes based on shape [S1].
*   **Limitations:** High sensitivity to noise; requires smoothing which degrades resolution; lacks specificity for individual features due to curvature ambiguity [S1][S2, unofficial].
*   **When to use:** When the goal is to visualize the geometric shape, contours, or bumpiness of structures, especially in NPR contexts [S1][S2, unofficial].
*   **When not to use:** When precise segmentation of specific internal features is required without shape bias, or in very noisy data where smoothing destroys detail [S1][S2, unofficial].

## Background
The curvature-based transfer function was introduced by Kindlmann et al. [KWTM03] as a method to use curvature as a second dimension in the transfer function domain [S1][S3, unofficial]. It allows for non-photorealistic renderings that highlight volume contours [S3, unofficial]. Real-time GPU implementations were later detailed in *GPU Gems 2* [PF05] and [HSS∗05] for iso-surfaces [S1]. The approach is part of the broader family of data feature-based classification methods that extend beyond simple intensity mapping [S1].

## Sources
[S1] A Survey of Transfer Functions Suitable for Volume Rendering, https://doi.org/10.2312/vg/vg10/077-083, fetched 2026-10-26
[S2] Context-Preserving Volume Image Exploration Using A 3D Painting Metaphor, https://doi.org/10.32920/ryerson.14646888, fetched 2026-10-26 (unofficial)
[S3] MULTIVARIATE TRANSFER FUNCTION DESIGN, https://www.sci.utah.edu/~lzhou/dissertation_final_zhoul.pdf, fetched 2026-10-27 (unofficial)
[S4] A volume exploration approach based on clustering and dimensional reduction to support the definition of multidimensional transfer functions, http://hdl.handle.net/11449/153618, fetched 2026-10-26 (unofficial)
