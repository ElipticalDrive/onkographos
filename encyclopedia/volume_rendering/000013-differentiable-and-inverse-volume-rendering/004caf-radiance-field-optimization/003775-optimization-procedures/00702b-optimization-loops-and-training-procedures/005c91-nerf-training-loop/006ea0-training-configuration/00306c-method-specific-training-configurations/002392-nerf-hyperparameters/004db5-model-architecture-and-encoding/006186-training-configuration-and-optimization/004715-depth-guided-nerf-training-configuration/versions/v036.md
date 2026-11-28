# Depth-Guided NeRF Training Configuration

Also called: DäRF Training Configuration, Earth Mover’s Distance Depth-Guided NeRF

## Overview
This configuration defines the training protocol for a NeRF variant that incorporates depth priors to improve reconstruction quality, specifically utilizing an Earth Mover’s Distance (EMD) based depth loss. It distinguishes itself from standard NeRF training by integrating explicit depth supervision and architectural regularization (dropout) to handle uncertainty in depth estimation. It builds on [Training Configuration and Optimization](18409-training-configuration-and-optimization.md).

## How it works
**Inputs:**
*   Camera poses and intrinsics.
*   RGB images.
*   Ground truth depth maps (optional, used for loss calculation and thresholding).
*   Scene-specific depth bounds ($min\_depth$, $max\_depth$).

**Outputs:**
*   Trained NeRF model parameters (MLP weights, embedding tables).

**Data Structures:**
*   **NeRF MLP:** Standard NeRF architecture with positional encoding.
*   **Dropout Layers:** Added after the first 8 fully connected ReLU layers.
*   **Depth Loss Weight ($\lambda$):** Scalar hyperparameter.
*   **Uncertainty Weight ($\gamma$):** Scalar hyperparameter.

**Parameters:**
*   **Optimizer:** Adam.
*   **Initial Learning Rate ($lr$):** $5 \times 10^{-4}$.
*   **LR Schedule:** Reduce to $5 \times 10^{-5}$ after 80% of total steps.
*   **Weight Decay:** $1 \times 10^{-6}$.
*   **Total Steps:** 500,000.
*   **Depth Scale Learning Rate:** $1 \times 10^{-7}$.
*   **Depth Loss Weight ($\lambda$):** $0.007$ for most scenes; $0.02$ for Scene 710.
*   **Uncertainty Weight ($\gamma$):** $1.0$.
*   **Dropout Probability:** $0.1$.

**Algorithm Steps:**
1.  **Initialize** the NeRF model based on the original NeRF architecture [S1].
2.  **Modify Architecture:** Insert dropout layers with probability $p=0.1$ after each of the first eight fully connected ReLU-activated layers that encode the positional encoding [S1].
3.  **Configure Optimizer:** Initialize Adam optimizer with $lr = 5 \times 10^{-4}$ and weight decay $1 \times 10^{-6}$ [S1].
4.  **Set Depth Scale LR:** Assign a learning rate of $1 \times 10^{-7}$ specifically for the depth scale parameter [S1].
5.  **Compute Threshold:** For depth evaluation and uncertainty, compute a threshold $\tau$ using the scene's depth bounds:
    $$ \tau = (max\_depth - min\_depth) \cdot 0.0001 $$
    For ScanNet, typical values are $max\_depth = 10$ and $min\_depth = 0.001$ [S1].
6.  **Training Loop:**
    a. Sample rays and points using stratified sampling (as per shared principles).
    b. Render color and depth.
    c. Compute photometric loss (MSE/L1) between rendered and ground-truth colors.
    d. Compute depth loss using the configured weight $\lambda$ [S1].
    e. Compute uncertainty loss using weight $\gamma = 1$ [S1].
    f. Backpropagate and update weights.
    g. Apply LR schedule: reduce $lr$ to $5 \times 10^{-5}$ after 80% of 500,000 steps [S1].
7.  **Evaluate:** Compute metrics on valid pixels ($K$) where ground truth depth exists:
    $$ RMSE(d, \hat{d}) = \sqrt{\frac{1}{K} \sum_{k=1}^{K} (d_k - \hat{d}_k)^2} $$
    $$ RMSE_{log}(d, \hat{d}) = \sqrt{\frac{1}{K} \sum_{k=1}^{K} (\log d_k - \log \hat{d}_k)^2} $$
    $$ AbsRel(d, \hat{d}) = \frac{1}{K} \sum_{k=1}^{K} \frac{|d_k - \hat{d}_k|}{d_k} $$
    $$ SqRel(d, \hat{d}) = \frac{1}{K} \sum_{k=1}^{K} \frac{(d_k - \hat{d}_k)^2}{d_k} $$
    [S1]

**Edge Cases and Pitfalls:**
*   **Scene 710:** Requires a higher depth loss weight ($\lambda = 0.02$) compared to other scenes ($\lambda = 0.007$) [S1].
*   **Depth Bounds:** The threshold $\tau$ depends on $max\_depth$ and $min\_depth$. Incorrect bounds lead to incorrect uncertainty evaluation [S1].
*   **Overfitting:** The dropout layers are critical for preventing overfitting, especially in sparse settings [S1].

## Subtopics

### DS-NeRF Depth Loss Integration
Integrates the depth loss implementation from the official DS-NeRF code base into the training pipeline. This variant is used when re-training on specific splits (e.g., LLFF) because original publication results used different splits or unreleased configurations. For DTU, official results are reported directly as reproduction failed due to missing config files, suggesting potential undocumented architectural tricks (e.g., reduced PE frequency) were used in the original work [S2].

## Operating characteristics
*   **Time Complexity:** Standard NeRF training complexity, scaled by 500,000 steps [S1].
*   **Memory Complexity:** Standard NeRF memory usage plus overhead for depth map storage and dropout states [S1].
*   **Strengths:** Improved depth accuracy and structural consistency through explicit depth supervision and uncertainty modeling [S1].
*   **Limitations:** Requires ground truth depth maps for training (or reliable pseudo-labels); sensitive to depth scale learning rate [S1].
*   **When to Use:** When depth accuracy is critical and ground truth depth is available or can be reliably estimated [S1].
*   **When Not to Use:** When depth supervision is unavailable or unreliable [S1].

## Background
Originated from the DäRF (Depth-guided NeRF) method, which introduces Earth Mover’s Distance for depth guidance [S1]. The configuration details were provided in the supplementary material of the DäRF paper [S1]. It is used in datasets like ScanNet and LLFF [S1, S2].

## Sources
[S1] Depth-Guided NeRF Training via Earth Mover’s Distance, https://link.springer.com/chapter/10.1007/978-3-031-73039-9_1, fetched 2026-10-27
[S2] SPARF: Neural Radiance Fields from Sparse and Noisy Poses, https://doi.org/10.1109/cvpr52729.2023.00408, fetched 2026-10-26
