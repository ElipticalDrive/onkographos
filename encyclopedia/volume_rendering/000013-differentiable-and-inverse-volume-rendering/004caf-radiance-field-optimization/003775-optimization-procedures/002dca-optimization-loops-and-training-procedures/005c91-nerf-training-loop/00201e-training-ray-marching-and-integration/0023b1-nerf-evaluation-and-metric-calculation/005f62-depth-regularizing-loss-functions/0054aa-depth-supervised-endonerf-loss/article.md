# Depth-Supervised EndoNeRF Loss

Also called: Depth-Supervised Loss, EndoNeRF Loss

## Overview
A training objective that combines photometric color loss with a depth-supervised loss using estimated stereo depth to accelerate convergence and remove the need for a depth refinement module [S1]. It sets itself apart from standard NeRF losses by explicitly constraining the rendered depth to match external stereo-matching estimates, thereby reducing training time compared to methods like STTR that require iterative depth refinement [S1].

## How it works
**Inputs and Outputs**
*   **Inputs**:
    *   $\mathcal{R}$: The set of camera rays $\textbf{r}(z)$.
    *   $t \in [0,1]$: The time parameter for dynamic scenes.
    *   $\hat{\textbf{C}}(\textbf{r}(z), t)$: The rendered color for ray $\textbf{r}(z)$ at time $t$.
    *   $\textbf{C}(\textbf{r}(z), t)$: The ground truth pixel color for ray $\textbf{r}(z)$ at time $t$.
    *   $\hat{\textbf{D}}(\textbf{r}(z), t)$: The rendered depth for ray $\textbf{r}(z)$ at time $t$.
    *   $\textbf{D}(\textbf{r}(z), t)$: The estimated stereo depth for ray $\textbf{r}(z)$ at time $t$, obtained via stereo-matching feature correspondence of robotic surgery videos using Unimatch [S1].
    *   $\lambda_d$: The weight coefficient for the depth loss term [S1].
*   **Output**:
    *   $\mathcal{L}$: The total loss value.

**Data Structures**
*   **Rays**: Represented as $\textbf{r}(z)$, where $z_j$ is the distance of the $j$-th sample along the ray.
*   **Depth Maps**: $\textbf{D}$ is a 2D map (or per-ray value) derived from Unimatch [S1]. No specific bit-packing is defined in the sources; it is treated as a dense depth estimate [S1].

**Steps**
1.  **Estimate Stereo Depth**: Compute the estimated stereo depth $\textbf{D}(\textbf{r}(z),t)$ by performing stereo-matching on feature correspondence of the robotic surgery videos using the Unimatch algorithm [S1].
2.  **Render Color and Depth**: For each ray $\textbf{r}(z) \in \mathcal{R}$ and time $t \in [0,1]$, compute the rendered color $\hat{\textbf{C}}(\textbf{r}(z), t)$ and rendered depth $\hat{\textbf{D}}(\textbf{r}(z), t)$ from the current radiance field representation [S1].
3.  **Compute Photometric Loss**: Calculate the L2 norm squared between the rendered color and the ground truth color: $\left\| \hat{\textbf{C}}(\textbf{r}(z), t) - \textbf{C}(\textbf{r}(z), t) \right\| _2^2$ [S1].
4.  **Compute Depth Loss**: Calculate the Huber loss between the rendered depth and the estimated stereo depth: ${\text {Huber}}\left( \hat{\textbf{D}}(\textbf{r}(z), t), \textbf{D}(\textbf{r}(z), t) \right)$ [S1].
5.  **Aggregate Loss**: Sum the photometric and weighted depth losses over all rays and time steps to obtain $\mathcal{L}$ [S1].

**Formulas**
The total loss function is defined as [S1]:

$$
\mathcal {L}= \sum _{\textbf{r}(z) \in \mathcal {R}, t\in [0,1]} \left\| \hat{\textbf{C}}(\textbf{r}(z), t) - \textbf{C}(\textbf{r}(z), t) \right\| _2^2 + \lambda _d {\text {Huber}}\left( \hat{\textbf{D}}(\textbf{r}(z), t), \textbf{D}(\textbf{r}(z), t) \right)
$$

Where:
*   $\textbf{r}(z)$ is the camera ray.
*   $z_j$ is the distance of the $j$-th sample along the ray.
*   $t \in [0,1]$ is the time.
*   $\hat{\textbf{C}}$ is the rendered color.
*   $\textbf{C}$ is the ground truth pixel color.
*   $\hat{\textbf{D}}$ is the rendered depth.
*   $\textbf{D}$ is the estimated stereo depth from Unimatch.
*   $\lambda_d$ is the weight for the depth loss.

**Parameters**
*   $\lambda_d$: The weight coefficient for the Huber depth loss term. Typical values are not specified in the sources [S1].

**Pitfalls**
*   **Outliers**: The Huber loss is adopted for the depth term because it is more stable to outliers than L2 loss [S1]. Using L2 for depth may lead to instability if the stereo estimates contain noise or errors [S1].
*   **Stereo Quality**: The effectiveness of the depth supervision depends on the quality of the stereo-matching provided by Unimatch. Poor stereo correspondence may lead to suboptimal convergence [S1].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def depth_supervised_endonerf_loss(
    rays: List[Ray],
    time_steps: List[float],
    rendered_colors: Tensor,      # Shape: (num_rays, num_time_steps, 3)
    ground_truth_colors: Tensor,  # Shape: (num_rays, num_time_steps, 3)
    rendered_depths: Tensor,      # Shape: (num_rays, num_time_steps)
    stereo_depths: Tensor,        # Shape: (num_rays, num_time_steps)
    lambda_d: float
) -> Tensor:
    """
    Computes the Depth-Supervised EndoNeRF Loss.
    
    Args:
        rays: List of camera rays.
        time_steps: List of time values t in [0, 1].
        rendered_colors: Predicted RGB values from the radiance field.
        ground_truth_colors: Ground truth RGB values from the input images.
        rendered_depths: Predicted depth values from the radiance field.
        stereo_depths: Estimated stereo depth values from Unimatch.
        lambda_d: Weight for the depth loss term.
        
    Returns:
        Total loss value.
    """
    # 1. Photometric Color Loss (L2)
    # Calculate element-wise squared difference between rendered and ground truth colors
    color_diff = rendered_colors - ground_truth_colors
    color_loss = torch.sum(color_diff ** 2)
    
    # 2. Depth Loss (Huber)
    # Calculate Huber loss between rendered and stereo depth
    depth_diff = rendered_depths - stereo_depths
    # Huber loss implementation: 
    # If |diff| <= delta: 0.5 * diff^2
    # Else: delta * (|diff| - 0.5 * delta)
    # Note: The specific delta value for Huber loss is not given in sources [S1].
    # Using a standard default delta of 1.0 if not specified.
    delta = 1.0
    abs_diff = torch.abs(depth_diff)
    quadratic = torch.clamp(abs_diff, max=delta)
    linear = abs_diff - quadratic
    depth_loss = torch.sum(0.5 * quadratic ** 2 + delta * linear)
    
    # 3. Total Loss
    total_loss = color_loss + lambda_d * depth_loss
    
    return total_loss
```

## Subtopics

### Depth Supervision and Pseudo-Depth
Uses external depth estimates to guide training. Sets itself apart by using stereo-matching (Unimatch) rather than pseudo-depth from other NeRF variants. Choose when stereo data is available and training speed is critical. [Link](13742-depth-supervision-and-pseudo-depth.md)

### MIMO Objective
Focuses on multi-modal inputs. Sets itself apart by incorporating additional modalities beyond RGB and depth. Choose when multi-sensor data is available. [Link](10931-mimo-objective.md)

## Operating characteristics
*   **Time Complexity**: Not given in the sources.
*   **Memory Complexity**: Not given in the sources.
*   **Measured Performance**: Supervising with Unimatch depth maps decreases training time compared to STTR [S1]. This is because the depth refinement module proposed in EndoNeRF (which requires depth rendering of all training images) is no longer needed [S1].
*   **Scaling**: Not given in the sources.
*   **Strengths**: Accelerates convergence; removes the need for a computationally expensive depth refinement module; robust to outliers via Huber loss.
*   **Limitations**: Relies on the accuracy of the external stereo-matching algorithm (Unimatch); requires stereo-capable input data or pre-computed stereo depths.
*   **When to use**: When training dynamic radiance fields for robotic surgery videos where stereo depth estimates are available and training efficiency is a priority.
*   **When not**: When stereo depth estimates are unreliable or unavailable; when computational resources for external stereo matching are prohibitive.

## Background
The method originates from the Efficient EndoNeRF paper, which focuses on reconstruction and data-driven surgical simulation [S1]. It was developed to address the computational overhead of depth refinement in previous EndoNeRF variants [S1]. The approach leverages Unimatch for stereo depth estimation [S1].

## Sources
[S1] Efficient EndoNeRF reconstruction and its application for data-driven surgical simulation., https://doi.org/10.1007/s11548-024-03114-1, fetched 2026-10-26
