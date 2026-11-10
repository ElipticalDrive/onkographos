# NeRF Training Configuration

Also called: NeRF Training Loop Configuration, NeRF Training Setup, Sparsity Prior, Sparsity Regularization, Sparsity Loss

## Overview
NeRF Training Configuration refers to the specific hyperparameters, optimization routines, and regularization terms used to train a Neural Radiance Field (NeRF) model. It solves the problem of converging to a physically plausible volumetric representation by preventing the network from generating arbitrary, non-photorealistic geometry in unobserved regions. It distinguishes itself from sibling methods like [Mip-NeRF](3381-mip-nerf.md) or [Instant Neural Graphics Primitives (Instant NGP)](8637-instant-neural-graphics-primitives-instant-ngp.md) by explicitly incorporating a **sparsity prior** (or sparsity loss) to condense volume density, ensuring the resulting field is suitable for downstream sparse representations like octrees [S1, unofficial][S2].

## How it works
The training configuration defines the data pipeline, the composite loss function (including the sparsity term), and the optimizer settings required to minimize the difference between rendered and ground-truth images while enforcing geometric sparsity.

### Inputs and Outputs
*   **Inputs**: Training images (RGB), camera poses (extrinsics), camera intrinsics (focal length, image dimensions), and a bounding box defining the scene volume.
*   **Outputs**: A trained NeRF model (typically two MLPs for coarse and fine sampling) and associated optimizer state.

### Data Structures
The training loop manages the following data structures:
1.  **Dataset**: A collection of rays generated from training images. Each ray is defined by origin $\mathbf{o}$, direction $\mathbf{d}$, and target color $\mathbf{C}^*$.
2.  **Model State**:
    *   `coarse_model_state_dict`: Parameters for the coarse sampling MLP.
    *   `fine_model_state_dict`: Parameters for the fine sampling MLP.
    *   `optimizer_state_dict`: State for the Adam optimizer.
3.  **Checkpoint**: A dictionary saved to disk with keys: `'epoch'`, `'coarse_model_state_dict'`, `'fine_model_state_dict'`, `'optimizer_state_dict'` [S3, unofficial].

### Numbered Steps
1.  **Data Loading**: Load the training dataset (e.g., BlenderDataset) at a specified resolution (e.g., shorter side 400 pixels) [S3, unofficial]. Extract camera intrinsics: height $H$, width $W$, and focal length $f$ [S3, unofficial].
2.  **Ray Generation**: For each training image, generate rays. The camera intrinsic matrix is constructed as:
    $$
    K = \begin{bmatrix} f & 0 & 0.5 \cdot W \\ 0 & f & 0.5 \cdot H \\ 0 & 0 & 1 \end{bmatrix}
    $$
    [S3, unofficial].
3.  **Model Initialization**: Initialize two NeRF models: a coarse model and a fine model [S3, unofficial]. Combine their parameters.
4.  **Optimizer Setup**: Create an Adam optimizer with learning rate $\eta = 5 \times 10^{-4}$ and betas $(\beta_1, \beta_2) = (0.9, 0.999)$ [S3, unofficial].
5.  **Loss Definition**: Define the composite loss function $\mathcal{L}$:
    $$
    \mathcal{L} = \mathcal{L}_{\text{RGB}} + \beta_s \mathcal{L}_{\text{sparsity}}
    $$
    where $\mathcal{L}_{\text{RGB}}$ is the Mean Squared Error (MSE) between predicted and target colors, and $\mathcal{L}_{\text{sparsity}}$ is the sparsity regularization term [S1, unofficial][S2].
6.  **Training Loop**:
    *   Sample a batch of rays (batch size = 1) [S3, unofficial].
    *   Perform volume rendering to predict color $\hat{\mathbf{C}}(\mathbf{r})$ using the coarse and fine sampling strategies [S1, unofficial].
    *   Sample $K$ uniformly random points within the bounding box to compute $\mathcal{L}_{\text{sparsity}}$ [S1, unofficial].
    *   Compute gradients and update model parameters.
7.  **Checkpointing**: Save the model state, optimizer state, and epoch number to a `.pt` file [S3, unofficial].

### Formulas
**Volume Rendering**:
The predicted color for a ray $\mathbf{r}$ is:
$$
\hat{C}(\mathbf{r}) = \sum_{i=0}^N T_i (1 - \exp(-\sigma_i \delta_i)) c_i
$$
where $T_i = \exp \left( -\sum_{j=0}^{i-1} \sigma_j \delta_j \right)$, $\sigma_i$ is density, $\delta_i$ is distance between samples, and $c_i$ is color [S1, unofficial].

**RGB Loss**:
$$
\mathcal{L}_{\text{RGB}} = \sum_{\mathbf{r} \in \mathcal{R}} \| \mathbf{C}(\mathbf{r}) - \hat{C}(\mathbf{r}) \|_2^2
$$
where $\mathcal{R}$ is the set of rays in the batch [S1, unofficial].

**Sparsity Loss**:
$$
\mathcal{L}_{\text{sparsity}} = \frac{1}{K} \sum_{k=1}^K | 1 - \exp(-\delta \sigma_k) |
$$
where $\sigma_k$ are density values at $K$ uniformly random points within the bounding box, and $\delta$ is an exponential scaling factor [S1, unofficial][S2].

### Parameters
*   **Learning Rate**: $5 \times 10^{-4}$ [S3, unofficial].
*   **Sparsity Weight ($\beta_s$)**: $0.01$ [S2].
*   **Exponential Scaling Factor ($\delta$)**: $0.1$ [S2].
*   **Batch Size**: 1 [S3, unofficial].
*   **Image Resolution**: 400 (shorter side) [S3, unofficial].
*   **Optimizer Betas**: $(0.9, 0.999)$ [S3, unofficial].

### Edge Cases and Pitfalls
*   **Solid Background**: Without the sparsity prior, NeRF can converge to a solution where unobserved portions or the background are solid (high density), which degrades the spatial resolution of subsequent octree-based representations [S1, unofficial].
*   **Checkpoint Epoch Restoration**: Model loading from checkpoints does not automatically restore the epoch number, requiring manual handling or a TODO fix [S3, unofficial].
*   **Precision**: Denominators in interpolation steps (e.g., calculating finer z-values) must be guarded against zero values (replaced with $10^{-5}$ or similar small constant) to prevent NaN gradients [S3, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.optim as optim

def compute_sparsity_loss(model, bounding_box, K=1000, delta=0.1):
    """
    Computes the sparsity loss by sampling K random points in the bounding box.
    """
    # Sample K random points within the bounding box
    # Assuming bounding_box is defined by min and max coordinates
    random_points = torch.rand(K, 3) * (bounding_box.max - bounding_box.min) + bounding_box.min
    
    # Query density from the model (assuming model.forward returns density)
    # Note: In practice, positional encoding is applied to random_points
    densities = model.query_density(random_points)
    
    # L_sparsity = (1/K) * sum |1 - exp(-delta * sigma_k)|
    sparsity_term = torch.abs(1.0 - torch.exp(-delta * densities))
    loss = torch.mean(sparsity_term)
    return loss

def train_step(model, optimizer, ray_batch, target_color, bounding_box, beta_s=0.01, delta=0.1):
    """
    Performs one training step.
    """
    optimizer.zero_grad()
    
    # 1. Volume Rendering
    # ray_batch contains origin, direction, and other ray info
    # model.forward_coarse and model.forward_fine perform the volume rendering
    predicted_color = model.volume_render(ray_batch)
    
    # 2. RGB Loss (MSE)
    rgb_loss = nn.functional.mse_loss(predicted_color, target_color)
    
    # 3. Sparsity Loss
    sparsity_loss = compute_sparsity_loss(model, bounding_box, K=1000, delta=delta)
    
    # 4. Total Loss
    total_loss = rgb_loss + beta_s * sparsity_loss
    
    # 5. Backpropagation
    total_loss.backward()
    optimizer.step()
    
    return total_loss.item(), rgb_loss.item(), sparsity_loss.item()

def train_neRF(dataset, coarse_model, fine_model, epochs=1000, lr=5e-4, beta_s=0.01, delta=0.1):
    """
    Main training loop.
    """
    # Combine parameters
    all_params = list(coarse_model.parameters()) + list(fine_model.parameters())
    
    # Optimizer
    optimizer = optim.Adam(all_params, lr=lr, betas=(0.9, 0.999))
    
    # Loss function
    mse_loss = nn.MSELoss()
    
    for epoch in range(epochs):
        for batch in dataset:
            # Extract ray batch and target color
            ray_batch = batch['rays']
            target_color = batch['colors']
            
            # Forward pass and loss computation
            loss, rgb_l, spar_l = train_step(
                model=fine_model, # Or combined model wrapper
                optimizer=optimizer,
                ray_batch=ray_batch,
                target_color=target_color,
                bounding_box=dataset.bounding_box,
                beta_s=beta_s,
                delta=delta
            )
        
        # Checkpointing
        if epoch % 10 == 0:
            checkpoint = {
                'epoch': epoch,
                'coarse_model_state_dict': coarse_model.state_dict(),
                'fine_model_state_dict': fine_model.state_dict(),
                'optimizer_state_dict': optimizer.state_dict()
            }
            torch.save(checkpoint, f'checkpoint_epoch_{epoch}.pt')
```

## Subtopics

### [NeuS Training Loss](2535-neus-training-loss.md)
The NeuS training loss is a composite objective function used to optimize neural implicit surface representations (specifically Signed Distance Fields or Unsigned Distance Fields) from multi-view images without 3D ground-truth supervision. It solves the problem of recovering high-fidelity open surfaces by combining photometric consistency with geometric regularization. It distinguishes itself from standard NeRF training [NeRF Training Configuration](1203-nerf-training-configuration.md) by explicitly enforcing the Eikonal equation on the SDF/UDF gradients and using a mask-based binary cross-entropy loss to drive opacity toward binary values, thereby stabilizing the thin-surface approximation.

## Operating characteristics
*   **Time Complexity**: Training time is dominated by the volume rendering step, which requires querying the MLP for $N$ samples per ray. The sparsity loss adds $K$ additional MLP queries per step. With batch size 1, this is computationally expensive compared to rasterization-based methods.
*   **Memory Complexity**: The model stores the weights of the MLPs. The sparsity loss does not require storing intermediate gradients for all $K$ points if computed on the fly, but the checkpoint size grows with the number of layers.
*   **Performance**: The sparsity prior improves the robustness of octree conversion (e.g., PlenOctree) by preventing solid background artifacts [S1, unofficial]. However, it does not directly improve PSNR/SSIM on the training images compared to a model without it, as the primary driver is the RGB loss.
*   **Strengths**: Prevents "ghosting" or solid background artifacts in novel views; facilitates efficient sparse representation conversion.
*   **Limitations**: Requires careful tuning of $\beta_s$ and $\delta$; adds computational overhead per step due to random sampling for sparsity loss.
*   **When to use**: When the downstream application involves converting the NeRF to a sparse representation (like an octree) or when the scene has large unobserved background regions.
*   **When not to use**: If real-time training speed is critical and sparse conversion is not planned, or if the scene is fully enclosed with no background.

## Background
The sparsity prior was introduced in the context of PlenOctrees for real-time rendering of NeRFs [S1, unofficial]. The authors observed that without regularization, NeRFs tend to fill unobserved space with arbitrary geometry, which negatively impacts the efficiency of octree-based compression. The configuration described here, including the specific Adam optimizer settings and batch size, is derived from common PyTorch implementations of NeRF [S3, unofficial]. The sparsity loss formula is also discussed in the context of L0-Sampler, which contrasts its density condensation approach with other sampling refinements [S2].

## Sources
[S1] PlenOctrees For Real-time Rendering of Neural Radiance Fields (NeRFs), http://alexyu.net/plenoctrees/, fetched 2026-10-28 (unofficial)
[S2] L0-Sampler: An L0Model Guided Volume Sampling for NeRF, https://doi.org/10.1109/cvpr52733.2024.02021, fetched 2026-11-04
[S3] charan2598/NeRF-PyTorch, https://github.com/charan2598/NeRF-PyTorch, fetched 2026-10-27 (unofficial)
[S4] Specularity in NeRFs: A Comparative Study of Ref-NeRF and NRFF, https://doi.org/10.5201/ipol.2025.562, fetched 2026-10-26
