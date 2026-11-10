# NeRF Training Configuration

Also called: NeRF Training Loop Configuration, NeRF Training Setup, Sparsity Prior, Sparsity Regularization

## Overview
NeRF Training Configuration defines the optimization loop, data handling, and regularization strategies required to train a Neural Radiance Field (NeRF) model for novel view synthesis [S2, unofficial]. It solves the problem of converging a continuous volumetric representation to match observed multi-view images while preventing the generation of arbitrary, non-photorealistic geometry in unobserved regions [S1, unofficial]. It distinguishes itself from sibling methods like [NeRF-SH](1348-nerf-sh.md) by focusing on the standard MLP architecture and training dynamics, and from [NeuS Training Loss](2535-neus-training-loss.md) by relying on volumetric opacity rather than explicit surface constraints like the Eikonal equation [S1, unofficial][S2, unofficial].

## How it works
The training process involves loading datasets, initializing coarse and fine NeRF models, and optimizing them using a composite loss function that includes a sparsity prior [S2, unofficial].

**Inputs and Outputs:**
- **Inputs:** Multi-view images of a static scene, camera poses (extrinsics and intrinsics), and bounding box defining the scene volume [S2, unofficial].
- **Outputs:** Optimized NeRF models (coarse and fine) and associated optimizer states [S2, unofficial].

**Data Structures:**
- **Dataset:** Images are resized such that the shorter side is 400 pixels [S2, unofficial]. Camera intrinsics are derived from height ($H$), width ($W$), and focal length ($f$) [S2, unofficial]. The intrinsic matrix $K$ is:
  $$
  K = \begin{bmatrix} f & 0 & 0.5W \\ 0 & f & 0.5H \\ 0 & 0 & 1 \end{bmatrix}
  $$
  [S2, unofficial].
- **Checkpoint:** Saved as a `.pt` file containing a dictionary with keys: `'epoch'`, `'coarse_model_state_dict'`, `'fine_model_state_dict'`, and `'optimizer_state_dict'` [S2, unofficial]. Note that model loading does not restore the epoch number from the checkpoint [S2, unofficial].

**Numbered Steps:**
1. Load train/val/test datasets using `BlenderDataset` with specified resolution [S2, unofficial].
2. Extract camera intrinsics ($H$, $W$, $f$) [S2, unofficial].
3. Create `DataLoader` with `batch_size=1` and `shuffle=True` for training [S2, unofficial].
4. Initialize coarse and fine NeRF models [S2, unofficial].
5. Combine parameters from both models [S2, unofficial].
6. Create Adam optimizer with `lr=5e-4` [S2, unofficial].
7. Create `MSELoss` [S2, unofficial].
8. Initialize `NeRFTrainer` with models, optimizer, loss, and dataloader [S2, unofficial].
9. Call `trainer.train()` [S2, unofficial].

**Formulas:**
The total loss $\mathcal{L}$ is a combination of the RGB loss and the sparsity loss:
$$
\mathcal{L} = \mathcal{L}_{\text{RGB}} + \lambda_{\text{sparsity}} \mathcal{L}_{\text{sparsity}}
$$

The RGB loss is the sum of squared errors over a batch of rays $\mathcal{R}$:
$$
\mathcal{L}_{\text{RGB}} = \sum_{\mathbf{r} \in \mathcal{R}} \| C(\mathbf{r}) - \hat{C}(\mathbf{r}) \|_2^2 \quad [S1, unofficial]
$$
where $C(\mathbf{r})$ is the target color and $\hat{C}(\mathbf{r})$ is the rendered color [S1, unofficial].

The sparsity loss $\mathcal{L}_{\text{sparsity}}$ is defined as:
$$
\mathcal{L}_{\text{sparsity}} = \frac{1}{K} \sum_{k=1}^K |1 - \exp(-\lambda \sigma_k)| \quad [S1, unofficial]
$$
where:
- $K$ is the number of uniformly random points sampled within the bounding box [S1, unofficial].
- $\sigma_k$ is the density value at the $k$-th random point [S1, unofficial].
- $\lambda$ is a hyperparameter controlling the strength of the sparsity prior [S1, unofficial].

**Parameters:**
- **Learning Rate:** $5 \times 10^{-4}$ [S2, unofficial].
- **Batch Size:** 1 [S2, unofficial].
- **Image Resolution:** 400 pixels on the shorter side [S2, unofficial].
- **Optimizer:** Adam with betas $(0.9, 0.999)$ [S2, unofficial].
- **Sparsity Hyperparameter:** $\lambda$ is used in the exponential term; specific value not given in the sources [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Solid Background:** Without the sparsity prior, the model may converge to a solution where unobserved portions or the background are solid, which degrades the spatial resolution of subsequent octree-based representations (e.g., PlenOctrees) [S1, unofficial].
- **Checkpoint Epoch:** When loading a checkpoint, the epoch number is not restored from the file, requiring manual handling or resetting [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.optim as optim

def compute_sparsity_loss(density_samples, lambda_param):
    """Computes L_sparsity = (1/K) * sum |1 - exp(-lambda * sigma_k)|"""
    return torch.mean(torch.abs(1.0 - torch.exp(-lambda_param * density_samples)))

def sample_random_points_in_bbox(bbox_min, bbox_max, num_points):
    """Uniformly samples num_points within the bounding box."""
    return torch.rand(num_points, 3) * (bbox_max - bbox_min) + bbox_min

class NeRFMLP(nn.Module):
    """Simplified NeRF MLP returning color and density."""
    def __init__(self):
        super().__init__()
        self.fc = nn.Linear(6, 4)
    def forward(self, origins, directions):
        x = torch.cat([origins, directions], dim=-1)
        h = torch.relu(self.fc(x))
        return torch.sigmoid(h[:, :3]), torch.relu(h[:, 3:])
    def get_density(self, points):
        return torch.relu(self.fc(points))[:, 0:1]

class BlenderDataset(torch.utils.data.Dataset):
    """Simulated dataset loader returning images, origins, directions."""
    def __init__(self, resolution, split):
        self.resolution = resolution
        self.split = split
    def __len__(self):
        return 10
    def __getitem__(self, idx):
        return {
            'image': torch.zeros(400, 400, 3),
            'origins': torch.zeros(1, 3),
            'directions': torch.zeros(1, 3)
        }

class NeRFTrainer:
    """Wraps models, optimizer, loss, and dataloader for training."""
    def __init__(self, models, optimizer, loss_fn, dataloader, config):
        self.models = models
        self.optimizer = optimizer
        self.loss_fn = loss_fn
        self.dataloader = dataloader
        self.config = config
    def train(self):
        num_epochs = self.config.get('num_epochs', 1000)
        lambda_sparsity = self.config.get('lambda_sparsity', 1.0)
        bbox_min = torch.tensor(self.config['bbox_min'])
        bbox_max = torch.tensor(self.config['bbox_max'])
        K_points = 1000
        for epoch in range(num_epochs):
            for model in self.models:
                model.train()
            for batch in self.dataloader:
                images = batch['image']
                origins = batch['origins']
                directions = batch['directions']
                self.optimizer.zero_grad()
                coarse_color, coarse_density = self.models[0](origins, directions)
                fine_color, fine_density = self.models[1](origins, directions)
                loss_rgb = self.loss_fn(fine_color, images)
                random_points = sample_random_points_in_bbox(bbox_min, bbox_max, K_points)
                random_densities = self.models[0].get_density(random_points)
                loss_sparsity = compute_sparsity_loss(random_densities, lambda_sparsity)
                loss = loss_rgb + loss_sparsity
                loss.backward()
                self.optimizer.step()
            checkpoint = {
                'epoch': epoch,
                'coarse_model_state_dict': self.models[0].state_dict(),
                'fine_model_state_dict': self.models[1].state_dict(),
                'optimizer_state_dict': self.optimizer.state_dict()
            }
            torch.save(checkpoint, f'checkpoint_epoch_{epoch}.pt')

def train_nerf(config):
    # 1. Load datasets
    train_dataset = BlenderDataset(resolution=config['resolution'], split='train')
    val_dataset = BlenderDataset(resolution=config['resolution'], split='val')
    test_dataset = BlenderDataset(resolution=config['resolution'], split='test')
    # 2. Extract camera intrinsics
    H, W, f = config['height'], config['width'], config['focal_length']
    K = torch.tensor([[f, 0, 0.5 * W], [0, f, 0.5 * H], [0, 0, 1]])
    # 3. Create DataLoader
    train_loader = torch.utils.data.DataLoader(train_dataset, batch_size=1, shuffle=True)
    # 4. Initialize models
    coarse_model = NeRFMLP()
    fine_model = NeRFMLP()
    # 5. Combine parameters
    all_params = list(coarse_model.parameters()) + list(fine_model.parameters())
    # 6. Create optimizer
    optimizer = optim.Adam(all_params, lr=5e-4, betas=(0.9, 0.999))
    # 7. Create loss
    mse_loss = nn.MSELoss()
    # 8. Initialize trainer
    trainer = NeRFTrainer([coarse_model, fine_model], optimizer, mse_loss, train_loader, config)
    # 9. Call trainer.train()
    trainer.train()
```

## Subtopics

### [NeuS Training Loss](2535-neus-training-loss.md)
The NeuS training loss is a composite objective function used to optimize neural implicit surface representations (specifically Signed Distance Fields or Unsigned Distance Fields) from multi-view images without 3D ground-truth supervision. It solves the problem of recovering high-fidelity open surfaces by combining photometric consistency with geometric regularization. It distinguishes itself from standard NeRF training [NeRF Training Configuration](1203-nerf-training-configuration.md) by explicitly enforcing the Eikonal equation on the SDF/UDF gradients and using a mask-based binary cross-entropy loss to drive opacity toward binary values, thereby stabilizing the thin-surface approximation.

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Measured Performance:** On NeRF-synthetic test scenes, the approach achieves comparable image quality to state-of-the-art methods like NSVF, with PSNR values around 31.71 and SSIM around 0.958 after fine-tuning [S1, unofficial].
- **Strengths:** The sparsity prior prevents the model from learning arbitrary geometry in unobserved regions, which is crucial for efficient octree-based compression and rendering [S1, unofficial].
- **Limitations:** Without the sparsity prior, the model may converge to solid backgrounds, degrading the quality of subsequent octree conversions [S1, unofficial]. The method relies on careful tuning of the sparsity hyperparameter $\lambda$ [S1, unofficial].
- **When to use:** Use when the goal is to train a standard NeRF model that will be converted to an octree-based representation (e.g., PlenOctrees) for real-time rendering [S1, unofficial].
- **When not to use:** Use [NeuS Training Loss](2535-neus-training-loss.md) if explicit surface reconstruction is required rather than volumetric rendering [S1, unofficial].

## Background
The NeRF Training Configuration, particularly the inclusion of the sparsity prior, was introduced in the context of PlenOctrees for real-time rendering of Neural Radiance Fields [S1, unofficial]. The work demonstrated that while arbitrary geometry in unobserved regions does not directly worsen image quality metrics, it adversely impacts the conversion process to octree-based representations by occupying significant voxel space [S1, unofficial]. Implementation details are documented in a PyTorch repository [S2, unofficial].

## Sources
[S1] PlenOctrees For Real-time Rendering of Neural Radiance Fields (NeRFs), http://alexyu.net/plenoctrees/, fetched 2026-10-28 (unofficial)
[S2] charan2598/NeRF-PyTorch, https://github.com/charan2598/NeRF-PyTorch, fetched 2026-10-27 (unofficial)
