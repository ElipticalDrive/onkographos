# Warm-up Strategy

Also called: Encoder-freezing Warm-up, Warm-up

## Overview
A training phase that prevents the model from overfitting to the input view and ensures fine details are captured during distillation by temporarily freezing or adjusting objectives before full joint training [S1]. It serves as a stabilization mechanism for radiance-field optimization, particularly when introducing perceptual or adversarial losses that might otherwise cause mode collapse or weak distillation if applied immediately [S1, S2].

## How it works
The method involves a two-stage initialization process where the optimization trajectory is constrained to a simpler loss landscape before introducing complex perceptual or adversarial objectives.

**Inputs and Outputs:**
*   **Input:** A scene representation (e.g., NeRF or 3D Gaussian Splatting) initialized from a single image or sparse views; a dataset of training images.
*   **Output:** An optimized scene representation ready for full joint training or novel view synthesis.

**Data Structures:**
*   **Optimizer State:** Standard optimizer state (e.g., Adam moments) initialized at the start of the warm-up phase.
*   **Loss Weights:** A binary switch or weight parameter $\lambda_{wd}$ that is $0$ during warm-up and $>0$ after warm-up.

**Steps:**
1.  **Initialize:** Set the scene parameters (e.g., neural network weights or Gaussian attributes) to their initial values derived from the encoder or reconstruction step. Initialize the optimizer.
2.  **Determine Warm-up Duration ($T_{warm}$):**
    *   For standard datasets (e.g., Deep Blending, Mip-NeRF 360, Tanks & Temples): $T_{warm} = 3,000$ iterations [S2].
    *   For large-scale cityscape datasets with large viewpoint/geometry variations and inconsistent appearance (e.g., BungeeNeRF): $T_{warm} = 5,000$ iterations [S2].
3.  **Warm-up Phase (Iterations $t < T_{warm}$):**
    *   Compute the base reconstruction loss $L_{orig}$ (e.g., standard 3DGS loss or pixel-wise MSE/SSIM) [S2].
    *   If using the encoder-freezing variant (Pix2NeRF context): Freeze the encoder parameters. The decoder/field parameters are updated to minimize $L_{orig}$ [S1].
    *   If using the WD Loss variant (3DGS context): Minimize only $L_{orig}$. The perceptual/WD loss $L_{wd}$ is not included in the total loss [S2].
    *   Update parameters using gradient descent on $L_{total} = L_{orig}$ [S2].
4.  **Transition Phase (Iteration $t = T_{warm}$):**
    *   Unfreeze the encoder (if applicable) [S1].
    *   Enable the perceptual/WD loss component [S2].
5.  **Full Training Phase (Iterations $t \ge T_{warm}$):**
    *   Compute the total loss. For WD-based methods: $L_{total} = L_{orig} + \gamma L_{wd}$ (or similar combination), where $L_{orig}$ acts as a mild regularizer [S2]. For Pix2NeRF: Jointly optimize encoder and field parameters with reconstruction and adversarial objectives [S1].
    *   Update parameters using gradient descent on $L_{total}$ [S1, S2].

**Formulas:**
*   **Total Loss (WD Variant):**
    $$ L_{total} = \begin{cases} L_{orig} & \text{if } t < T_{warm} \\ L_{orig} + \gamma L_{wd} & \text{if } t \ge T_{warm} \end{cases} $$
    Where $L_{orig}$ is the original rendering loss [S2], $L_{wd}$ is the perceptual WD loss, and $\gamma$ is a scaling coefficient calibrated per dataset [S2].
*   **Total Loss (Pix2NeRF Variant):**
    During warm-up, the encoder is frozen, effectively minimizing only the reconstruction component relative to the fixed encoder output [S1].

**Parameters:**
*   **Warm-up Iterations ($T_{warm}$):** 3,000 for most datasets; 5,000 for BungeeNeRF [S2].
*   **Reconstruction Weight ($\lambda_{recon}$):** Must not be lowered to 1 in place of warm-up, as this breaks the balance between reconstruction and adversarial objectives [S1].

**Edge Cases and Pitfalls:**
*   **Mode Collapse:** Replacing warm-up with a lower reconstruction weight ($\lambda_{recon} = 1$) causes mode collapse for novel view synthesis because the balance between reconstruction and adversarial objectives is missing [S1].
*   **Weak Distillation:** If the encoder is never unfrozen after warm-up, the distillation remains relatively weak, resulting in few fine details [S1].
*   **Overfitting:** Without warm-up, the model tends to overfit the input view and cannot produce meaningful content from novel poses [S1].
*   **Instability:** For datasets with large-scale variations (e.g., BungeeNeRF), skipping warm-up or using insufficient iterations leads to instability caused by pruning and densification heuristics [S2].

## Subtopics

### Warm-up Strategy for WD Loss
A training initialization phase that uses the original 3DGS loss before introducing the perceptual WD objective to stabilize optimization, especially for datasets with large viewpoint/geometry variations [S2]. It is chosen when using WD-based losses (WD or WD-R) to prevent instability from pruning/densification heuristics and to provide a stable geometric initialization [S2]. It differs from the general encoder-freezing warm-up by focusing on loss landscape stability rather than encoder parameter constraints [S2].

## Operating characteristics
*   **Time Complexity:** Adds $T_{warm}$ iterations of training to the total training time. For standard datasets, this is 3,000 iterations; for BungeeNeRF, 5,000 iterations [S2].
*   **Memory Complexity:** No additional memory overhead compared to standard training, as it uses the same data structures and optimizer state [S2].
*   **Performance:** Prevents overfitting to input views and mode collapse [S1]. Ensures fine details are captured by allowing the model to first learn a stable geometric base [S1, S2].
*   **Strengths:** Stabilizes optimization for perceptual losses; prevents mode collapse; improves novel view synthesis quality [S1, S2].
*   **Limitations:** Increases total training time by the warm-up duration [S2]. Requires careful tuning of $T_{warm}$ for specific datasets (e.g., longer for BungeeNeRF) [S2].
*   **When to Use:** When training radiance fields with perceptual/adversarial losses (e.g., Pix2NeRF, WD-3DGS) to avoid overfitting or instability [S1, S2].
*   **When Not to Use:** When using only simple reconstruction losses without perceptual components, or when computational budget is extremely tight and fine details are not critical [S1, S2].

## Background
The warm-up strategy originated in the context of unsupervised single-shot novel view synthesis, specifically in the Pix2NeRF framework [S1]. It was introduced to address the tendency of GAN-based distillation methods to overfit to the input image or collapse into trivial solutions when adversarial and reconstruction losses are balanced improperly [S1]. Later, similar warm-up concepts were adopted in 3D Gaussian Splatting optimization to stabilize the introduction of perceptual losses (WD loss) in large-scale cityscape datasets [S2].

## Sources
[S1] Pix2NeRF: Unsupervised Conditional $\pi$-GAN for Single Image to Neural Radiance Fields Translation, https://doi.org/10.1109/cvpr52688.2022.00395, fetched 2026-10-29
[S2] Drop-In Perceptual Optimization for 3D Gaussian Splatting, https://doi.org/10.1007/978-3-032-37350-2_25, fetched 2026-10-30
