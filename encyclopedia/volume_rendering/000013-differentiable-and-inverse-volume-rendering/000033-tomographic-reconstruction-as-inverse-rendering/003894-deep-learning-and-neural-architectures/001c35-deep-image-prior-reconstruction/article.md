# Deep Image Prior Reconstruction

Also called: DIP Reconstruction, DRP Denoising

## Overview
Deep Image Prior Reconstruction is an iterative inverse rendering method that reconstructs a volume or image from sparse or noisy projection data by using a neural network as an implicit regularizer [S1, unofficial]. It distinguishes itself from traditional iterative algebraic methods by jointly optimizing a latent variable for data consistency and the network weights to fit the sinogram [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** Sparse sinogram $p$ (measured projections) [S1, unofficial].
- **Input:** Differentiable projector supporting forward and backward projection operators [S1, unofficial].
- **Input:** PriorModel (neural network architecture) [S1, unofficial].
- **Output:** Reconstructed image/volume $x$ [S1, unofficial].

**Data Structures:**
The method stores: projector (object), epochs (int), inner_steps (int), beta (float), lr (float), loss (nn.MSELoss) [S1, unofficial]. It maintains a latent variable tensor $z$ and the PriorModel parameters $\theta$ [S1, unofficial].

**Parameters:**
- `epochs`: Number of outer optimization epochs. Default: 250 [S1, unofficial].
- `inner_steps`: Number of Adam optimization steps on PriorModel weights per epoch. Default: 50 [S1, unofficial].
- `beta`: Step size for the latent variable update. Default: 0.25 [S1, unofficial].
- `lr`: Learning rate for the Adam optimizer. Not given in the sources.

**Algorithm Steps:**
1. Initialize the PriorModel if not provided, moving it to the device [S1, unofficial].
2. Initialize the Adam optimizer with the PriorModel's parameters and learning rate `lr` [S1, unofficial].
3. Initialize the latent variable $z$ by applying Filtered Back Projection (FBP) to the sparse sinogram $p$ and detaching it from the computation graph: $z = \text{FBP}(p).\text{detach()}$ [S1, unofficial].
4. For each epoch from 0 to `epochs`-1:
   a. Compute the current image estimate $x = \text{prior}(z)$ [S1, unofficial].
   b. Compute the data consistency residual $\delta = \text{projector.backward}(p - \text{projector.forward}(x))$ [S1, unofficial].
   c. Update the latent variable: $z = x + \beta \cdot \delta$ [S1, unofficial].
5. For each inner step from 0 to `inner_steps`-1:
   a. Compute the forward projection of the current image estimate: $\hat{p} = \text{projector.forward}(\text{prior}(z))$ [S1, unofficial].
   b. Compute the MSE loss between $\hat{p}$ and the measured sinogram $p$: $\mathcal{L} = \text{MSE}(\hat{p}, p)$ [S1, unofficial].
   c. Perform backpropagation and one Adam optimizer step on the PriorModel parameters [S1, unofficial].
6. After all epochs, return the final reconstruction by passing the final $z$ through the prior network: $\text{output} = \text{prior}(z)$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.optim as optim

def deep_image_prior_reconstruction(
    sinogram: torch.Tensor,
    projector: object,
    prior_model: nn.Module,
    epochs: int = 250,
    inner_steps: int = 50,
    beta: float = 0.25,
    lr: float = 0.01,
    device: str = 'cpu'
) -> torch.Tensor:
    # Move prior model to device
    prior_model = prior_model.to(device)
    
    # Initialize Adam optimizer on prior model parameters
    optimizer = optim.Adam(prior_model.parameters(), lr=lr)
    
    # Initialize latent variable z using FBP
    # [G] FBP implementation not provided in sources; assumed available.
    z_init = torch.zeros_like(sinogram)
    z = z_init.detach().to(device)
    
    # Loss function
    mse_loss = nn.MSELoss()
    
    for epoch in range(epochs):
        # --- Data Consistency Step ---
        # 1. Compute current image estimate from latent z
        x = prior_model(z)
        
        # 2. Forward project the estimate
        p_hat = projector.forward(x)
        
        # 3. Compute residual in sinogram space and back-project
        delta = projector.backward(sinogram - p_hat)
        
        # 4. Update latent variable
        z = x + beta * delta
        
        # --- Network Optimization Steps ---
        for _ in range(inner_steps):
            # 1. Forward pass through prior model
            x = prior_model(z)
            
            # 2. Forward projection
            p_hat = projector.forward(x)
            
            # 3. Compute loss
            loss = mse_loss(p_hat, sinogram)
            
            # 4. Backpropagation and optimizer step
            optimizer.zero_grad()
            loss.backward()
            optimizer.step()
            
    # Final reconstruction
    output = prior_model(z)
    return output
```

**Edge Cases and Pitfalls:**
Not given in the sources.

## Subtopics

### Unsupervised Gradient-Domain Neural Reconstruction
An unsupervised neural network variant for gradient-domain volumetric photon density estimation, specifically for volumetric photon mapping [S2]. It uses a GradNet variant with encoded shift connections and auxiliary features (transmittance, photon density) to smooth images globally while preserving high-frequency details locally [S2]. It produces higher-quality results than L1, L2, or GradNet baselines [S2]. Cannot reconstruct images with strong noise and tends to over-blur some features [S2].

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Measured Performance:** Not given in the sources.
- **Strengths and Limitations:** Not given in the sources.
- **When to Use/Not:** Not given in the sources.

## Background
Deep Image Prior Reconstruction is an iterative reconstruction method that uses a neural network (PriorModel) as an implicit regularizer [S1, unofficial]. It alternates between updating an intermediate latent variable using a data consistency step and optimizing the network weights to fit sparse sinogram data [S1, unofficial]. The method is initialized with Filtered Back Projection (FBP) and optimized via the Adam optimizer [S1, unofficial].

## Sources
[S1] ApaarRaina/DiffTomo, https://github.com/ApaarRaina/DiffTomo, fetched 2026-10-27 (unofficial)
[S2] A survey on rendering homogeneous participating media, https://doi.org/10.1007/s41095-021-0249-1, fetched 2026-10-26
