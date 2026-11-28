# Gradient Scaling by Distance Squared
## Overview
Gradient Scaling by Distance Squared is a technique applied during the backpropagation phase of training ray compositing to improve training stability for distant objects [S1, unofficial]. It scales the gradients of field outputs based on the squared distance of samples from the camera [S1, unofficial]. As a child of Training Ray Compositing (Sigma) [3868-training-ray-compositing-sigma.md], it addresses gradient magnitude imbalances along rays by weighting parameter updates with sample depth [G].

## How it works
**Inputs and Outputs:**
- **Inputs:** Upstream gradients of field outputs (e.g., density $\sigma$, color $\mathbf{c}$), ray sample positions, and a boolean flag `use_gradient_scaling` [S1, unofficial].
- **Outputs:** Scaled gradients for network parameter updates [G].

**Data Structures:**
- **Ray Samples:** A tensor of shape $(N, 3)$ or $(N, 4)$ containing the 3D coordinates of $N$ samples along the ray [G].
- **Gradients:** A tensor of shape $(N, C)$ containing the upstream gradients $\frac{\partial L}{\partial \hat{y}}$, where $C$ is the channel dimension [G].

**Steps:**
1. **Check Enable Flag:** Evaluate `use_gradient_scaling` [S1, unofficial]. If false, return the input gradients unchanged [S1, unofficial].
2. **Extract Positions:** If ray samples are in homogeneous coordinates $(N, 4)$, discard the fourth component to obtain 3D positions $\mathbf{p}_i$ [G].
3. **Compute Distances:** Calculate the squared Euclidean distance $d_i^2$ from the camera origin $\mathbf{o}$ to each sample position $\mathbf{p}_i$:
   $$ d_i^2 = \sum_{k=1}^{3} (p_{i,k} - o_k)^2 $$
   where $p_{i,k}$ and $o_k$ are the $k$-th components of the sample and camera origin, respectively [G].
4. **Compute Scaling Factors:** Calculate the scaling factor $s_i$ for each sample as the inverse of the squared distance plus a small constant $\epsilon$:
   $$ s_i = \frac{1}{d_i^2 + \epsilon} $$
   [G] The constant $\epsilon$ prevents division by zero when a sample coincides with the camera origin.
5. **Apply Scaling:** Multiply the upstream gradients $\frac{\partial L}{\partial \hat{y}_{i,j}}$ by $s_i$ to obtain the scaled gradients $\frac{\partial L}{\partial y_{i,j}}$:
   $$ \frac{\partial L}{\partial y_{i,j}} = s_i \cdot \frac{\partial L}{\partial \hat{y}_{i,j}} $$
   [S1, unofficial]
6. **Backpropagate:** Pass the scaled gradients to the network optimizer for parameter updates [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def scale_gradients_by_distance_squared(gradients, ray_samples, camera_origin, use_gradient_scaling=True, epsilon=1e-6):
    """
    Scales gradients based on the squared distance of samples from the camera.
    
    Args:
        gradients (list of lists): Upstream gradients of shape (N, C).
        ray_samples (list of lists): Sample positions of shape (N, 3) or (N, 4).
        camera_origin (list): Camera origin of shape (3,).
        use_gradient_scaling (bool): Whether to apply scaling.
        epsilon (float): Small value to prevent division by zero.
        
    Returns:
        list of lists: Scaled gradients of shape (N, C).
    """
    if not use_gradient_scaling:
        return gradients
        
    # Extract 3D positions if 4D homogeneous coordinates are provided
    positions = [sample[:3] for sample in ray_samples] if len(ray_samples[0]) == 4 else ray_samples
    
    # Compute squared distances from camera origin
    distances_sq = [sum((p[k] - camera_origin[k])**2 for k in range(3)) for p in positions]
    
    # Compute scaling factors: 1 / (d^2 + epsilon)
    scaling_factors = [1.0 / (d + epsilon) for d in distances_sq]
    
    # Apply scaling element-wise
    scaled_gradients = [[gradients[i][j] * scaling_factors[i] for j in range(len(gradients[0]))] for i in range(len(gradients))]
    
    return scaled_gradients
```

**Parameters:**
- `use_gradient_scaling`: A boolean flag to enable or disable gradient scaling [S1, unofficial].
- `epsilon`: A small constant to prevent division by zero. Typical value: $10^{-6}$ [G].
- Typical values for other parameters: Not given in the sources [G].

**Edge Cases and Pitfalls:**
- **Zero Distance:** If a sample is exactly at the camera origin, $d_i = 0$, leading to division by zero. The $\epsilon$ guard is required [G].
- **Coordinate System:** The distance calculation assumes ray samples and the camera origin share the same coordinate system. Mismatched systems produce incorrect scaling [G].
- **Numerical Stability:** Not given in the sources [G].

## Subtopics

### [Modulated Gradient Scaling](12838-modulated-gradient-scaling.md)
A gradient regularization technique that adapts the shape of the distance-based scaling function using a sinusoidal curve to provide implicit geometric guidance, specifically chosen for sparse view and Normalized Device Coordinate (NDC) settings to prevent density artifacts and avoid local minima during initial training.

## Operating characteristics
- **Time Complexity:** $O(N)$ per ray, where $N$ is the number of samples [G].
- **Memory Complexity:** $O(N)$ for storing scaling factors if not computed in-place [G].
- **Strengths:** Improves training stability by balancing gradient magnitudes between near and far samples [S1, unofficial].
- **Limitations:** Not given in the sources [G].
- **When to Use:** Training neural radiance fields or similar implicit representations where depth variation is significant and gradient imbalance causes instability [G].
- **When Not to Use:** Not given in the sources [G].

## Background
Gradient Scaling by Distance Squared originates in the Ganerf framework [S1, unofficial]. It is implemented as part of the Training Ray Compositing family [3868-training-ray-compositing-sigma.md] to address gradient imbalance in direct volume ray casting by weighting backpropagation updates with sample depth [S1, unofficial].

## Sources
[S1] barbararoessle/ganerf, https://github.com/barbararoessle/ganerf, fetched 2026-10-27 (unofficial)
