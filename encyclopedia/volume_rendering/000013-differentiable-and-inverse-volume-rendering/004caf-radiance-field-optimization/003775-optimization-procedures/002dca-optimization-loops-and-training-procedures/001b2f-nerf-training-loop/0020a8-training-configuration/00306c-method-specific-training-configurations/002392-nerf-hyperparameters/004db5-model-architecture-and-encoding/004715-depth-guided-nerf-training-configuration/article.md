# Depth-Guided NeRF Training Configuration

Also called: Depth-Guided NeRF Training

## Overview
A training configuration for Neural Radiance Fields (NeRF) that incorporates depth priors to guide optimization, addressing the ambiguity of monocular depth estimation and improving geometry reconstruction [S1]. It distinguishes itself from standard NeRF training by integrating a depth loss term with specific uncertainty weighting and architectural regularization (dropout) to prevent overfitting on noisy depth inputs [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** RGB images, camera poses, ground truth depth maps (optional but used for loss), and a threshold $\tau$ for depth validity.
*   **Outputs:** Optimized NeRF model parameters (weights and biases) and depth scale parameter.

**Data Structures:**
*   **NeRF Network:** Builds upon the original NeRF architecture [S1].
    *   **Modification:** Dropout layers with probability $p=0.1$ are added after each of the first eight fully connected ReLU-activated layers that encode the positional encoding [S1].
*   **Depth Threshold:** A scalar $\tau$ used to evaluate the uncertainty of a depth map [S1].

**Parameters:**
*   **Depth loss weight ($\lambda$):** $0.007$ for all scenes except scene 710, where it is $0.02$ [S1].
*   **Uncertainty weight ($\gamma$):** $1$ [S1].
*   **Optimizer:** Adam [S1].
*   **Initial learning rate:** $5 \times 10^{-4}$ [S1].
*   **Learning rate schedule:** Reduce to $5 \times 10^{-5}$ after 80% of total steps [S1].
*   **Weight decay:** $1 \times 10^{-6}$ [S1].
*   **Total training steps:** $500,000$ [S1].
*   **Depth scale learning rate:** $1 \times 10^{-7}$ [S1].
*   **Threshold ($\tau$):** Calculated as $\tau = (\text{max\_depth} - \text{min\_depth}) \cdot 0.0001$ [S1]. For ScanNet, $\text{max\_depth} = 10$ and $\text{min\_depth} = 0.001$ [S1].

**Numbered Steps:**
1.  **Initialize** the NeRF network with the modified architecture (dropout after first 8 FC layers) [S1].
2.  **Calculate Threshold:** Compute $\tau$ based on the scene's depth range [S1].
3.  **Optimize:** Use the Adam optimizer with the specified learning rates and weight decay [S1].
4.  **Loss Computation:** Combine the standard NeRF rendering loss with the depth loss weighted by $\lambda$ and $\gamma$ [S1].
5.  **Schedule:** Reduce the main learning rate to $5 \times 10^{-5}$ after 80% of the $500,000$ steps [S1].
6.  **Terminate:** After $500,000$ steps [S1].

**Formulas:**
The depth evaluation metrics (used for validation/loss monitoring) are computed over pixels $k=1,...,K$ with valid ground truth depth [S1]:

$$
\text{RMSE}(d, \hat{d}) = \sqrt{\frac{1}{K} \sum_{k=1,...,K} (d_k - \hat{d}_k)^2}
$$

$$
\text{RMSE}_{\log}(d, \hat{d}) = \sqrt{\frac{1}{K} \sum_{k=1,...,K} (\log d_k - \log \hat{d}_k)^2}
$$

$$
\text{AbsRel}(d, \hat{d}) = \frac{1}{K} \sum_{k=1,...,K} \frac{|d_k - \hat{d}_k|}{d_k}
$$

$$
\text{SqRel}(d, \hat{d}) = \frac{1}{K} \sum_{k=1,...,K} \frac{(d_k - \hat{d}_k)^2}{d_k}
$$

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Configuration
TOTAL_STEPS = 500000
LR_INITIAL = 5e-4
LR_FINAL = 5e-5
LR_DEPTH_SCALE = 1e-7
WEIGHT_DECAY = 1e-6
LAMBDA_DEPTH = 0.007  # Use 0.02 for scene 710
GAMMA_UNCERTAINTY = 1
DROPOUT_PROB = 0.1
MAX_DEPTH = 10.0
MIN_DEPTH = 0.001

def compute_threshold(max_depth, min_depth):
    return (max_depth - min_depth) * 0.0001

def build_nerf_model():
    model = OriginalNeRF()
    # Add dropout after first 8 FC layers encoding positional encoding
    for i, layer in enumerate(model.positional_encoding_layers[:8]):
        if isinstance(layer, FullyConnectedReLU):
            model.insert_dropout_after(layer, prob=DROPOUT_PROB)
    return model

def train_depth_guided_nerf(images, poses, depth_maps, scene_id='default'):
    model = build_nerf_model()
    optimizer = Adam(model.parameters(), lr=LR_INITIAL, weight_decay=WEIGHT_DECAY)
    
    # Separate optimizer for depth scale if applicable
    # Note: Specific implementation of depth scale parameter depends on architecture
    # Assuming it is part of the model parameters but with different LR
    # In practice, this might require a parameter group or separate optimizer
    # For simplicity, we assume the main optimizer handles it with a global LR
    # or that the depth scale is a specific parameter with a custom LR schedule.
    # The source specifies a distinct LR for depth scale, implying a separate 
    # parameter group or optimizer.
    
    depth_scale_optimizer = Adam(
        [p for p in model.parameters() if p.name == 'depth_scale'], 
        lr=LR_DEPTH_SCALE
    )
    
    threshold = compute_threshold(MAX_DEPTH, MIN_DEPTH)
    
    for step in range(TOTAL_STEPS):
        # Learning rate schedule
        if step > TOTAL_STEPS * 0.8:
            current_lr = LR_FINAL
        else:
            current_lr = LR_INITIAL
            
        optimizer.param_groups[0]['lr'] = current_lr
        
        # Forward pass
        rgb_pred, depth_pred, uncertainty = model(ray_samples)
        
        # Compute losses
        rgb_loss = compute_rgb_loss(rgb_pred, rgb_gt)
        
        # Depth loss integration
        # The source mentions integrating depth loss from DS-NeRF [S2]
        # and using uncertainty weight gamma [S1]
        depth_loss = compute_depth_loss(depth_pred, depth_gt, threshold)
        
        total_loss = rgb_loss + LAMBDA_DEPTH * GAMMA_UNCERTAINTY * depth_loss
        
        # Backward pass
        optimizer.zero_grad()
        depth_scale_optimizer.zero_grad()
        total_loss.backward()
        optimizer.step()
        depth_scale_optimizer.step()
        
        # Validation metrics (optional, for monitoring)
        # RMSE, RMSE_log, AbsRel, SqRel computed on valid pixels
```

**Edge Cases and Pitfalls:**
*   **Scene 710:** Requires a higher depth loss weight ($\lambda = 0.02$) compared to other scenes ($\lambda = 0.007$) [S1].
*   **Threshold Calculation:** The threshold $\tau$ is scene-dependent and must be calculated using the specific max/min depth values of the dataset (e.g., ScanNet) [S1].
*   **Overfitting:** The dropout layers are critical to prevent overfitting, especially when using noisy depth priors [S1].

## Subtopics

### DS-NeRF Depth Loss Integration
Integrates the depth loss implementation from the official DS-NeRF code base into the training pipeline [S2]. Due to missing configuration files for DTU in the official release and a different train/test split used in the original LLFF publication, re-training on specific splits (LLFF train/test) is required to reproduce results [S2].

## Operating characteristics
*   **Time Complexity:** Proportional to the number of training steps ($500,000$) and the cost of depth loss computation [S1].
*   **Memory Complexity:** Standard NeRF memory usage plus storage for depth maps and uncertainty maps [S1].
*   **Strengths:** Improves geometry reconstruction by leveraging depth priors; dropout prevents overfitting to noisy depth [S1].
*   **Limitations:** Requires ground truth depth maps; performance is sensitive to the depth loss weight $\lambda$ and the threshold $\tau$ [S1].
*   **When to use:** When depth priors are available and geometry accuracy is critical [S1].
*   **When not to use:** When depth maps are unavailable or highly unreliable without careful thresholding [S1].

## Background
Originated from the Depth-Guided NeRF method, which uses Earth Mover’s Distance to align predicted depth with ground truth [S1]. The configuration details (dropout, learning rates, weights) were established in experiments on ScanNet and other datasets [S1]. The integration of DS-NeRF's depth loss highlights the modular nature of these configurations [S2].

## Sources
[S1] Depth-Guided NeRF Training via Earth Mover’s Distance, https://link.springer.com/chapter/10.1007/978-3-031-73039-9_1, fetched 2026-10-27
[S2] SPARF: Neural Radiance Fields from Sparse and Noisy Poses, https://doi.org/10.1109/cvpr52729.2023.00408, fetched 2026-10-26
