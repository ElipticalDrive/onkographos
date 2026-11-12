# ResNet-Based CT Reconstruction
Also called: CNN Reconstruction, Deep Learning CT Reconstruction, Learned Post-Processing, Learned Post-Processing with ResNet

## Overview
ResNet-Based CT Reconstruction is a learned post-processing method that maps noisy filtered backprojection (FBP) reconstructions directly to ground truth images, learning to suppress artifacts and noise [S1, unofficial]. It differs from pure post-processing methods by using the physics of the acquisition geometry to guide the denoising and artifact suppression [S1, unofficial]. Unlike Gradient-Aware ResNet Reconstruction, it treats the input as a static noisy image rather than conditioning on the explicit Radon matrix [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   Input: Noisy FBP reconstruction $f_{BP}$ (2D image tensor).
*   Output: Cleaned reconstruction $f_{pred}$ (2D image tensor).

**Data Structures:**
*   `ResNet`: A convolutional neural network model instance.
*   `inputs`: Tensor of shape `(batch_size, channels, height, width)`.
*   `targets`: Tensor of shape `(batch_size, channels, height, width)` containing ground truth images.

**Training Procedure:**
1.  **Dataset Generation:**
    *   Create synthetic ground truth images $f_{true}$ consisting of 5 to 20 overlapping ellipses of varying sizes [S1, unofficial].
    *   Generate sinograms from $f_{true}$ using a projector and projection geometry [S1, unofficial].
    *   Add noise to sinograms (e.g., using `astra.functions.add_noise_to_sino` with a specific noise level) [S1, unofficial].
    *   Reconstruct noisy sinograms using filtered backprojection (FBP) to obtain $f_{BP}$ [S1, unofficial].
2.  **Model Initialization:** Initialize the `ResNet()` model [S1, unofficial].
3.  **Loss Function:** Define the loss as Mean Squared Error (MSE) between the network output and the ground truth: $L = \text{MSE}(f_{pred}, f_{true})$ [S1, unofficial].
4.  **Optimization:** Use the Adam optimizer with a learning rate of 0.001 [S1, unofficial].
5.  **Training Loop:**
    *   Iterate for `num_epochs` (default 10) [S1, unofficial].
    *   For each batch:
        *   Forward pass: `outputs = model(inputs.float())` [S1, unofficial].
        *   Compute loss: `loss = criterion(outputs, targets.float())` [S1, unofficial].
        *   Backpropagate: `loss.backward()` [S1, unofficial].
        *   Update weights: `optimizer.step()` [S1, unofficial].
6.  **Evaluation:** Evaluate on a validation set [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.optim as optim

# Assume ResNet is defined elsewhere or imported
# class ResNet(nn.Module): ...

def train_resnet_ct_reconstruction(model, train_loader, val_loader, num_epochs=10, batch_size=32, lr=0.001):
    """
    Trains a ResNet for CT reconstruction post-processing.
    
    Args:
        model: ResNet instance.
        train_loader: DataLoader yielding (inputs, targets) tuples.
        val_loader: DataLoader for validation.
        num_epochs: Number of training epochs.
        batch_size: Batch size for training.
        lr: Learning rate for Adam optimizer.
    """
    criterion = nn.MSELoss()
    optimizer = optim.Adam(model.parameters(), lr=lr)
    
    model.train()
    for epoch in range(num_epochs):
        for inputs, targets in train_loader:
            # inputs: noisy FBP reconstructions
            # targets: ground truth images
            
            # Forward pass
            outputs = model(inputs.float())
            
            # Compute loss
            loss = criterion(outputs, targets.float())
            
            # Backward pass and optimization
            optimizer.zero_grad()
            loss.backward()
            optimizer.step()
            
        # Validation step
        model.eval()
        val_loss = 0.0
        with torch.no_grad():
            for inputs, targets in val_loader:
                outputs = model(inputs.float())
                val_loss += criterion(outputs, targets.float()).item()
        val_loss /= len(val_loader)
        print(f"Epoch {epoch+1}/{num_epochs}, Val Loss: {val_loss:.4f}")
        model.train()

# Example usage for synthetic data generation (conceptual)
def generate_synthetic_dataset():
    """
    Generates synthetic dataset of (f_BP, f_true) pairs.
    """
    # 1. Create ground truth: 5-20 overlapping ellipses
    f_true = create_ellipse_phantom(num_ellipses=10)
    
    # 2. Generate sinograms
    proj_geom, projector_id, g = create_sinogram(f_true, n_angles=180, max_angle=np.pi, n_samples=512)
    
    # 3. Add noise to sinograms
    g_noisy = astra.functions.add_noise_to_sino(g, noise_level=10000)
    g_noisy_id = astra.data2d.create('-sino', proj_geom, g_noisy)
    
    # 4. Reconstruct using FBP (or other initial method)
    # Note: The source uses gmres_solver for some examples, but the method description
    # specifies mapping from FBP. We assume FBP reconstruction here.
    f_bp = filtered_backprojection(g_noisy_id, proj_geom)
    
    return f_bp, f_true
```

**Parameters:**
*   `lr`: Learning rate, default `0.001` [S1, unofficial].
*   `batch_size`: Training batch size, default `32` [S1, unofficial].
*   `num_epochs`: Number of training epochs, default `10` [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** Inputs and targets should be cast to float32 (`inputs.float()`) to ensure numerical stability during gradient computation [S1, unofficial].
*   **Boundary Conditions:** The synthetic data generation must ensure that the ellipses and noise levels are representative of the target domain to avoid poor generalization [S1, unofficial].
*   **Degenerate Input:** If the input FBP reconstruction is severely corrupted (e.g., missing data beyond what FBP can handle), the ResNet may fail to recover the ground truth, as it relies on the initial reconstruction containing sufficient structural information [S1, unofficial].

## Subtopics

### [Gradient-Aware ResNet Reconstruction](8269-gradient-aware-resnet-reconstruction.md)
Extends standard ResNet by conditioning on the explicit Radon matrix $A$, incorporating geometric projection constraints directly during reconstruction.

## Operating characteristics
*   **Time Complexity:** Training time depends on the number of epochs, batch size, and network depth. Inference is fast, typically one forward pass through the ResNet [S1, unofficial].
*   **Memory Complexity:** Memory usage is dominated by the ResNet parameters and the batch size during training [S1, unofficial].
*   **Measured Performance:** The method reduces MSE compared to standard FBP or Tikhonov regularization, as demonstrated on synthetic phantom data [S1, unofficial].
*   **Strengths:** Effective at suppressing noise and artifacts in FBP reconstructions; end-to-end differentiable training allows for joint optimization [S1, unofficial].
*   **Limitations:** Requires a large dataset of paired noisy/clean images for training; generalization to unseen anatomy or acquisition geometries may be limited if not included in training data [S1, unofficial].
*   **When to Use:** When computational resources for iterative reconstruction are limited, and a pre-trained model can be applied for fast denoising [S1, unofficial].
*   **When Not to Use:** When the acquisition geometry differs significantly from the training data, or when quantitative accuracy is critical and the model's bias is unknown [S1, unofficial].

## Background
The method originates from the use of convolutional neural networks (CNNs) for medical image reconstruction, specifically leveraging ResNet architectures for artifact suppression in CT [S1, unofficial]. It builds on the concept of learned post-processing, where traditional reconstruction methods (like FBP) are used to generate an initial estimate, which is then refined by a deep learning model [S1, unofficial]. The specific implementation cited uses synthetic data with ellipsoidal phantoms to train the network [S1, unofficial].

## Sources
[S1] imanfs/inverse-problems-CT-scans, https://github.com/imanfs/inverse-problems-CT-scans, fetched 2026-10-27 (unofficial)
