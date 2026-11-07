# Space Warping (Log+Warp)

Also called: log+warp

## Overview
Space Warping (Log+Warp) is a positional encoding strategy for neural radiance fields that warps 3D space towards the view cell center using radial distortion. It solves the problem of high-frequency background details requiring excessive network capacity by dampening these frequencies while preserving foreground detail [S1]. It sets itself apart from standard logarithmic sampling by applying an additional inverse-square-root warp to the encoded coordinates, ensuring that sample locations in space remain consistent across different viewpoints [S1].

## How it works
**Inputs and Outputs:**
*   Input: A query position $\mathbf{x}$ in 3D space, the view cell center $\mathbf{c}$, and a maximum depth parameter $d_{max}$.
*   Output: A warped feature vector $\mathbf{f}$ suitable for neural network input.

**Data Structures:**
*   $\mathbf{x}$: 3D vector representing the sample position.
*   $\mathbf{c}$: 3D vector representing the camera/view cell center.
*   $\tilde{d}_i$: Logarithmic depth value associated with the sample.

**Steps:**
1.  Compute the relative position vector $\mathbf{p}$ from the view cell center to the sample position.
2.  Calculate the warping scalar $W(\mathbf{p})$ using the inverse square root of the product of the magnitude of $\mathbf{p}$ and $d_{max}$.
3.  Scale the relative position vector by the warping scalar.
4.  Encode the scaled vector using a positional encoding function.

**Formulas:**
The warping function is defined as [S1]:
$$ W(\mathbf{p}) = \frac{1}{\sqrt{|\mathbf{p}| \cdot d_{max}}} $$

The warped feature vector is computed as [S1]:
$$ \mathbf{f} = \text{encode}\left( (\mathbf{x}(\tilde{d}_i) - \mathbf{c}) \cdot W(\mathbf{x}(\tilde{d}_i) - \mathbf{c}) \right) $$

Where:
*   $\mathbf{x}(\tilde{d}_i)$ is the 3D position derived from the logarithmic depth $\tilde{d}_i$.
*   $\mathbf{c}$ is the view cell center.
*   $\text{encode}(\cdot)$ is the positional encoding function (e.g., sine/cosine mappings).
*   $d_{max}$ is a scaling parameter representing the maximum depth extent.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def log_warp_encode(x, c, d_max, encode_func):
    """
    Computes the Log+Warp positional encoding.
    
    Args:
        x: numpy array, shape (3,), 3D position of the sample.
        c: numpy array, shape (3,), view cell center.
        d_max: float, maximum depth parameter.
        encode_func: callable, positional encoding function.
        
    Returns:
        numpy array, the encoded feature vector.
    """
    # Step 1: Compute relative position
    p = x - c
    
    # Step 2: Calculate warping scalar
    # Note: Handle case where p is zero to avoid division by zero
    magnitude_p = np.linalg.norm(p)
    if magnitude_p == 0:
        w = 1.0 / np.sqrt(d_max) # Fallback or limit as p->0
    else:
        w = 1.0 / np.sqrt(magnitude_p * d_max)
        
    # Step 3: Scale relative position
    scaled_p = p * w
    
    # Step 4: Encode
    f = encode_func(scaled_p)
    
    return f
```

**Parameters:**
*   $d_{max}$: Not given in the sources.

**Edge Cases:**
*   If $\mathbf{p} = \mathbf{0}$ (sample is at the view cell center), the denominator becomes zero. The implementation must handle this singularity, typically by clamping or using a limit value [G].

## Subtopics

### [NDC Warping](7267-ndc-warping.md)
NDC Warping is a space-warping technique for forward-facing scenes that maps an infinitely far view frustum to a bounded box by squashing the z-axis. It differs from Log+Warp by operating in Normalized Device Coordinates (NDC) space, providing linear sampling in disparity from the near plane to infinity in the original space. It is chosen when the scene is forward-facing and a bounded NDC representation is beneficial. See [NDC Warping](7267-ndc-warping.md).

### Proper Space Warping
Proper Space Warping defines a warping function $F: \mathbb{R}^3 \to \mathbb{R}^3$ such that for any two points $x_1, x_2$ in a visible region $S$, the squared Euclidean distance in warp space equals the sum of squared distances on all visible cameras $C_i$ [S2]:
$$ \|F(x_1) - F(x_2)\|_2^2 = \sum_i \|C_i(x_1) - C_i(x_2)\|_2^2 $$
It is constructed by finding a projection matrix $M$ that minimizes reconstruction error. The function is $F(x) = M G(x)$, where $G(x) = [C_1(x), \dots, C_{n_c}(x)]$ maps 3D points to projected 2D coordinates [S2]. $M$ is derived from the first three eigenvectors of the covariance matrix of projected points obtained via Principal Component Analysis (PCA) on evenly sampled points in $S$ [S2]. A post-normalization step ensures points in warp space are located around the origin [S2]. It is chosen when multi-view consistency based on camera projections is critical [S2].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Performance:** The warping brings background closer to the view cell for positional encoding, dampening high frequencies in the background while preserving foreground detail, which reduces the burden on the network [S1].
*   **Strengths:** Improves network efficiency for background regions; maintains consistent sample evaluation across viewpoints [S1].
*   **Limitations:** Ray samples do not follow a straight line after warping, although sample locations in space stay consistent [S1].
*   **When to use:** When rendering scenes with significant background detail that requires high-frequency encoding, or when computational efficiency of the neural network is a priority [S1].
*   **When not to use:** When strict linear ray paths in warped space are required for other downstream processes [S1].

## Background
Space Warping (Log+Warp) was introduced in the DONeRF paper [S1]. It addresses the issue where real cameras and graphics techniques like mip-mapping average background details, which neural networks must then reconstruct, requiring high frequencies that are inefficient to encode [S1]. Proper Space Warping was introduced in the F2-NeRF paper [S2] to provide a mathematically rigorous definition of warping based on multi-view consistency.

## Sources
[S1] DONeRF: Towards Real‐Time Rendering of Compact Neural Radiance Fields using Depth Oracle Networks, https://doi.org/10.1111/cgf.14340, fetched 2026-10-26
[S2] F2-NeRF: Fast Neural Radiance Field Training with Free Camera Trajectories, https://doi.org/10.1109/cvpr52729.2023.00404, fetched 2026-10-26
