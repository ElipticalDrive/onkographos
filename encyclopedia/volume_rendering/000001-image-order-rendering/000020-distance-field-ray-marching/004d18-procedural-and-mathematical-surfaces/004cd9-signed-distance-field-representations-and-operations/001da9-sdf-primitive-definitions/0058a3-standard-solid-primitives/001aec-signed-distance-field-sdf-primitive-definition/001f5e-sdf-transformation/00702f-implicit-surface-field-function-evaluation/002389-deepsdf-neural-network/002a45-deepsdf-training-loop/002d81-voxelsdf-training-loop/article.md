# VoxelSDF Training Loop

Also called: Training Procedure (L1 Loss + Adam + StepLR), VoxelSDF Training Procedure

## Overview
The VoxelSDF Training Loop is the optimization procedure for the VoxelSDF architecture, which predicts Signed Distance Function (SDF) values by interpolating features from a voxelized latent embedding grid [S1, unofficial]. It solves the problem of learning compact, spatially-aware implicit representations by conditioning a neural network on local voxel features rather than global latent codes . It sets itself apart from global implicit representations like DeepSDF by using a structured voxel grid to store latent codes, allowing for localized feature interpolation .

## How it works
**Inputs and Outputs:**
*   **Input:** A dataset of 3D points and their corresponding ground-truth SDF values, indexed by shape identifier (e.g., ShapeNetSDF).
*   **Output:** Updated parameters for the VoxelSDF decoder (MLP) and the GridEmbedding (lookup table of latent codes).

**Data Structures:**
*   **GridEmbedding:** A learnable tensor storing latent codes for each voxel in the grid. Accessed via `embeddings(index)` where `index` maps shapes to specific voxel coordinates or embeddings.
*   **Model:** The VoxelSDF decoder, typically an MLP that takes latent codes and 3D points as input.
*   **Optimizer State:** Maintained by Adam optimizer for both model parameters and embedding parameters.

**Numbered Steps:**
1.  **Initialization:** Initialize the `GridEmbedding` tensor for the dataset size. Initialize the Adam optimizer with separate learning rates for the model and embeddings. Initialize a StepLR scheduler.
2.  **Epoch Loop:** Iterate through epochs from `start_epoch` to `epochs`.
3.  **Batch Iteration:** For each batch in the dataloader:
    a. Move `points`, `sdfs`, and `index` to the target device (e.g., GPU).
    b. Retrieve latent codes: `latent_codes = embeddings(index)`.
    c. Forward pass: `sdf_pred = model(latent_codes, points)`.
    d. Reshape `sdf_pred` and `sdfs` to `(real_batch_size, -1)`.
    e. Compute L1 loss: `loss = criterion(sdf_pred, sdfs)`.
    f. Zero gradients: `optimizer.zero_grad()`.
    g. Backward pass: `loss.backward()`.
    h. Update parameters: `optimizer.step()`.
4.  **Scheduler Step:** After each epoch, step the learning rate scheduler.
5.  **Checkpointing:** Every 100 epochs, save the model state dict and embedding state dict.
6.  **Logging:** Record epoch loss.

**Formulas:**
*   **L1 Loss:** $L = \frac{1}{N} \sum_{i=1}^{N} | \hat{y}_i - y_i |$, where $\hat{y}_i$ is the predicted SDF and $y_i$ is the ground truth SDF.
*   **Adam Optimizer:** Uses adaptive moment estimation with separate learning rates for model and embeddings.
*   **StepLR Scheduler:** Reduces learning rate by a factor (gamma) every `step_size` epochs.

**Parameters:**
*   **Learning Rate (Model):** $1 \times 10^{-4}$ [S1, unofficial].
*   **Learning Rate (Embeddings):** $1 \times 10^{-2}$ [S1, unofficial].
*   **Step Size (Scheduler):** 100 epochs (default) [S1, unofficial].
*   **Batch Size:** Not given in the sources.
*   **Total Epochs:** Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Device Placement:** Points, SDFs, and indices must be moved to the correct device (CPU/GPU) before computation [S1, unofficial].
*   **Reshaping:** Predictions and targets must be reshaped to match the batch dimension before loss computation [S1, unofficial].
*   **Gradient Zeroing:** Gradients must be zeroed before each backward pass to prevent accumulation [S1, unofficial].
*   **Checkpointing:** Models are saved every 100 epochs; intermediate epochs are not saved by default [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import numpy as np
import os

class VoxelSDFTrainer:
    def __init__(self, model, embeddings, dataloader, epochs=1000, start_epoch=0, result_dir="./results"):
        self.model = model
        self.embeddings = embeddings
        self.dataloader = dataloader
        self.epochs = epochs
        self.start_epoch = start_epoch
        self.result_dir = result_dir
        self.device = torch.device("cuda" if torch.cuda.is_available() else "cpu")
        
        # Initialize optimizer with separate learning rates
        self.optimizer = torch.optim.Adam([
            {"params": self.model.parameters(), "lr": 1e-4},
            {"params": self.embeddings.parameters(), "lr": 1e-2}
        ])
        
        # Initialize scheduler
        self.scheduler = torch.optim.lr_scheduler.StepLR(self.optimizer, step_size=100, gamma=0.1)
        
        # Initialize criterion
        self.criterion = nn.L1Loss()
        
        self.losses = []

    def train_epoch(self, epoch):
        self.model.train()
        epoch_loss = 0.0
        
        for points, sdfs, index in self.dataloader:
            # Move data to device
            points = points.to(self.device)
            sdfs = sdfs.to(self.device)
            index = index.to(self.device)
            
            real_batch_size = index.shape[0]
            
            # Forward pass
            latent_codes = self.embeddings(index)
            sdf_pred = self.model(latent_codes, points)
            
            # Reshape for loss computation
            sdf_pred = sdf_pred.view(real_batch_size, -1)
            sdfs = sdfs.view(real_batch_size, -1)
            
            # Compute loss
            loss = self.criterion(sdf_pred, sdfs)
            
            # Backward pass and optimization
            self.optimizer.zero_grad()
            loss.backward()
            self.optimizer.step()
            
            epoch_loss += loss.item()
            
        return epoch_loss / len(self.dataloader)

    def train(self):
        for e in range(self.start_epoch, self.epochs + 1):
            epoch_loss = self.train_epoch(e)
            self.losses.append(epoch_loss)
            
            # Step scheduler
            self.scheduler.step()
            
            # Save model every 100 epochs
            if e % 100 == 0:
                self.save_model(e)
                np.save(os.path.join(self.result_dir, "losses.npy"), np.array(self.losses))
                
            yield e, self.losses

    def save_model(self, epoch):
        model_path = os.path.join(self.result_dir, f"voxel_sdf_epoch_{epoch}.param")
        embedding_path = os.path.join(self.result_dir, f"embeddings_epoch_{epoch}.param")
        torch.save(self.model.state_dict(), model_path)
        torch.save(self.embeddings.state_dict(), embedding_path)

    def load_model(self, epoch):
        model_path = os.path.join(self.result_dir, f"voxel_sdf_epoch_{epoch}.param")
        embedding_path = os.path.join(self.result_dir, f"embeddings_epoch_{epoch}.param")
        self.model.load_state_dict(torch.load(model_path, weights_only=True))
        self.embeddings.load_state_dict(torch.load(embedding_path, weights_only=True))
```

## Subtopics

### [MetaSDF](19189-metasdf.md)
MetaSDF is a method for learning a prior over signed distance function (SDF) shape spaces by treating shape reconstruction as a meta-learning problem, where a neural network is specialized to a new shape via few steps of gradient descent on context observations. It sets itself apart from auto-decoder frameworks by performing specialization in the forward pass via a fixed number of gradient steps, making it an order of magnitude faster at inference, and from pooling-based set-encoders by avoiding the underfitting of context sets.

### [SDF Training Step](19737-sdf-training-step.md)
The SDF Training Step is the core optimization routine for the VoxelSDF architecture, which predicts Signed Distance Function (SDF) values for 3D points by interpolating features from a voxelized latent embedding grid. It solves the problem of learning compact, spatially-aware implicit representations by conditioning a neural network on local voxel features rather than global latent codes. It sets itself apart from global implicit representations like DeepSDF by using a structured voxel grid to store latent codes, allowing for localized feature interpolation.

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of training points and epochs. Each epoch requires a forward and backward pass through the model and embeddings.
*   **Memory Complexity:** Proportional to the size of the GridEmbedding tensor and the model parameters. The batch size also affects memory usage during training.
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** The use of a voxel grid allows for localized feature interpolation, which can improve scalability compared to global latent codes for large datasets.
*   **Strengths:** Combines the benefits of voxel-based representations (spatial locality) with implicit representations (continuous SDF values).
*   **Limitations:** Requires careful tuning of learning rates for model and embeddings. The voxel grid size can impact memory usage.
*   **When to Use:** Suitable for learning compact, spatially-aware implicit representations for 3D shape reconstruction.
*   **When Not to Use:** Not suitable for applications requiring real-time inference without pre-trained models, or when memory constraints are severe.

## Background
The VoxelSDF Training Loop is based on the VoxelSDF architecture, which integrates voxel-based latent embeddings with a neural network decoder for SDF prediction [S1, unofficial]. It builds on the DeepSDF framework by introducing a voxelized latent code structure . The method was introduced in the context of 3D shape reconstruction and representation learning.

## Sources
[S1] TurtleJayWang/VC-SDFGen, https://github.com/TurtleJayWang/VC-SDFGen, fetched 2026-10-28 (unofficial)
