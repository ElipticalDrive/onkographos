# 2D Gaussian Alpha Computation (Batched)

Also called: Alpha Computation, Batched Gaussian Alpha Forward

## Overview
This method computes the alpha values for all $N$ 2D Gaussians across all $W \times H$ pixels in a single batched operation, serving as the core rasterization step in Gaussian Splatting [S1, unofficial]. It builds on [Single Point Alpha Forward](3478-single-point-alpha-forward.md) by vectorizing the evaluation over the entire image grid, enabling efficient GPU execution [G]. It differs from [Explicit Intersection Alpha Compositing](4814-explicit-intersection-alpha-compositing.md) by computing raw alpha contributions for all pixel-gaussian pairs rather than identifying explicit intersection pairs for compositing [S1, unofficial].

## How it works
**Inputs:**
- `means_2D`: Tensor of shape $(N, 2)$, the 2D means of the Gaussians in screen space [S1, unofficial].
- `cov_2D`: Tensor of shape $(N, 2, 2)$, the 2D covariance matrices for each Gaussian [S1, unofficial].
- `opacities`: Tensor of shape $(N,)$, the opacity values for each Gaussian [S1, unofficial].
- $W, H$: Integer width and height of the image [S1, unofficial].

**Outputs:**
- `alphas`: Tensor of shape $(N, H, W)$, the alpha values for each Gaussian at each pixel [S1, unofficial].

**Data Structures:**
- `points_2D`: Tensor of shape $(W \cdot H, 2)$, containing the coordinates of every pixel center [S1, unofficial].
- `power`: Tensor of shape $(N, W \cdot H)$, containing the negative half-squared Mahalanobis distance for each Gaussian-pixel pair [S1, unofficial].
- `exp_power`: Tensor of shape $(N, W \cdot H)$, containing the exponential of the power term [S1, unofficial].

**Steps:**
1. **Generate Pixel Grid:** Create a meshgrid of pixel coordinates $(x, y)$ for $x \in [0, W-1]$ and $y \in [0, H-1]$, stack and flatten them into `points_2D` of shape $(W \cdot H, 2)$ [S1, unofficial].
2. **Broadcast Dimensions:** Expand `points_2D` to shape $(1, W \cdot H, 2)$ and `means_2D` to shape $(N, 1, 2)$ to enable broadcasting during evaluation [S1, unofficial].
3. **Evaluate 2D Gaussian:** Compute the power term for all pairs using the standard 2D Gaussian exponent $-\frac{1}{2} (\mathbf{x} - \boldsymbol{\mu})^T \Sigma^{-1} (\mathbf{x} - \boldsymbol{\mu})$, resulting in `power` of shape $(N, W \cdot H)$ [G].
4. **Compute Exponential:** Calculate the exponential term, clamping positive power values to zero to prevent numerical overflow: $\text{exp\_power} = \exp(\min(0, \text{power}))$ [S1, unofficial].
5. **Apply Opacity:** Multiply the exponential term by the opacity of each Gaussian, resulting in a tensor of shape $(N, W \cdot H)$ [S1, unofficial].
6. **Reshape:** Reshape the alpha tensor to $(N, H, W)$ to match the image layout [S1, unofficial].
7. **Clamp for Stability:** Clamp all alpha values to a maximum of $0.99$ to prevent numerical instability during subsequent compositing steps [S1, unofficial].
8. **Threshold Small Values:** Zero out alpha values smaller than $1/255.0$ to reduce memory bandwidth and computation for negligible contributions [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_batched_alphas(means_2D, cov_2D, opacities, W, H):
    # Step 1: Generate pixel grid
    points_2D = []
    for y in range(H):
        for x in range(W):
            points_2D.append([float(x), float(y)])
    # points_2D shape: (W*H, 2)

    # Step 2: Broadcast dimensions (conceptual for pure Python, handled by loops below)
    N = len(means_2D)
    num_pixels = W * H

    # Step 3: Evaluate 2D Gaussian power
    power = [[0.0] * num_pixels for _ in range(N)]
    for i in range(N):
        cov_inv = inverse_2x2(cov_2D[i])
        for j in range(num_pixels):
            diff = [points_2D[j][0] - means_2D[i][0], points_2D[j][1] - means_2D[i][1]]
            left_prod = matmul_vector_2x2(diff, cov_inv)
            dot = left_prod[0] * diff[0] + left_prod[1] * diff[1]
            power[i][j] = -0.5 * dot

    # Step 4: Compute exponential with overflow guard
    exp_power = [[math.exp(power[i][j]) if power[i][j] <= 0.0 else 0.0 for j in range(num_pixels)] for i in range(N)]

    # Step 5: Apply opacity
    alphas_flat = [[opacities[i] * exp_power[i][j] for j in range(num_pixels)] for i in range(N)]

    # Step 6: Reshape to (N, H, W)
    alphas = [[[alphas_flat[i][r * W + c] for c in range(W)] for r in range(H)] for i in range(N)]

    # Step 7: Clamp for stability
    alphas = [[[min(alphas[i][r][c], 0.99) for c in range(W)] for r in range(H)] for i in range(N)]

    # Step 8: Threshold small values
    alphas = [[[0.0 if alphas[i][r][c] < 1.0 / 255.0 else alphas[i][r][c]] for c in range(W)] for r in range(H)] for i in range(N)]

    return alphas

def inverse_2x2(M):
    det = M[0][0] * M[1][1] - M[0][1] * M[1][0]
    return [[M[1][1] / det, -M[0][1] / det], [-M[1][0] / det, M[0][0] / det]]

def matmul_vector_2x2(v, M):
    return [M[0][0] * v[0] + M[0][1] * v[1], M[1][0] * v[0] + M[1][1] * v[1]]
```

**Parameters:**
- Clamping threshold: $0.99$ [S1, unofficial].
- Zeroing threshold: $1/255.0$ [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Numerical Overflow:** Positive power values due to precision errors cause $\exp(\text{power})$ to overflow, so the source clamps them to zero before exponentiation [S1, unofficial].
- **Covariance Inversion:** Degenerate covariances with zero determinants cause inversion failures, which is typically handled by ensuring valid covariance matrices in earlier pipeline stages [G].
- **Memory Usage:** The intermediate `power` tensor has shape $(N, W \cdot H)$, creating a memory bottleneck for high-resolution images and many Gaussians [G].

## Subtopics

### [Explicit Intersection Alpha Compositing](4814-explicit-intersection-alpha-compositing.md)
The final rasterization pass in the 2D Gaussian Splatting pipeline that combines pre-computed alpha values into a final RGB image and alpha mask using front-to-back compositing. It is distinguished by operating on explicit intersection pairs `(gaussian_id, pixel_id, image_id)` rather than iterating over pixels or Gaussians, allowing arbitrary intersection patterns and leveraging automatic differentiation for the backward pass. Choose it when explicit ray-splat intersection tracking and gradient flow are required. See [Explicit Intersection Alpha Compositing](4814-explicit-intersection-alpha-compositing.md).

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Performance:** Not given in the sources.
- **Strengths:** Not given in the sources.
- **Limitations:** Not given in the sources.
- **When to use:** Not given in the sources.

## Background
This method is derived from the implementation details of 3D Gaussian Splatting, specifically the batched alpha computation step used in rasterization [S1, unofficial]. The specific implementation details, including the clamping and thresholding values, are taken from an unofficial repository [S1, unofficial]. The approach leverages the vectorization capabilities of deep learning frameworks like PyTorch to perform the rasterization efficiently [G].

## Sources
[S1] Fhz2346/3D-Gaussian-Splatting, https://github.com/Fhz2346/3D-Gaussian-Splatting, fetched 2026-10-27 (unofficial)
