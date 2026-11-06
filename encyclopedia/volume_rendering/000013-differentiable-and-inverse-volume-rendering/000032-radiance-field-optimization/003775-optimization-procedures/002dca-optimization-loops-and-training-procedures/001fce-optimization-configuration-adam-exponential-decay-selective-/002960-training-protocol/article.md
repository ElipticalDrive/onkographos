# Training Protocol

Also called: Standard optimization procedure for neural networks in radiance-field optimization

## Overview
The training protocol defines the standard optimization procedure for neural networks in radiance-field optimization, encompassing optimizer settings, learning rate scheduling, and data augmentation strategies [S1]. It addresses the problem of stabilizing training and preventing convergence failure in differentiable rendering pipelines by providing a robust baseline for gradient descent. It is distinguished from its siblings, such as the [Gradient Descent Solver](3280-gradient-descent-solver.md), by its specific configuration of Adam hyperparameters and exponential decay schedules tailored for coordinate-based neural representations [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Neural network parameters (weights and biases), training data (e.g., point clouds, SDF grids, or images), and hyperparameters.
*   **Output:** Optimized neural network parameters representing the radiance field or implicit surface.

**Data Structures:**
*   **Optimizer State:** Maintains first and second moment estimates for Adam.
*   **Learning Rate:** A scalar value updated periodically.
*   **Training Loop State:** Epoch or step counter.

**Parameters:**
*   **Optimizer:** Adam [Kingma and Ba 2015] [S1].
*   **Beta 1 ($\beta_1$):** 0.9 [S1].
*   **Beta 2 ($\beta_2$):** 0.999 [S1].
*   **Initial Learning Rate ($\eta_0$):** 0.0001 [S1].
*   **Decay Interval:** 100 epochs [S1].
*   **Decay Factor:** 0.5 (halving) [S1].
*   **Gaussian Noise Standard Deviation ($\sigma$):** 0.5 (assuming unit cube cells) [S1].
*   **Loss Functions:** Same as UNDC (Unconstrained Neural Dual Contouring) [S1].

**Steps:**
1.  **Initialize** the neural network weights randomly or via a learned initialization [S1][S2, unofficial].
2.  **Set** the initial learning rate $\eta = 0.0001$ [S1].
3.  **For** each epoch (or step) from 1 to $E_{total}$:
    a. **Sample** a batch of training data (e.g., points from a point cloud or grid cells) [S1].
    b. **Apply Data Augmentation** to the input data:
        i. Apply random scaling and translation [S1].
        ii. Add Gaussian noise with standard deviation $\sigma = 0.5$ to input point clouds to simulate real scan noise [S1].
    c. **Forward Pass:** Compute the network output (e.g., density, color, or SDF) for the sampled inputs [S1].
    d. **Compute Loss:** Calculate the loss using the same functions as UNDC [S1].
    e. **Backward Pass:** Compute gradients with respect to the network parameters [S1].
    f. **Update Parameters:** Perform an Adam update step using $\beta_1=0.9$, $\beta_2=0.999$, and current learning rate $\eta$ [S1].
    g. **Update Learning Rate:** If the current epoch is a multiple of 100, update $\eta \leftarrow \eta / 2$ [S1].
4.  **Terminate** after $E_{total}$ epochs (400 for SDF/voxel inputs, 250 for point cloud inputs) [S1].

**Edge Cases and Pitfalls:**
*   **Noisy Real Scans:** For reconstruction from noisy real scans, the standard 3x3x3 conv3d layers should be replaced with 8 residual blocks to handle the increased complexity and noise [S1].
*   **Precision:** The learning rate decay is discrete (every 100 epochs), which may cause slight instability if the optimal learning rate lies between decay steps [G].
*   **Noise Simulation:** The Gaussian noise is applied to the input point clouds, not the output, to simulate acquisition errors [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def train_network(network, data_loader, epochs=400, batch_size=1):
    # Initialize Adam optimizer with specified hyperparameters
    optimizer = Adam(network.parameters(), lr=0.0001, betas=(0.9, 0.999))
    
    current_lr = 0.0001
    optimizer.param_groups[0]['lr'] = current_lr
    
    for epoch in range(1, epochs + 1):
        for batch in data_loader:
            # Apply data augmentation
            # 1. Random scaling and translation
            augmented_input = apply_random_scaling_and_translation(batch.input)
            
            # 2. Add Gaussian noise to point clouds (sigma=0.5)
            # Assuming batch.input is a point cloud
            noise = torch.randn_like(augmented_input) * 0.5
            augmented_input = augmented_input + noise
            
            # Forward pass
            optimizer.zero_grad()
            output = network(augmented_input)
            
            # Compute loss (same as UNDC)
            loss = compute_undc_loss(output, batch.target)
            
            # Backward pass
            loss.backward()
            
            # Update parameters
            optimizer.step()
        
        # Learning rate decay every 100 epochs
        if epoch % 100 == 0:
            current_lr /= 2
            for param_group in optimizer.param_groups:
                param_group['lr'] = current_lr

def train_noisy_scan_network(network, data_loader, epochs=400):
    # Replace 3x3x3 conv3d layers with 8 residual blocks
    network = replace_conv3d_with_residual_blocks(network, num_blocks=8)
    train_network(network, data_loader, epochs=epochs)
```

## Subtopics

### Learned Initializations for Neural Fields
A technique to address the inefficiency of optimizing a coordinate-based network from randomly initialized weights for each new signal by applying standard meta-learning algorithms to learn the initial weight parameters based on the underlying class of signals [S2, unofficial]. It is distinguished from the standard protocol by replacing random initialization with meta-learned priors. It should be chosen when faster convergence and better generalization from partial observations are required [S2, unofficial]. The steps involve applying meta-learning algorithms to learn initial weights for fully-connected networks based on signal classes (e.g., faces, chairs) and using these weights to start optimization [S2, unofficial].

### Pilot Network for Early Training
A small neural network used during the early training stage to aid the model in finding coarse geometry, preventing convergence failure caused by feature integration of all samples [S3]. It is distinguished by its temporary use and small size. It should be chosen when the main network fails to converge in early stages due to complex feature integration [S3]. The steps involve using a pilot network with two hidden layers of 64 neurons each for the first 300 training steps, then discarding it and switching to the main VFR framework for the remaining steps (e.g., up to 20K steps) [S3].

### Training Protocol with Augmented Backgrounds and Density Prior
A variant that introduces background augmentation and density priors to stabilize optimization and refine scene representation [S4]. It is distinguished by the phased introduction of these regularizations. It should be chosen when background augmentation causes instability or when density bias needs to be enforced [S4]. The steps involve: 1) Using constant color backgrounds (random gray) for the first 25 steps [S4]. 2) Switching to background augmentation after step 25 [S4]. 3) Applying density prior regularization starting at step 100 to refine density bias [S4]. Parameters include density regularization weight $\lambda_1 = 2 \cdot 10^{-5}$ and color regularization weight $8 \cdot 10^{-4}$ [S4].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of epochs and batch size. The learning rate decay adds negligible overhead [S1].
*   **Memory Complexity:** Depends on the network architecture and batch size. The Adam optimizer requires additional memory for first and second moment estimates [S1].
*   **Measured Performance:** Trained for 400 epochs for SDF/voxel inputs and 250 epochs for point cloud inputs [S1].
*   **Strengths:** Robust convergence due to Adam optimizer and learning rate decay; effective handling of noisy data through augmentation [S1].
*   **Limitations:** Fixed decay schedule may not be optimal for all datasets; requires careful tuning of augmentation parameters [S1].
*   **When to Use:** Standard training for neural dual contouring and similar radiance-field methods [S1].
*   **When Not to Use:** When faster convergence is needed (use Learned Initializations) or when convergence fails early (use Pilot Network) [S2, unofficial][S3].

## Background
The training protocol originates from the Neural Dual Contouring method [S1]. It builds on the Adam optimizer [Kingma and Ba 2015] and incorporates techniques from UNDC [S1]. The protocol has been adapted for various tasks including SDF reconstruction, point cloud reconstruction, and noisy real scan reconstruction [S1].

## Sources
[S1] Neural Dual Contouring, https://arxiv.org/abs/2202.01999v3, fetched 2026-10-26
[S2] Generalizing Neural Radiance Fields - BAIR Commons, https://bcommons.berkeley.edu/generalizing-neural-radiance-fields, fetched 2026-10-28 (unofficial)
[S3] Volume Feature Rendering for Fast Neural Radiance Field Reconstruction, https://doi.org/10.52202/075280-2853, fetched 2026-10-26
[S4] Text-based transfer function design for semantic volume rendering, https://doi.org/10.1109/vis55277.2024.00047, fetched 2026-10-26
