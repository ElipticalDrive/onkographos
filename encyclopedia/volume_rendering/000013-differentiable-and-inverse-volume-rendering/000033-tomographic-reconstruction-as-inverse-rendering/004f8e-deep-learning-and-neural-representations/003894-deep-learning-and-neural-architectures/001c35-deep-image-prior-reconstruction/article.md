# Deep Image Prior Reconstruction

Also called: DIP Reconstruction, DRP Denoising

## Overview
Deep Image Prior Reconstruction (DIP Reconstruction) is an iterative tomographic reconstruction method that utilizes a neural network as an implicit regularizer to solve the inverse problem of reconstructing a volume from sparse or noisy sinogram data [S1, unofficial]. It addresses the ill-posed nature of limited-angle or sparse-view tomography by leveraging the structural bias of a convolutional neural network architecture, which naturally favors natural image statistics over noise [S1, unofficial]. Unlike supervised deep learning methods that require large datasets of paired projections and ground-truth volumes, DIP Reconstruction is unsupervised, relying solely on the measured sinogram data to guide the optimization [S1, unofficial]. It distinguishes itself from other neural architectures in this family by explicitly decoupling the optimization of the network weights from the optimization of an intermediate latent variable, allowing the network to act as a dynamic prior that adapts to the specific data consistency constraints [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A sparse or noisy sinogram $p$ (measured projection data).
*   **Output:** A reconstructed 3D volume estimate $x$.
*   **Implicit Inputs:** A prior model architecture (PriorModel), a projector (forward/backward operators), and optimization hyperparameters.

**Data Structures:**
The method maintains the following state variables:
1.  **PriorModel:** A neural network (typically a U-Net or similar encoder-decoder structure) that maps a latent tensor to the volume space. Its parameters $\theta$ are optimized.
2.  **Latent Variable $z$:** An intermediate tensor of the same spatial dimensions as the volume output by PriorModel. This variable is updated via gradient ascent/descent on the data consistency term.
3.  **Projector:** An object implementing the forward projection operator $\mathcal{P}$ and its adjoint (back-projection) $\mathcal{P}^T$.
4.  **Optimizer:** An Adam optimizer instance managing the gradients of the PriorModel parameters $\theta$.

**Algorithm Steps:**
The reconstruction proceeds in an alternating optimization loop consisting of outer epochs and inner steps [S1, unofficial].

1.  **Initialization:**
    *   Initialize the PriorModel with random weights or a standard initialization scheme [S1, unofficial].
    *   Initialize the Adam optimizer with the PriorModel's parameters and a learning rate $lr$ [S1, unofficial].
    *   Compute an initial estimate of the volume by applying Filtered Back Projection (FBP) to the sinogram $p$. Detach this result from the computation graph to prevent gradients from flowing through it during the latent update:
        $$ z_0 = \text{FBP}(p).\text{detach}() $$
    *   Set hyperparameters: number of outer epochs $E$, number of inner steps $K$, and latent update step size $\beta$ [S1, unofficial].

2.  **Outer Loop (Epochs):**
    For each epoch $e$ from $0$ to $E-1$:
    *   **Data Consistency Update (Latent Step):**
        *   Compute the current volume estimate: $x = \text{PriorModel}(z)$.
        *   Compute the forward projection of the estimate: $\hat{p} = \mathcal{P}(x)$.
        *   Compute the residual in projection space: $r = p - \hat{p}$.
        *   Compute the back-projected residual (gradient of the data fidelity term with respect to the volume): $\delta = \mathcal{P}^T(r)$.
        *   Update the latent variable $z$ using gradient ascent on the data consistency term:
            $$ z \leftarrow z + \beta \cdot \delta $$
            *Note: The latent variable $z$ is not detached here; it accumulates changes based on the current network's interpretation of the data [S1, unofficial].*

    *   **Inner Loop (Network Weight Optimization):**
        For each inner step $k$ from $0$ to $K-1$:
        *   Compute the forward projection of the current volume estimate: $\hat{p} = \mathcal{P}(\text{PriorModel}(z))$.
        *   Compute the Mean Squared Error (MSE) loss between the predicted projections and the measured sinogram:
            $$ \mathcal{L} = \frac{1}{N} \sum_{i=1}^{N} (\hat{p}_i - p_i)^2 $$
        *   Perform backpropagation to compute gradients $\frac{\partial \mathcal{L}}{\partial \theta}$ with respect to the PriorModel parameters.
        *   Perform one Adam optimizer step to update $\theta$.

3.  **Final Output:**
    After completing all epochs, the final reconstruction is obtained by passing the final latent variable $z$ through the trained prior network:
    $$ x_{final} = \text{PriorModel}(z) $$

**Parameters:**
*   $E$ (epochs): Number of outer optimization epochs. Default: 250 [S1, unofficial].
*   $K$ (inner_steps): Number of Adam optimization steps per outer epoch. Default: 50 [S1, unofficial].
*   $\beta$ (beta/step_size): Step size for the latent variable update. Default: 0.25 [S1, unofficial].
*   $lr$: Learning rate for the Adam optimizer. Not specified in sources [G].

**Edge Cases and Pitfalls:**
*   **Precision:** The FBP initialization must be detached to avoid gradient conflicts between the latent update and the network weight update [S1, unofficial].
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
    # Initialize PriorModel if not provided
    if prior_model is None:
        prior_model = initialize_prior_model() # [G] Placeholder for specific architecture init
    
    prior_model.train()
    optimizer = optim.Adam(prior_model.parameters(), lr=lr)
    
    # Initialize latent variable z via FBP and detach
    # Note: FBP implementation is assumed to be available or part of the projector package
    z = fbp(sinogram_p).detach()
    
    loss_fn = nn.MSELoss()
    
    for epoch in range(epochs):
        # Outer Loop: Update Latent Variable z
        # 1. Compute current volume estimate
        x = prior_model(z)
        
        # 2. Compute forward projection
        p_hat = projector.forward(x)
        
        # 3. Compute residual and back-project
        residual = sinogram_p - p_hat
        delta = projector.backward(residual)
        
        # 4. Update latent variable
        z = z + beta * delta
        
        # Inner Loop: Optimize PriorModel weights
        for step in range(inner_steps):
            # 1. Forward pass
            x = prior_model(z)
            p_hat = projector.forward(x)
            
            # 2. Compute loss
            loss = loss_fn(p_hat, sinogram_p)
            
            # 3. Backpropagation and optimization step
            optimizer.zero_grad()
            loss.backward()
            optimizer.step()
            
    # Final reconstruction
    output = prior_model(z)
    return output

def fbp(sinogram):
    """
    Placeholder for Filtered Back Projection implementation.
    In a real implementation, this would use a library or custom CUDA kernel.
    """
    # [G] Standard FBP algorithm: Filter sinograms, then back-project
    # Returns a 3D volume tensor
    raise NotImplementedError("FBP implementation required")

def initialize_prior_model():
    """
    Placeholder for initializing the PriorModel architecture.
    Typically a U-Net or similar convolutional network.
    """
    # [G] Return a specific neural network instance
    raise NotImplementedError("PriorModel initialization required")
```

## Subtopics

### Unsupervised Gradient-Domain Neural Reconstruction
This variant, proposed by Xu et al., applies the deep image prior concept to gradient-domain volumetric photon density estimation, specifically for volumetric photon mapping [S2]. It utilizes a variant of GradNet with an encoded shift connection and a separated auxiliary feature branch that includes volume-based auxiliary features such as transmittance and photon density [S2]. The network is designed to smooth images at a global scale while preserving high-frequency details on a small scale [S2]. It produces higher-quality results than previous works such as L1, L2, or standard GradNet [S2]. However, it cannot reconstruct images with strong noise and tends to over-blur some features [S2]. While originally developed for volumetric photon mapping, the method is noted as straightforward to extend to other tasks like beam radiance estimation [S2].

## Operating characteristics
*   **Time Complexity:** The method is computationally intensive due to the alternating optimization loop. Each epoch involves one forward/backward pass for the latent update and $K$ forward/backward passes for the network weight update. The total number of forward projections per epoch is $K+1$. With typical defaults ($E=250, K=50$), this results in 12,750 forward projections per reconstruction [S1, unofficial].
*   **Memory Complexity:** Memory usage is dominated by the storage of the PriorModel parameters, the latent variable $z$, and the intermediate tensors during backpropagation. It does not require storing large training datasets, making it memory-efficient compared to supervised methods [S1, unofficial].
*   **Measured Performance:** Specific quantitative metrics (PSNR, SSIM) are not provided in the sources for general scenes [S1, unofficial]. The method is noted to produce higher-quality results than previous gradient-domain methods (L1, L2, GradNet) in the context of volumetric photon mapping [S2].
*   **Strengths:** Unsupervised nature (no ground truth needed); effective regularization for sparse-view tomography; leverages the natural image bias of CNNs [S1, unofficial].
*   **Limitations:** Computationally expensive due to iterative optimization; sensitive to hyperparameters ($\beta$, $lr$, epochs); may over-smooth features or fail with strong noise [S2]; requires a good initialization (FBP) [S1, unofficial].
*   **When to use:** Sparse-view or limited-angle tomography where traditional methods (FBP, SIRT) produce severe artifacts, and supervised deep learning is not feasible due to lack of training data [S1, unofficial].
*   **When not to use:** High-noise scenarios (tends to over-blur) [S2]; real-time applications requiring fast reconstruction [S1, unofficial].

## Background
Deep Image Prior Reconstruction was introduced as an application of the Deep Image Prior (DIP) concept to tomographic reconstruction. The DIP concept, originally proposed by Ulyanov et al. for image denoising and super-resolution, demonstrates that the architecture of a convolutional neural network itself acts as a powerful prior for natural images [S1, unofficial]. In the context of tomography, the method was adapted to handle the inverse problem of reconstructing 3D volumes from 2D projections. The specific implementation detailed in the sources is associated with the DiffTomo project [S1, unofficial]. The variant for gradient-domain volumetric photon mapping was proposed by Xu et al., building upon earlier work on GradNet [S2].

## Sources
[S1] ApaarRaina/DiffTomo, https://github.com/ApaarRaina/DiffTomo, fetched 2026-10-27 (unofficial)
[S2] A survey on rendering homogeneous participating media, https://doi.org/10.1007/s41095-021-0249-1, fetched 2026-10-26
