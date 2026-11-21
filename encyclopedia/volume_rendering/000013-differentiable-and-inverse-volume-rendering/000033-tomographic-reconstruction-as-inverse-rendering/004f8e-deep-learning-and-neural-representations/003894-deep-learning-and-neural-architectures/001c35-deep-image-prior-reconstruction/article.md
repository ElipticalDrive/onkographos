# Deep Image Prior Reconstruction

Also called: DIP Reconstruction, DRP Denoising (Deep Image Prior)

## Overview
Deep Image Prior Reconstruction is an iterative tomographic reconstruction method that uses a neural network as an implicit regularizer to optimize an intermediate latent variable and network weights against measured sinogram data [S1, unofficial]. It distinguishes itself from supervised neural architectures by decoupling the latent variable update from the network weight optimization, allowing the network architecture itself to act as a dynamic prior without external training data [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A sparse or noisy sinogram $p$ (measured projection data) [S1, unofficial].
*   **Output:** A reconstructed 3D volume estimate $x$ [S1, unofficial].

**Data Structures:**
The method maintains the following state variables:
1.  **PriorModel:** A neural network that maps a latent tensor to the volume space [S1, unofficial].
2.  **Latent Variable $z$:** An intermediate tensor updated via the data consistency term [S1, unofficial].
3.  **Projector:** An object implementing the forward projection operator $\mathcal{P}$ and its adjoint (back-projection) $\mathcal{P}^T$ [S1, unofficial].
4.  **Optimizer:** An Adam optimizer instance managing the gradients of the PriorModel parameters [S1, unofficial].
5.  **Loss:** An MSE loss function [S1, unofficial].

**Algorithm Steps:**
The reconstruction proceeds in an alternating optimization loop consisting of outer epochs and inner steps [S1, unofficial].

1.  **Initialization:**
    *   Initialize the PriorModel if not provided and move it to the device [S1, unofficial].
    *   Initialize the Adam optimizer with the PriorModel's parameters and a learning rate $lr$ [S1, unofficial].
    *   Compute an initial estimate of the volume by applying Filtered Back Projection (FBP) to the sinogram $p$ and detaching it from the computation graph: $z = \text{FBP}(p).\text{detach}()$ [S1, unofficial].
    *   Set hyperparameters: number of outer epochs $E$, number of inner steps $K$, and latent update step size $\beta$ [S1, unofficial].

2.  **Outer Loop (Epochs):**
    For each epoch from $0$ to $E-1$:
    *   Compute the current volume estimate: $x = \text{PriorModel}(z)$.
    *   Compute the data consistency residual: $\delta = \text{projector.backward}(p - \text{projector.forward}(x))$.
    *   Update the latent variable: $z = x + \beta \cdot \delta$.

3.  **Inner Loop (Network Weight Optimization):**
    For each inner step from $0$ to $K-1$:
    *   Compute the forward projection of the current volume estimate: $\hat{p} = \text{projector.forward}(\text{PriorModel}(z))$.
    *   Compute the Mean Squared Error (MSE) loss between the predicted projections and the measured sinogram: $\mathcal{L} = \text{MSE}(\hat{p}, p)$.
    *   Perform backpropagation to compute gradients with respect to the PriorModel parameters.
    *   Perform one Adam optimizer step to update the parameters.

4.  **Final Output:**
    After completing all epochs, the final reconstruction is obtained by passing the final latent variable $z$ through the prior network: $x_{\text{final}} = \text{PriorModel}(z)$ [S1, unofficial].

**Parameters:**
*   $E$ (epochs): Number of outer optimization epochs. Default: 250 [S1, unofficial].
*   $K$ (inner_steps): Number of Adam optimization steps per outer epoch. Default: 50 [S1, unofficial].
*   $\beta$ (beta/step_size): Step size for the latent variable update. Default: 0.25 [S1, unofficial].
*   $lr$: Learning rate for the Adam optimizer. Default not specified in the sources [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** The FBP initialization must be detached from the computation graph to prevent gradients from flowing through it during the latent update [S1, unofficial].
*   **Boundaries:** The projector must handle rays passing outside the volume bounds correctly, typically by contributing zero to the integral [G].
*   **Degenerate Input:** If the sinogram $p$ is extremely sparse or noisy, the FBP initialization may be poor, potentially leading to slow convergence or local minima [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.optim as optim

def deep_image_prior_reconstruction(sinogram_p, projector, prior_model=None, 
                                    epochs=250, inner_steps=50, beta=0.25, lr=0.01):
    """
    Reconstructs a volume from sinogram data using Deep Image Prior.
    
    Args:
        sinogram_p: Tensor of measured projections.
        projector: Object with methods forward(x) and backward(residual).
        prior_model: Neural network mapping latent z to volume x. If None, initialized.
        epochs: Number of outer epochs.
        inner_steps: Number of inner Adam steps per epoch.
        beta: Step size for latent variable update.
        lr: Learning rate for Adam optimizer.
        
    Returns:
        Reconstructed volume tensor.
    """
    # [G] Standard U-Net initialization
    if prior_model is None:
        prior_model = nn.Sequential(
            nn.Conv3d(1, 64, 3, padding=1),
            nn.ReLU(),
            nn.Conv3d(64, 1, 3, padding=1)
        )
    prior_model.train()
    optimizer = optim.Adam(prior_model.parameters(), lr=lr)
    
    # [G] Standard FBP algorithm: filter sinograms and back-project into 3D volume
    z = fbp(sinogram_p).detach()
    
    loss_fn = nn.MSELoss()
    
    for _ in range(epochs):
        # Outer Loop: Update Latent Variable z
        x = prior_model(z)
        residual = sinogram_p - projector.forward(x)
        delta = projector.backward(residual)
        z = x + beta * delta
        
        # Inner Loop: Optimize PriorModel weights
        for _ in range(inner_steps):
            p_hat = projector.forward(prior_model(z))
            loss = loss_fn(p_hat, sinogram_p)
            optimizer.zero_grad()
            loss.backward()
            optimizer.step()
            
    # Final reconstruction
    output = prior_model(z)
    return output

def fbp(sinogram):
    # [G] Standard FBP algorithm: filter sinograms and back-project into 3D volume
    raise NotImplementedError("FBP implementation required")
```

## Subtopics

### Unsupervised Gradient-Domain Neural Reconstruction
This variant, proposed by Xu et al., proposes an unsupervised neural network for image reconstruction of gradient-domain volumetric photon density estimation, specifically for volumetric photon mapping [S2]. It utilizes a variant of GradNet with an encoded shift connection and a separated auxiliary feature branch that includes volume-based auxiliary features such as transmittance and photon density [S2]. The network is designed to smooth images at a global scale while preserving high-frequency details on a small scale [S2]. It produces higher-quality results than previous works such as L1, L2, or standard GradNet [S2]. However, it cannot reconstruct images with strong noise and tends to over-blur some features [S2]. While originally developed for volumetric photon mapping, the method is noted as straightforward to extend to other tasks like beam radiance estimation [S2].

## Operating characteristics
*   **Time Complexity:** The method is computationally intensive due to the alternating optimization loop [S1, unofficial]. Each epoch involves one forward/backward pass for the latent update and $K$ forward/backward passes for the network weight update [S1, unofficial]. The total number of forward projections per epoch is $K+1$ [S1, unofficial].
*   **Memory Complexity:** Memory usage is dominated by the storage of the PriorModel parameters, the latent variable $z$, and the intermediate tensors during backpropagation [S1, unofficial].
*   **Measured Performance:** Specific quantitative metrics (PSNR, SSIM) are not provided in the sources for general scenes [S1, unofficial]. The method is noted to produce higher-quality results than previous gradient-domain methods (L1, L2, GradNet) in the context of volumetric photon mapping [S2].
*   **Strengths:** Unsupervised nature (no ground truth needed); effective regularization for sparse-view tomography; leverages the natural image bias of CNNs [S1, unofficial].
*   **Limitations:** Computationally expensive due to iterative optimization [S1, unofficial]. The method cannot reconstruct images with strong noise and tends to over-blur some features [S2].
*   **When to use:** Sparse-view or limited-angle tomography where traditional methods produce severe artifacts [S1, unofficial].
*   **When not to use:** High-noise scenarios (tends to over-blur) [S2].

## Background
Deep Image Prior Reconstruction was introduced as an application of the Deep Image Prior concept to tomographic reconstruction [S1, unofficial]. The DIP concept demonstrates that the architecture of a convolutional neural network itself acts as a powerful prior for natural images [S1, unofficial]. In the context of tomography, the method was adapted to handle the inverse problem of reconstructing 3D volumes from 2D projections [S1, unofficial]. The specific implementation detailed in the sources is associated with the DiffTomo project [S1, unofficial]. The variant for gradient-domain volumetric photon mapping was proposed by Xu et al., building upon earlier work on GradNet [S2].

## Sources
[S1] ApaarRaina/DiffTomo, https://github.com/ApaarRaina/DiffTomo, fetched 2026-10-27 (unofficial)
[S2] A survey on rendering homogeneous participating media, https://doi.org/10.1007/s41095-021-0249-1, fetched 2026-10-26
