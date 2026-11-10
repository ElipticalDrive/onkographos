# 1D Lookup Table with Interpolation

Also called: HU -> mu mapping, Piecewise Linear HU to Attenuation Transfer Function, Piecewise Linear Transfer Function

## Overview
This method maps scalar voxel intensities, typically Hounsfield Units (HU) in medical imaging, to optical properties such as attenuation coefficients or opacity using a piecewise linear function defined by user-specified keypoints [S1, unofficial]. It solves the problem of visualizing specific tissue types by assigning distinct visual weights to different density ranges [S1, unofficial]. Unlike [Binary and Soft Classification](6229-binary-and-soft-classification.md) which uses hard thresholds or complex basis functions, this approach provides a computationally efficient, linear interpolation between defined material boundaries, serving as the foundational implementation for [Piecewise Linear Interpolation (Lookup Table)](2643-piecewise-linear-interpolation-lookup-table.md).

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D volume tensor $V$ of scalar values (e.g., HU), and a set of $K$ keypoints defining the transfer function.
*   **Output:** A transformed volume tensor $M$ containing the mapped values (e.g., attenuation coefficients $\mu$ or RGBA properties).

**Data Structures:**
The transfer function is defined by a tensor of keypoints $P$ with shape $(K, 2)$, where $K$ is the number of control points.
*   Column 0: Input domain values (e.g., HU).
*   Column 1: Output domain values (e.g., attenuation $\mu$ or opacity $\alpha$).

**Steps:**
1.  **Define Keypoints:** Construct the tensor $P \in \mathbb{R}^{K \times 2}$. Each row $i$ contains $(x_i, y_i)$, where $x_i$ is the input scalar value and $y_i$ is the target output value [S1, unofficial].
2.  **Normalize Input Domain:** Map the input scalar values to a normalized range $[0, 1]$ to facilitate interpolation. For HU values, the standard normalization is:
    $$ x_p = \frac{x + 3500}{7000} $$
    where $x$ is the raw HU value. The range $[-3500, 3071]$ maps to $[0, 1]$ [S1, unofficial].
3.  **Extract Normalized Keypoints:** Create 1D tensors $X_p$ and $Y_p$ of shape $(1, K)$ containing the normalized input and output values from $P$ [S1, unofficial].
4.  **Interpolate:** Apply piecewise linear interpolation to the volume. For each voxel value $v$ in $V$:
    *   Normalize $v$ to $v_p$ using the formula in Step 2.
    *   Find the interval $[x_{p,j}, x_{p,j+1}]$ such that $x_{p,j} \le v_p \le x_{p,j+1}$.
    *   Compute the interpolated output $y_p$ using linear interpolation:
        $$ y_p = y_{p,j} + \frac{(v_p - x_{p,j})}{(x_{p,j+1} - x_{p,j})} (y_{p,j+1} - y_{p,j}) $$
    *   Map $y_p$ back to the physical output range if necessary (e.g., if $Y_p$ was normalized).
    *   Assign the result to the corresponding position in the output volume $M$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def piecewise_linear_hu_to_mu(volume, keypoints):
    """
    Maps a volume of HU values to attenuation coefficients using piecewise linear interpolation.
    
    Args:
        volume: 3D numpy array or tensor of shape (D, H, W) containing HU values.
        keypoints: 2D array/tensor of shape (K, 2). 
                   Column 0: HU values (x).
                   Column 1: Attenuation values (y).
                   
    Returns:
        mu_volume: 3D array/tensor of shape (D, H, W) with mapped attenuation values.
    """
    # 1. Extract input (x) and output (y) keypoints
    x_kp = keypoints[:, 0]
    y_kp = keypoints[:, 1]
    
    # 2. Sort keypoints by x value to ensure monotonicity for interpolation
    sort_idx = np.argsort(x_kp)
    x_kp = x_kp[sort_idx]
    y_kp = y_kp[sort_idx]
    
    # 3. Define normalization parameters for HU
    # Standard range: -3500 to 3071 maps to 0 to 1
    # Note: If keypoints are already in normalized [0,1] space, skip this step.
    # Assuming keypoints are in raw HU space for this implementation.
    x_min = -3500.0
    x_max = 3071.0 # 3500 + 7000*1.0 - 3500? No, 7000 range. -3500 + 7000 = 3500. 
                   # Source says 3071. We follow source.
    
    def normalize_hu(hu):
        return (hu - x_min) / (x_max - x_min)
    
    def denormalize_hu(norm_val):
        # If output y_kp is also normalized, we might need to denormalize.
        # Assuming y_kp is in physical units (e.g., attenuation coefficient).
        return norm_val 

    # 4. Normalize keypoints
    x_kp_norm = normalize_hu(x_kp)
    y_kp_norm = y_kp # Assuming y is not normalized or already in target units
    
    # 5. Process volume
    # Flatten for vectorized operation or iterate
    flat_vol = volume.flatten()
    flat_norm = np.clip(normalize_hu(flat_vol), 0.0, 1.0)
    
    # 6. Interpolate
    # np.interp performs piecewise linear interpolation
    # x: 1D array of query points
    # xp: 1D array of sample points (normalized keypoints)
    # fp: 1D array of sample values (output keypoints)
    mu_flat = np.interp(flat_norm, x_kp_norm, y_kp_norm)
    
    # 7. Reshape output
    mu_volume = mu_flat.reshape(volume.shape)
    
    return mu_volume
```

**Parameters:**
*   **Keypoints ($K$):** The number of control points. Typical values include 5-10 for medical imaging (e.g., air, lung, soft tissue, bone, metal) [S1, unofficial].
*   **HU Range:** The input domain is typically $[-3500, 3071]$ for medical CT data, normalized to $[0, 1]$ [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Out-of-Bounds Values:** Values outside the range of the keypoints must be clamped to the first or last keypoint value. The normalization step should clamp normalized values to $[0, 1]$ [G].
*   **Unsorted Keypoints:** The interpolation function requires keypoints to be sorted by their input domain values. Unsorted keypoints will produce incorrect results [G].
*   **Identical X Values:** Keypoints with identical input domain values ($x_i = x_{i+1}$) cause division by zero in linear interpolation. These should be removed or merged [G].

## Subtopics

### Target Value Configuration
A variant where the user configures up to five target data values, each associated with an RGB color triplet, an allowable value deviation window, and a maximum opacity [S2, unofficial]. This method is used to highlight specific isovalues in the volume.
*   **Definition:** Users select target values and define a "window" of allowable deviation. Samples near the center of the window are more opaque than those near the edges [S2, unofficial].
*   **Parameters:**
    *   **Number of Targets:** Up to 5 [S2, unofficial].
    *   **RGB Triplets:** Three real numbers in $[0.0, 1.0]$ per target [S2, unofficial].
    *   **Deviation Window:** Defines the range of values rendered with non-zero opacity around the target [S2, unofficial].
    *   **Max Opacity:** A value in $[0.0, 1.0]$ scaling the final opacity [S2, unofficial].
*   **Pitfalls:** Using identical RGB triplets for different target values causes color ambiguity. RGB triplets must be distinct enough to prevent color crossover due to surface highlighting variations [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of voxels, assuming $K$ is small and constant. Interpolation is a constant-time operation per voxel if the interval is found via binary search or direct indexing [G].
*   **Memory Complexity:** $O(N)$ for the output volume. The transfer function itself requires $O(K)$ memory.
*   **Strengths:** Computationally inexpensive, easy to implement, and intuitive for users to define material boundaries [S3, unofficial].
*   **Limitations:** Lacks smoothness ($C^0$ continuous only), which can result in visible "steps" or artifacts in the rendered image if the number of keypoints is low [S3, unofficial]. Not suitable for complex, non-linear material distributions without many keypoints.
*   **When to Use:** Real-time volume rendering where performance is critical, or when the data has distinct, well-separated density classes [S3, unofficial].

## Background
The method originates from early volume rendering techniques for medical imaging, specifically mapping Hounsfield Units to attenuation coefficients [S1, unofficial]. It is a standard implementation of piecewise linear transfer functions, widely used in tools like VTK and various research frameworks [S1, unofficial][S3, unofficial]. The specific normalization of HU values to $[-3500, 3071]$ is common in CT imaging [S1, unofficial].

## Sources
[S1] pcarnah/flashdrr, https://github.com/pcarnah/flashdrr, fetched 2026-10-26 (unofficial)
[S2] DTIC ADA202661: Volumetric Rendering Techniques for the Display of Three-Dimensional Aerodynamic Flow Field Data, https://archive.org/details/DTIC_ADA202661, fetched 2026-10-26 (unofficial)
