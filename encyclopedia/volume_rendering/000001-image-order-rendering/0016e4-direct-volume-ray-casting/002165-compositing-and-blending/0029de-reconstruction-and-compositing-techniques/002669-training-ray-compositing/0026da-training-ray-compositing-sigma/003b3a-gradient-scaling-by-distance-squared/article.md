# Gradient Scaling by Distance Squared


## Overview
Gradient Scaling by Distance Squared is a technique used during the backpropagation phase of training ray compositing to improve training stability for distant objects [S1, unofficial]. It scales the gradients of field outputs based on the squared distance of samples from the camera. This method is a specific variant of Training Ray Compositing (Sigma) [3868-training-ray-compositing-sigma.md] that addresses the imbalance in gradient magnitudes caused by varying sample depths along a ray.

## How it works
**Inputs and Outputs:**
- **Inputs:** The gradients of the field outputs (e.g., density $\sigma$, color $\mathbf{c}$) with respect to the network parameters, and the positions of the ray samples relative to the camera origin.
- **Outputs:** The scaled gradients, which are then used to update the network parameters via an optimizer.

**Data Structures:**
- **Ray Samples:** A tensor of shape $(N, 3)$ or $(N, 4)$ containing the 3D coordinates of $N$ samples along the ray, typically in world or camera space.
- **Gradients:** A tensor of the same shape as the field outputs, containing the upstream gradients $\frac{\partial L}{\partial \hat{y}}$.

**Steps:**
1. **Check Enable Flag:** Determine if gradient scaling is enabled via the boolean parameter `use_gradient_scaling` [S1, unofficial].
2. **Compute Distances:** For each sample $i$ along the ray, calculate the Euclidean distance $d_i$ from the camera origin to the sample position $\mathbf{p}_i$.
   $$ d_i = \| \mathbf{p}_i - \mathbf{o} \|_2 $$
   where $\mathbf{o}$ is the camera origin.
3. **Compute Scaling Factors:** Calculate the scaling factor $s_i$ for each sample as the inverse of the squared distance.
   $$ s_i = \frac{1}{d_i^2} $$
   [G] To prevent division by zero when a sample is at the camera origin, a small epsilon $\epsilon$ is added to the denominator: $s_i = \frac{1}{d_i^2 + \epsilon}$.
4. **Apply Scaling:** Multiply the upstream gradients $\frac{\partial L}{\partial \hat{y}_i}$ by the scaling factor $s_i$ to obtain the scaled gradients $\frac{\partial L}{\partial y_i}$.
   $$ \frac{\partial L}{\partial y_i} = s_i \cdot \frac{\partial L}{\partial \hat{y}_i} $$
5. **Backpropagate:** Pass the scaled gradients to the subsequent layers of the network for parameter updates.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def scale_gradients_by_distance_squared(gradients, ray_samples, camera_origin, use_gradient_scaling=True, epsilon=1e-6):
    """
    Scales gradients based on the squared distance of samples from the camera.
    
    Args:
        gradients (Tensor): Upstream gradients of shape (N, C) where N is number of samples and C is channel dimension.
        ray_samples (Tensor): Sample positions of shape (N, 3) or (N, 4).
        camera_origin (Tensor): Camera origin of shape (3,) or (4,).
        use_gradient_scaling (bool): Whether to apply scaling.
        epsilon (float): Small value to prevent division by zero.
        
    Returns:
        Tensor: Scaled gradients of shape (N, C).
    """
    if not use_gradient_scaling:
        return gradients
        
    # Extract 3D positions if 4D (homogeneous)
    if ray_samples.shape[1] == 4:
        positions = ray_samples[:, :3]
    else:
        positions = ray_samples
        
    # Compute distances from camera origin
    # camera_origin might need broadcasting if it's (1, 3)
    distances = torch.norm(positions - camera_origin, dim=1, keepdim=True)
    
    # Compute scaling factors: 1 / (d^2 + epsilon)
    scaling_factors = 1.0 / (distances**2 + epsilon)
    
    # Apply scaling
    # gradients shape: (N, C), scaling_factors shape: (N, 1)
    scaled_gradients = gradients * scaling_factors
    
    return scaled_gradients
```

**Parameters:**
- `use_gradient_scaling`: A boolean flag to enable or disable gradient scaling [S1, unofficial].
- `epsilon`: A small constant to prevent division by zero. Not explicitly given in sources, but standard practice [G].

**Edge Cases and Pitfalls:**
- **Zero Distance:** If a sample is exactly at the camera origin, $d_i = 0$, leading to division by zero. The epsilon guard is necessary [G].
- **Numerical Stability:** For very large distances, $d_i^2$ may overflow or underflow depending on the data type. Using inverse distance squared directly helps mitigate this compared to computing distance first and then squaring.
- **Coordinate System:** The distance calculation assumes the ray samples and camera origin are in the same coordinate system. Mismatched systems will lead to incorrect scaling.

## Subtopics

### [Modulated Gradient Scaling](12838-modulated-gradient-scaling.md)
A gradient scaling technique that modulates the shape of the scaling function to adaptively reflect scene arrangements and guide rough geometry, specifically designed for NDC environments and sparse view settings to prevent density artifacts.

## Operating characteristics
- **Time Complexity:** $O(N)$, where $N$ is the number of samples per ray. The operation involves a norm calculation and element-wise multiplication.
- **Memory Complexity:** $O(N)$ for storing the scaling factors if not computed on the fly.
- **Strengths:** Improves training stability by balancing gradient magnitudes between near and far samples. Prevents the model from being dominated by gradients from distant, potentially less informative samples.
- **Limitations:** May over-emphasize near samples if the scene has significant depth variation. Requires careful tuning of the epsilon parameter.
- **When to Use:** When training neural radiance fields or similar implicit representations where depth variation is significant and training instability is observed due to gradient imbalance.
- **When Not to Use:** When the scene is shallow or depth variation is minimal, or when using a different compositing strategy that inherently handles depth weighting.

## Background
Gradient Scaling by Distance Squared is a technique used in the context of training neural rendering models, specifically within the Ganerf framework [S1, unofficial]. It addresses the challenge of training stability in direct volume ray casting by ensuring that gradients from samples at different depths contribute appropriately to the parameter updates. The method is part of the broader family of Training Ray Compositing techniques [3868-training-ray-compositing-sigma.md].

## Sources
[S1] barbararoessle/ganerf, https://github.com/barbararoessle/ganerf, fetched 2026-10-27 (unofficial)
