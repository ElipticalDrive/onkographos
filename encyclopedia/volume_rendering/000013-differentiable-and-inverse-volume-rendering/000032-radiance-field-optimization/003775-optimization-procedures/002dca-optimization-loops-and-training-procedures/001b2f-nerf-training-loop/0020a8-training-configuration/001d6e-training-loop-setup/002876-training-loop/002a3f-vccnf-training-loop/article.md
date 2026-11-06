# VCCNF Training Loop

Also called: VCCNF Training Procedure

## Overview
The VCCNF Training Loop is a supervised optimization procedure for a Voxelized Continuous Coordinate Normalizing Flow (VCCNF) [S1, unofficial]. It learns to map Gaussian noise to shape latent codes conditioned on voxel data by minimizing the Mean Squared Error (MSE) between predicted and target latent codes [S1, unofficial]. It differs from its sibling [MVDream Multi-View Diffusion Guidance](6345-mvdream-multi-view-diffusion-guidance.md), which uses multi-view diffusion guidance, by relying on direct voxel-to-latent regression without diffusion-based sampling [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** A dataset of voxel grids (SDFs) and corresponding shape indices. Pre-trained embeddings are loaded from `embedding_dir/embeddings_epoch_{embedding_epoch}.param` [S1, unofficial].
- **Output:** A trained VCCNF model checkpoint (`vccnf_epoch_{e}.param`) and a loss history array (`losses_vccnf.npy`) [S1, unofficial].

**Data Structures:**
- **Voxel Data:** Loaded from `.npz` files as `torch.from_numpy(data["voxel_data"]).float()` [S1, unofficial].
- **Latent Codes:** Retrieved via `embeddings(indices)` [S1, unofficial].
- **Gaussian Latents:** `torch.randn_like(latent_codes)` [S1, unofficial].

**Parameters:**
- **Loss Function:** `nn.MSELoss` [S1, unofficial].
- **Scheduler:** `StepLR` with `step_size=100` and `gamma=0.5`, applied after each epoch [S1, unofficial].
- **Dataset Split:** 0.8 train / 0.2 validation, seeded with `manual_seed(42)` [S1, unofficial].

**Steps:**
1. Initialize the dataset split and load pre-trained embeddings.
2. For each epoch $e$ from `start_epoch` to `epochs`:
   a. Set model to training mode (`model.train()`).
   b. Iterate over batches `(voxel_data, indices)`.
   c. Move `indices` to the target device.
   d. Retrieve target latent codes: $L_{target} = \text{embeddings}(indices)$.
   e. Generate Gaussian noise: $L_{noise} = \mathcal{N}(0, I)$ with the same shape as $L_{target}$.
   f. Compute predicted latent codes: $L_{pred} = \text{VCCNF}(voxel\_data, L_{noise})$.
   g. Compute loss: $\mathcal{L} = \text{MSE}(L_{pred}, L_{target})$.
   h. Zero gradients, perform backward pass, and step the optimizer.
   i. Accumulate loss and step the scheduler.
   j. Append epoch loss to the history list.
   k. If $(e+1)$ is divisible by 100, print progress, save `losses_vccnf.npy`, and save `vccnf_epoch_{e+1}.param`.
   l. Yield $e$ and losses.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import numpy as np

def train_vccnf(model, embeddings, dataloader, start_epoch, epochs, embedding_epoch, device):
    # Load pre-trained embeddings
    # Note: Exact loading mechanism for embeddings_epoch_{embedding_epoch}.param is not specified in source [S1, unofficial]
    # Assuming embeddings is a callable or tensor lookup based on indices
    loss_fn = nn.MSELoss()
    optimizer = torch.optim.Adam(model.parameters())
    scheduler = torch.optim.lr_scheduler.StepLR(optimizer, step_size=100, gamma=0.5)
    
    losses = []
    
    for e in range(start_epoch, epochs):
        model.train()
        epoch_loss = 0.0
        
        for voxel_data, indices in dataloader:
            indices = indices.to(device)
            voxel_data = voxel_data.to(device)
            
            # Retrieve target latent codes
            latent_codes = embeddings(indices)
            
            # Generate Gaussian noise
            gaussian_latents = torch.randn_like(latent_codes)
            
            # Forward pass
            latent_pred = model(voxel_data, gaussian_latents)
            
            # Compute loss
            loss = loss_fn(latent_pred, latent_codes)
            
            # Backward pass
            optimizer.zero_grad()
            loss.backward()
            optimizer.step()
            
            epoch_loss += loss.item()
        
        scheduler.step()
        losses.append(epoch_loss)
        
        if (e + 1) % 100 == 0:
            print(f"Epoch {e+1}, Loss: {epoch_loss}")
            np.save("losses_vccnf.npy", np.array(losses))
            torch.save(model.state_dict(), f"vccnf_epoch_{e+1}.param")
            
        yield e, losses
```

**Edge Cases and Pitfalls:**
- **Resuming Training:** The function `latest_epoch()` computes the epoch as `len(losses)//100*100`. There is no existence check for the weight file before loading, which can cause crashes if the file is missing [S1, unofficial]. This differs from `DeepSDFTraining` which performs such checks [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of epochs and batch size. The scheduler steps every epoch.
- **Memory Complexity:** Depends on the voxel grid size and latent dimension. Gaussian noise and latent codes are stored in memory per batch.
- **Performance:** Not given in the sources.
- **Strengths:** Simple MSE-based supervision; leverages pre-trained embeddings.
- **Limitations:** No validation loss tracking mentioned; potential crash on resume if weights are missing.
- **When to use:** When voxel data is available and a direct mapping to latent codes is desired.
- **When not to use:** When diffusion-based guidance is required (use [MVDream Multi-View Diffusion Guidance](6345-mvdream-multi-view-diffusion-guidance.md) instead).

## Background
The VCCNF Training Loop is part of the VC-SDFGen project, which combines Voxelized Continuous Coordinate Normalizing Flows with SDF generation [S1, unofficial]. It builds on [Training Loop](4290-training-loop.md) and is distinct from [MVDream Multi-View Diffusion Guidance](6345-mvdream-multi-view-diffusion-guidance.md) in its use of direct regression rather than diffusion guidance.

## Sources
[S1] TurtleJayWang/VC-SDFGen, https://github.com/TurtleJayWang/VC-SDFGen, fetched 2026-10-28 (unofficial)
