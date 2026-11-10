# Explicit Tone Mapping

Also called: Camera response modeling, HDR to LDR conversion

## Overview
Explicit Tone Mapping converts a High Dynamic Range (HDR) radiance image $I_h$ into a Low Dynamic Range (LDR) display image $I_l$ by modeling the physical image acquisition process of a digital camera [S1]. It differs from implicit or learned tone mapping by explicitly parameterizing the white balance and Camera Response Function (CRF) as distinct, differentiable stages, allowing for self-calibration during the optimization of HDR radiance fields [S1].

## How it works
The method implements a tone mapping function $T$ parameterized by radiometric parameters $\theta$, such that $I_l = T(I_h, \theta)$. The process follows the sequential stages of camera image acquisition: white balance calibration followed by non-linear response [S1].

### Inputs and Outputs
*   **Input**: HDR pixel color $C_h(\mathbf{r}) \in \mathbb{R}^3$ for a specific ray $\mathbf{r}$, obtained by ray-marching HDR radiance fields.
*   **Output**: LDR pixel color $I_l \in \mathbb{R}^3$.
*   **Parameters**:
    *   White balance scale $\theta_w = [w_r, w_g, w_b]^\top \in \mathbb{R}^3$. Regularized to be positive [S1].
    *   CRF control points $\theta_g$ (256 values) [S1].
    *   Leaky thresholding coefficient $\alpha$ [S1].

### Data Structures
*   **CRF Grid**: A 1D grid of 256 control points representing the CRF values at uniformly sampled points in the domain $[0, 1]$. The grid is used for differentiable 1D grid-sampling [S1].
*   **Boundary Conditions**: The CRF is constrained such that $g(0) = 0$ and $g(1) = 1$ [S1].

### Steps

1.  **White Balance Calibration**:
    Apply the white balance function $w$ to the HDR pixel color $C_h(\mathbf{r})$ using element-wise multiplication with $\theta_w$ [S1]:
    $$
    I_w = w(C_h(\mathbf{r}), \theta_w) = C_h(\mathbf{r}) \odot \theta_w = \begin{bmatrix} w_r c_h^r \\ w_g c_h^g \\ w_b c_h^b \end{bmatrix}
    $$
    where $C_h(\mathbf{r}) = [c_h^r, c_h^g, c_h^b]^\top$ [S1]. The parameters $\theta_w$ are regularized to ensure positive values [S1].

2.  **Camera Response Function (CRF) Application**:
    Map the white-balanced image $I_w$ to the LDR image $I_l$ using the CRF $g$. The function $g$ is parameterized as a discrete piece-wise linear function divided into 256 intervals over $[0, 1]$, defined by 256 control points [S1].
    
    To ensure differentiability, the pixel value of $I_w$ is mapped to $I_l$ by interpolating the CRF values of nearby control points using 1D grid-sampling [S1]. The range of $I_w$ beyond $[0, 1]$ is thresholded [S1].

3.  **Leaky Thresholding for Saturation**:
    To propagate loss in saturation regions during training and prevent bias from missing texture cues in over/undersaturated areas, a leaky thresholding function $g_{leaky}$ is applied [S1]:
    $$
    g_{leaky}(x) = \begin{cases} 
    \alpha x, & x < 0 \\
    g(x), & 0 \leq x \leq 1 \\
    -\frac{\alpha}{\sqrt{x}} + \alpha + 1, & x > 1 
    \end{cases}
    $$
    where $\alpha$ is the thresholding coefficient [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def explicit_tone_mapping(hdr_pixel, white_balance_params, crf_grid, alpha):
    """
    Converts an HDR pixel to LDR using explicit white balance and CRF.
    
    Args:
        hdr_pixel (tuple): (r, g, b) values from HDR radiance field.
        white_balance_params (tuple): (wr, wg, wb) positive scale factors.
        crf_grid (list): 256 control points for the CRF, indexed 0..255.
        alpha (float): Thresholding coefficient for leaky saturation.
        
    Returns:
        tuple: (r, g, b) LDR values clamped to [0, 1].
    """
    # Step 1: White Balance
    # Element-wise product of HDR color and white balance parameters
    # Regularization of white_balance_params to be positive is assumed 
    # to be handled by the optimizer constraints, not this function.
    wb_r = hdr_pixel[0] * white_balance_params[0]
    wb_g = hdr_pixel[1] * white_balance_params[1]
    wb_b = hdr_pixel[2] * white_balance_params[2]
    
    # Step 2 & 3: CRF with Leaky Thresholding
    # Apply g_leaky to each channel independently
    ldr_r = apply_leaky_crf(wb_r, crf_grid, alpha)
    ldr_g = apply_leaky_crf(wb_g, crf_grid, alpha)
    ldr_b = apply_leaky_crf(wb_b, crf_grid, alpha)
    
    # Clamp final output to [0, 1] as per LDR display encoding requirements
    return (
        max(0.0, min(1.0, ldr_r)),
        max(0.0, min(1.0, ldr_g)),
        max(0.0, min(1.0, ldr_b))
    )

def apply_leaky_crf(value, crf_grid, alpha):
    """
    Applies the explicit CRF with leaky thresholding to a single channel value.
    
    Args:
        value (float): The input pixel value (potentially outside [0, 1]).
        crf_grid (list): 256 control points for the CRF.
        alpha (float): Thresholding coefficient.
        
    Returns:
        float: The tone-mapped value.
    """
    # Step 3: Leaky Thresholding logic
    if value < 0:
        return alpha * value
    elif value > 1:
        # Leaky threshold for x > 1
        # Formula: -alpha / sqrt(x) + alpha + 1
        return -alpha / (value ** 0.5) + alpha + 1
    else:
        # Step 2: CRF interpolation for 0 <= x <= 1
        # The CRF is defined by 256 control points for uniformly sampled points in [0, 1].
        # Index calculation: map value in [0, 1] to index in [0, 255].
        # Uniform sampling implies index = value * (num_points - 1)
        num_points = 256
        index_float = value * (num_points - 1)
        
        # Get lower and upper indices for interpolation
        idx_low = int(index_float)
        idx_high = idx_low + 1
        
        # Handle boundary case where index_float is exactly 255.0
        if idx_high >= num_points:
            idx_high = num_points - 1
            idx_low = num_points - 1
            
        # Interpolation weight
        t = index_float - idx_low
        
        # Get control point values
        val_low = crf_grid[idx_low]
        val_high = crf_grid[idx_high]
        
        # Linear interpolation
        return val_low + t * (val_high - val_low)
```

### Edge Cases and Pitfalls
*   **Saturation**: In regions with wide dynamic range, LDR images may contain over- and undersaturation where no geometric or photometric cues exist. The leaky thresholding method is required to propagate loss in these regions during training [S1].
*   **Boundary Conditions**: The CRF must satisfy $g(0)=0$ and $g(1)=1$ to adhere to standard camera response models [S1].
*   **White Balance Regularization**: The white balance parameters $\theta_w$ must be regularized to be positive to maintain physical propriety [S1].

## Operating characteristics
*   **Complexity**: The operation is $O(1)$ per pixel, involving a constant number of multiplications and a single linear interpolation over a fixed-size grid (256 points).
*   **Differentiability**: The use of 1D grid-sampling and piece-wise linear interpolation makes the CRF differentiable with respect to the control points, enabling gradient-based optimization of the radiance field and tone mapping parameters simultaneously [S1].
*   **Strengths**: Explicitly models physical camera properties (white balance, CRF), allowing for self-calibration of HDR radiance fields from uncalibrated LDR inputs [S1].
*   **Limitations**: Requires optimization of additional parameters ($\theta_w, \theta_g$) alongside the radiance field, which may increase training complexity. Saturation regions remain problematic for geometry recovery despite leaky thresholding [S1].

## Background
Explicit Tone Mapping was introduced in the context of HDR-Plenoxels, a method for self-calibrating high dynamic range radiance fields [S1]. It addresses the challenge of recovering HDR radiance fields from LDR images by treating tone mapping as a learnable, differentiable module rather than a fixed post-processing step [S1].

## Sources
[S1] HDR-Plenoxels: Self-Calibrating High Dynamic Range Radiance Fields, https://doi.org/10.1007/978-3-031-19824-3_23, fetched 2026-10-26
[S2] aivv73/bend-voxel, https://github.com/aivv73/bend-voxel, fetched 2026-10-28 (unofficial)
