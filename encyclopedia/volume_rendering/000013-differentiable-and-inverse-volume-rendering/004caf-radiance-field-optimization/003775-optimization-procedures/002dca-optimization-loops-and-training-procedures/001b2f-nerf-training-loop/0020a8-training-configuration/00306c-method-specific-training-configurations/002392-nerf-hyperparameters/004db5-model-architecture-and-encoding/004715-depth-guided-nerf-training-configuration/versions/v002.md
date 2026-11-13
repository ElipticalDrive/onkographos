# Depth-Guided NeRF Training Configuration

Also called: Depth-Guided NeRF Training

## Overview
This configuration defines the hyperparameters, architectural modifications, and optimization settings for training Neural Radiance Fields (NeRF) using depth priors to improve geometric reconstruction [S1]. It distinguishes itself from standard NeRF training by explicitly integrating a depth loss term and modifying the network architecture with dropout to prevent overfitting when leveraging auxiliary depth supervision [S1]. It builds on [Model Architecture and Encoding](13550-model-architecture-and-encoding.md).

## How it works
**Inputs:**
*   Camera poses and intrinsics.
*   RGB images.
*   Ground truth depth maps (optional, used for loss calculation).
*   Scene-specific depth bounds ($min\_depth$, $max\_depth$).

**Outputs:**
*   Trained NeRF model parameters (density $\sigma$ and color $c$).
*   Rendered RGB images and depth maps.

**Data Structures:**
*   **Neural Network:** Multi-Layer Perceptron (MLP) as defined in [Model Architecture and Encoding](13550-model-architecture-and-encoding.md).
    *   **Modification:** Dropout layers with probability $p=0.1$ are inserted after each of the first eight fully connected ReLU-activated layers that encode the positional encoding [S1].
*   **Loss Function Components:**
    *   RGB Loss: Standard pixel-wise difference (e.g., MSE) between rendered and ground truth RGB.
    *   Depth Loss: Weighted difference between rendered/estimated depth and ground truth depth.
    *   Uncertainty Weight: Scalar $\gamma$ scaling the depth loss contribution [S1].

**Parameters:**
*   **Depth Loss Weight ($\lambda$):**
    *   Default: $0.007$ [S1].
    *   Exception: $0.02$ for scene 710 [S1].
*   **Uncertainty Weight ($\gamma$):** $1$ [S1].
*   **Optimizer:** Adam [S1].
*   **Learning Rate ($lr$):**
    *   Base: $5 \times 10^{-4}$ [S1].
    *   Schedule: Reduced to $5 \times 10^{-5}$ after 80% of total steps [S1].
*   **Weight Decay:** $1 \times 10^{-6}$ [S1].
*   **Total Training Steps:** $500,000$ [S1].
*   **Depth Scale Learning Rate:** $1 \times 10^{-7}$ [S1].
*   **Depth Threshold ($\tau$):** Used to evaluate depth map uncertainty. Calculated as:
    $$ \tau = (max\_depth - min\_depth) \cdot 0.0001 $$
    *   Typical values for ScanNet: $min\_depth = 0.001$, $max\_depth = 10$ [S1].

**Algorithm Steps:**
1.  **Initialize** the NeRF network with positional encoding as per [Model Architecture and Encoding](13550-model-architecture-and-encoding.md).
2.  **Insert Dropout** layers with $p=0.1$ after the first eight fully connected ReLU layers [S1].
3.  **Set Hyperparameters:**
    *   $\lambda \leftarrow 0.007$ (or $0.02$ for scene 710).
    *   $\gamma \leftarrow 1$.
    *   $lr \leftarrow 5 \times 10^{-4}$.
    *   $lr_{scale} \leftarrow 1 \times 10^{-7}$.
    *   $weight\_decay \leftarrow 1 \times 10^{-6}$.
    *   $total\_steps \leftarrow 500,000$.
4.  **For** each training step $t$ from $1$ to $total\_steps$:
    a. Sample rays from training images.
    b. Render RGB and Depth for each ray using the current network state.
    c. Compute RGB loss against ground truth.
    d. Compute Depth loss against ground truth depth maps.
        *   If using uncertainty weighting, apply threshold $\tau$ derived from scene bounds [S1].
    e. Compute total loss: $L = L_{rgb} + \lambda \cdot \gamma \cdot L_{depth}$ [S1].
    f. Update network weights using Adam optimizer with $lr$ and $weight\_decay$ [S1].
    g. Update depth scale parameters with $lr_{scale}$ [S1].
    h. **Schedule:** If $t > 0.8 \cdot total\_steps$, set $lr \leftarrow 5 \times 10^{-5}$ [S1].
5.  **Return** trained model.

**Edge Cases and Pitfalls:**
*   **Scene 710:** Requires a higher depth loss weight ($\lambda=0.02$) compared to other scenes ($\lambda=0.007$) [S1].
*   **Depth Bounds:** The threshold $\tau$ depends on scene-specific $min\_depth$ and $max\_depth$ values [S1].
*   **Overfitting:** The dropout modification is critical when using depth priors to prevent the network from overfitting to noise in depth maps [S1].

## Subtopics

### DS-NeRF Depth Loss Integration
Integrates the depth loss implementation from the official DS-NeRF code base into the training pipeline [S2]. Due to missing configuration files for DTU in the official release and different train/test splits for LLFF in the original publication, re-training on specific splits (LLFF) using released configuration files is required for reproducibility [S2].

## Operating characteristics
*   **Time Complexity:** Determined by the number of training steps (500,000) and the cost of rendering rays through the modified MLP [S1].
*   **Memory Complexity:** Standard NeRF memory usage plus storage for depth maps and uncertainty thresholds [S1].
*   **Strengths:** Improves geometric accuracy by leveraging depth priors; dropout prevents overfitting to noisy depth data [S1].
*   **Limitations:** Requires ground truth depth maps for training; sensitive to scene-specific depth bounds for threshold calculation [S1].
*   **When to Use:** When high-fidelity geometry is required and depth supervision is available [S1].
*   **When Not to Use:** When depth maps are unavailable or unreliable [S1].

## Background
Originated from the work on Depth-Guided NeRF Training via Earth Mover’s Distance [S1]. The method modifies the original NeRF architecture [15] by adding dropout layers and integrating depth loss with specific hyperparameters [S1]. It is used in contexts requiring improved 3D reconstruction from RGB-D data [S1].

## Sources
[S1] Depth-Guided NeRF Training via Earth Mover’s Distance, https://link.springer.com/chapter/10.1007/978-3-031-73039-9_1, fetched 2026-10-27
[S2] SPARF: Neural Radiance Fields from Sparse and Noisy Poses, https://doi.org/10.1109/cvpr52729.2023.00408, fetched 2026-10-26
