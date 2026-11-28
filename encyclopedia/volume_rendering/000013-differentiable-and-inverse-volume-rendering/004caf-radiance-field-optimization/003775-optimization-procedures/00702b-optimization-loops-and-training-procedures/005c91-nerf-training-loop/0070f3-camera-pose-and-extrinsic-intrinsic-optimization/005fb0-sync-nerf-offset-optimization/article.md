# Sync-NeRF Offset Optimization

Also called: Joint offset and NeRF optimization, Per-camera Time Offset with Dynamic NeRF, Sync-MixVoxels, Sync-K-Planes

## Overview
Sync-NeRF Offset Optimization addresses the problem of reconstructing dynamic scenes from multi-view videos that are not temporally synchronized [S1]. Unlike standard dynamic NeRFs that assume all cameras capture the same scene moment at a given frame index, this method introduces learnable per-camera time offsets to align the temporal axes of unsynchronized inputs [S1]. It sets itself apart by treating synchronization as a differentiable optimization problem, jointly refining camera-specific time shifts alongside the neural radiance field parameters [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of $K$ unsynchronized multi-view video sequences [S1]. For each camera $k \in \{1, \dots, K\}$, the input consists of images $C_k(r, t)$ at discrete time indices $t$ [S1]. The underlying dynamic NeRF model $F_\Theta$ (which may be grid-based or implicit) and its initial parameters $\Theta$ [G].
*   **Outputs:** Optimized NeRF parameters $\Theta$, optimized per-camera time offsets $\delta_k$ for each training camera, and optionally a test-time offset $\delta_{test}$ for novel view synthesis [S1].

**Data Structures:**
*   **Time Offsets ($\delta$):** A vector of learnable parameters, one per camera [S1].
    *   Layout: $\delta = [\delta_1, \delta_2, \dots, \delta_K]$ [S1].
    *   Type: Continuous scalar (float) [S1].
    *   Initialization: $\delta_k = 0.0$ for all $k$ [S1].
*   **Calibrated Time ($t_k$):** The effective time input to the NeRF for a specific camera and frame [S1].
    *   Formula: $t_k = t + \delta_k$ [S1].

**Numbered Steps:**
1.  **Initialization:** Initialize the learnable time offsets $\delta_k$ for all $K$ training cameras to zero [S1]. Initialize the NeRF parameters $\Theta$ using the chosen dynamic NeRF baseline (e.g., MixVoxels, K-Planes) [G].
2.  **Calibration:** For each training pixel defined by camera index $k$, ray $r$, and frame index $t$, compute the calibrated time $t_k$:
    $$ t_k = t + \delta_k $$
    [S1]
3.  **Rendering:** Query the dynamic NeRF $F_\Theta$ using the calibrated time $t_k$ to predict the color and density [S1]. The specific query depends on the baseline architecture:
    *   For implicit function-based approaches:
        $$ F_\Theta : (x, d, t_k) \rightarrow (c, \sigma) $$
        [S1]
    *   For grid-based models (e.g., MixVoxels, K-Planes):
        $$ F_\Theta : (f(x, t_k), d) \rightarrow (c, \sigma) $$
        where $f(x, t_k)$ retrieves the spatial-temporal feature grid at coordinates $(x, t_k)$ [S1].
    *   Render the pixel color $\hat{C}(r, t + \delta_k)$ using standard volume rendering integration (numerical quadrature) [S1].
4.  **Loss Computation:** Calculate the Mean Squared Error (MSE) between the rendered color $\hat{C}$ and the ground truth color $C_k(r, t)$:
    $$ \mathcal{L}_{RGB} = \sum_{k,r,t} \left( \hat{C}(r, t + \delta_k) - C_k(r, t) \right)^2 $$
    [S1]
5.  **Optimization:** Minimize $\mathcal{L}_{RGB}$ with respect to both the NeRF parameters $\Theta$ and the time offsets $\delta_k$ using gradient descent [S1]. No additional regularizers are required for the offsets [S1].
6.  **Test-Time Synchronization (Optional):** If rendering from a test view that is also unsynchronized, freeze $\Theta$ and $\delta_k$ (training offsets) and optimize a specific test offset $\delta_{test}$ to minimize reconstruction error for the novel view [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sync_nerf_offset_optimization(cameras, time_indices, ground_truth_images, dynamic_nerf, num_iterations, learning_rate):
    # cameras: list of K camera indices
    # time_indices: list of integer frame indices
    # ground_truth_images: dict mapping (k, t) to image tensor C_k(r, t)
    # dynamic_nerf: object with .parameters() and .render(k, t_k) method
    # num_iterations: int
    # learning_rate: float

    # 1. Initialize learnable time offsets delta_k for each camera to zero
    delta = {k: 0.0 for k in cameras}

    # Combine parameters for optimization
    all_params = list(dynamic_nerf.parameters()) + list(delta.values())

    for iteration in range(num_iterations):
        # Zero gradients
        for p in all_params:
            p.grad = 0.0

        total_loss = 0.0

        # 2. Iterate over all training pixels (k, r, t)
        for k in cameras:
            for t in time_indices:
                # 3. Calculate calibrated time t_k = t + delta_k
                t_k = t + delta[k]

                # Get ground truth image for this camera and time
                C_gt = ground_truth_images[(k, t)]

                # 4. Compute rendered RGB pixel C_hat(r, t + delta_k)
                C_hat = dynamic_nerf.render(k, t_k)

                # 5. Compute MSE loss
                loss = sum((C_hat - C_gt) ** 2) / len(C_gt)
                total_loss += loss

        # 6. Backpropagate and update
        total_loss.backward()
        for p in all_params:
            p.data -= learning_rate * p.grad

    return dynamic_nerf, delta

def test_time_sync(dynamic_nerf, test_camera, test_ground_truth, delta_train, time_indices, num_iterations_test, learning_rate):
    # Optimize a specific test offset for a novel view
    delta_test = 0.0

    for iteration in range(num_iterations_test):
        delta_test.grad = 0.0

        total_loss = 0.0
        for t in time_indices:
            # Use frozen trained model and training offsets
            t_k = t + delta_train[test_camera]

            # Render with test offset
            C_hat = dynamic_nerf.render(test_camera, t_k + delta_test)
            C_gt = test_ground_truth.image

            loss = sum((C_hat - C_gt) ** 2) / len(C_gt)
            total_loss += loss

        total_loss.backward()
        delta_test.data -= learning_rate * delta_test.grad

    return delta_test
```

**Parameters:**
*   $\delta_k$: Learnable time offset for camera $k$. Initialized to $0.0$. Optimized via gradient descent [S1].
*   Typical Performance: On the Unsynchronized Dynamic Blender Dataset, the Mean Absolute Error (MAE) between predicted offsets and ground truth offsets is approximately $0.01$ seconds (less than one-third of a frame at 30 FPS) [S1].

**Edge Cases and Pitfalls:**
*   **Test View Synchronization:** Test views may also be unsynchronized relative to training views. The method requires optimizing a specific test offset $\delta_{test}$ with a frozen trained model to resolve this [S1].
*   **Unsynchronization Length:** Performance degrades with longer unsynchronization lengths (e.g., $1.5\times$ or $2\times$), though the method consistently improves over baselines [S1].

## Subtopics

### Sync-MixVoxels
Applies Sync-NeRF offset optimization to the MixVoxels baseline [S1]. It achieves an MAE of $0.0154$ seconds on the Unsynchronized Dynamic Blender Dataset [S1]. Choose this variant when using a grid-based implicit representation that benefits from continuous temporal interpolation [S1]. See [Sync-MixVoxels](sync-mixvoxels.md).

### Sync-K-Planes (Hybrid)
Applies Sync-NeRF offset optimization to the hybrid K-Planes baseline [S1]. It achieves an MAE of $0.0156$ seconds on the Unsynchronized Dynamic Blender Dataset [S1]. Choose this variant when a hybrid explicit/implicit representation is preferred for memory efficiency [S1]. See [Sync-K-Planes](sync-k-planes.md).

### Sync-K-Planes (Explicit)
Applies Sync-NeRF offset optimization to the explicit K-Planes baseline [S1]. It achieves an MAE of $0.0229$ seconds on the Unsynchronized Dynamic Blender Dataset [S1]. Choose this variant when fully explicit voxel-based rendering is required [S1]. See [Sync-K-Planes](sync-k-planes.md).

## Operating characteristics
*   **Time Complexity:** The optimization does not introduce additional computational complexity to the grid-based models during rendering, as the time offset is simply added to the time input before grid query [S1]. However, the joint optimization of $\delta_k$ adds gradient computation overhead for the offset parameters [G].
*   **Memory Complexity:** No additional memory is required for the offsets themselves (only $K$ floats), but the method avoids the memory overhead of learning hundreds of individual frame embeddings by using continuous time interpolation [S1].
*   **Performance:** On the Unsynchronized Dynamic Blender Dataset, Sync-NeRF variants significantly outperform unsynchronized baselines [S1]. For example, Sync-MixVoxels achieves a PSNR of $37.11$ compared to $31.06$ for the baseline MixVoxels [S1].
*   **Strengths:** Automatically synchronizes videos without manual effort or additional hardware (like timecode systems) [S1]. Applicable to various dynamic NeRF baselines [S1].
*   **Limitations:** Does not generalize to causal handheld cameras (future work) [S1]. May struggle with extreme unsynchronization lengths, though it remains robust [S1].

## Background
Sync-NeRF was introduced by Kim et al. in 2024 to address the limitation of existing dynamic NeRFs that assume synchronized multi-view inputs [S1]. Previous methods fail on unsynchronized data, producing severe artifacts and ghost effects because they employ a single latent embedding for a frame while multi-view images were captured at different moments [S1]. The method was verified on the Plenoptic Video Dataset and a newly built Unsynchronized Dynamic Blender Dataset [S1].

## Sources
[S1] Sync-NeRF: Generalizing Dynamic NeRFs to Unsynchronized Videos, https://doi.org/10.1609/aaai.v38i3.28057, fetched 2026-10-26
