# DeepSDF Training Loop

Also called: DeepSDF Optimization Loop, DeepSDF Training Procedure

## Overview
The DeepSDF Training Loop is the optimization procedure for the DeepSDF model, which jointly learns a continuous signed distance field (SDF) representation for 3D shapes and a per-shape latent code [S1, unofficial]. It solves the problem of reconstructing arbitrary 3D geometry from sparse or unstructured point cloud data by mapping (latent code, 3D coordinate) pairs to signed distance values using a multi-layer perceptron (MLP) [S1, unofficial]. It differs from voxel-based methods like Vox2SDF by optimizing a continuous implicit function rather than a discrete grid, allowing for higher resolution reconstruction without memory constraints proportional to grid size [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A dataset of 3D shapes, where each shape is represented by a set of 3D points and their corresponding ground-truth signed distance values. The dataset is batched into tuples `(points, sdfs, index)`, where `points` is a tensor of shape $(N, 3)$, `sdfs` is a tensor of shape $(N, 1)$, and `index` identifies the shape within the dataset [S1, unofficial].
*   **Outputs:** Updated MLP weights, updated embedding table weights, and a history of training losses [S1, unofficial].

**Data Structures:**
*   **Model:** An MLP that takes a latent code vector and a 3D point as input and outputs a scalar SDF value [S1, unofficial].
*   **Embeddings:** A learnable embedding table `self.embeddings` where each row corresponds to a shape's latent code vector [S1, unofficial].
*   **Optimizer:** An optimizer (e.g., Adam) that updates both the MLP parameters and the embedding parameters [S1, unofficial].
*   **Scheduler:** A learning rate scheduler (e.g., StepLR) that reduces the learning rate periodically [S1, unofficial].
*   **Checkpoints:** Files named `voxel_sdf_epoch_{epoch}.param` for MLP weights and `embeddings_epoch_{epoch}.param` for embedding weights [S1, unofficial]. Losses are stored in `losses.npy` [S1, unofficial].

**Parameters:**
*   `batch_size`: Number of points sampled per batch [S1, unofficial].
*   `epochs`: Total number of training epochs [S1, unofficial].
*   `start_epoch`: Epoch to start training from (for resuming) [S1, unofficial].
*   `scheduler`: `StepLR` with `step_size=500` and `gamma=0.5` [S1, unofficial].
*   `criterion`: `nn.L1Loss` with `reduction="sum"` [S1, unofficial].

**Algorithm:**
1.  Initialize the dataloader with the dataset, `batch_size`, and `shuffle=True` [S1, unofficial].
2.  Initialize `losses` list to track epoch losses [S1, unofficial].
3.  For each epoch `e` from `start_epoch` to `epochs`:
    a.  Reset `epoch_loss` to 0 [S1, unofficial].
    b.  For each batch `(points, sdfs, index)` from the dataloader:
        i.  Move `points` and `sdfs` to the device (e.g., GPU) [S1, unofficial].
        ii. Determine `real_batch_size` as `index.shape[0]` [S1, unofficial].
        iii. Retrieve `latent_codes` by indexing the embedding table: `latent_codes = self.embeddings(index.to(self.device))` [S1, unofficial].
        iv. Compute predicted SDFs: `sdf_pred = self.model(latent_codes, points)` [S1, unofficial].
        v.  Reshape `sdf_pred` and `sdfs` to `(real_batch_size, -1)` to flatten the spatial dimensions [S1, unofficial].
        vi. Compute loss: `loss = self.criterion(sdf_pred, sdfs)` [S1, unofficial].
        vii. Zero gradients: `self.optimizer.zero_grad()` [S1, unofficial].
        viii. Backpropagate: `loss.backward()` [S1, unofficial].
        ix.  Update parameters: `self.optimizer.step()` [S1, unofficial].
        x.   Accumulate loss: `epoch_loss += loss.item()` [S1, unofficial].
    c.  Step the scheduler: `self.scheduler.step()` [S1, unofficial].
    d.  If `e % 100 == 0`:
        i.   Save model state: `self.save_model(e)` [S1, unofficial].
        ii.  Append `epoch_loss` to `losses` and save `losses.npy` [S1, unofficial].
    e.  Yield `e` and `losses` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def train_loop(model, embeddings, optimizer, scheduler, criterion, dataloader, start_epoch, epochs, result_dir):
    losses = []
    for e in range(start_epoch, epochs + 1):
        epoch_loss = 0
        for points, sdfs, index in dataloader:
            points = points.to(model.device)
            sdfs = sdfs.to(model.device)
            
            real_batch_size = index.shape[0]
            
            # Retrieve latent codes for the shapes in this batch
            latent_codes = embeddings(index.to(model.device))
            
            # Forward pass
            sdf_pred = model(latent_codes, points)
            
            # Flatten predictions and targets for loss computation
            sdf_pred = sdf_pred.view(real_batch_size, -1)
            sdfs = sdfs.view(real_batch_size, -1)
            
            # Compute L1 loss
            loss = criterion(sdf_pred, sdfs)
            
            # Backward pass
            optimizer.zero_grad()
            loss.backward()
            optimizer.step()
            
            epoch_loss += loss.item()
        
        # Update learning rate
        scheduler.step()
        
        # Save checkpoint every 100 epochs
        if e % 100 == 0:
            save_model(model, embeddings, e, result_dir)
            losses.append(epoch_loss)
            np.save(os.path.join(result_dir, "losses.npy"), np.array(losses))
        
        yield e, losses

def save_model(model, embeddings, epoch, result_dir):
    model_path = os.path.join(result_dir, f"voxel_sdf_epoch_{epoch}.param")
    embedding_path = os.path.join(result_dir, f"embeddings_epoch_{epoch}.param")
    torch.save(model.state_dict(), model_path)
    torch.save(embeddings.state_dict(), embedding_path)

def latest_epoch(result_dir):
    losses_path = os.path.join(result_dir, "losses.npy")
    if os.path.exists(losses_path):
        losses = np.load(losses_path).tolist()
        latest_epoch = len(losses) // 100 * 100
        # Verify files exist, decrement if necessary
        model_path = os.path.join(result_dir, f"voxel_sdf_epoch_{latest_epoch}.param")
        embedding_path = os.path.join(result_dir, f"embeddings_epoch_{latest_epoch}.param")
        while not (os.path.exists(model_path) and os.path.exists(embedding_path)) and latest_epoch > 0:
            latest_epoch -= 100
            model_path = os.path.join(result_dir, f"voxel_sdf_epoch_{latest_epoch}.param")
            embedding_path = os.path.join(result_dir, f"embeddings_epoch_{latest_epoch}.param")
        return latest_epoch
    return 0
```

## Subtopics

### VoxelSDF Training Loop
Trains the VoxelSDF decoder and grid embeddings using L1 loss and an Adam optimizer with separate learning rates for the model and embeddings (1e-4 for model, 1e-2 for embeddings) [S1, unofficial]. It uses a step scheduler with `step_size=100` epochs [S1, unofficial]. The training loop iterates over the ShapeNetSDF dataset, performs a forward pass through the grid embeddings and model, computes L1 loss, and updates parameters [S1, unofficial]. It saves the model every 100 epochs [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of epochs and the number of batches per epoch [S1, unofficial].
*   **Memory Complexity:** Proportional to the batch size and the number of shapes in the dataset (for the embedding table) [S1, unofficial].
*   **Strengths:** Allows for high-resolution reconstruction by avoiding voxel grid memory constraints; learns a continuous representation [S1, unofficial].
*   **Limitations:** Requires careful tuning of learning rates for embeddings vs. model parameters; resuming training requires verifying checkpoint existence [S1, unofficial].
*   **When to use:** When reconstructing complex 3D shapes from point clouds with high fidelity is required [S1, unofficial].
*   **When not to use:** When real-time inference on low-memory devices is critical, or when discrete voxel representations are sufficient [S1, unofficial].

## Background
The DeepSDF method was introduced by Park et al. in 2019 [S1, unofficial]. It represents a shift from discrete voxel-based representations to continuous implicit functions for 3D shape reconstruction [S1, unofficial]. The training loop described here is based on the implementation in VC-SDFGen [S1, unofficial].

## Sources
[S1] TurtleJayWang/VC-SDFGen, https://github.com/TurtleJayWang/VC-SDFGen, fetched 2026-10-28 (unofficial)
