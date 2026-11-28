# TetraSDF Training Procedure

Also called: SDF neural network training, TetraSDF optimization

## Overview
TetraSDF Training Procedure optimizes a neural network to represent a signed distance field (SDF) using a multi-resolution tetrahedral grid, combining L1 regression, Eikonal regularization, and weight normalization constraints [S1, unofficial]. It differs from standard MLP-based SDF training by utilizing a tetrahedral grid structure for spatial representation, which allows for efficient multi-resolution handling and specific mesh extraction via sub-polygonization [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input**: A dataset of 3D points with ground truth SDF values, typically sampled from a mesh (e.g., Stanford 3D scanning models) [S1, unofficial].
*   **Output**: A trained `TetraSDF` neural network model capable of evaluating SDF values and gradients at arbitrary 3D coordinates [S1, unofficial].

**Data Structures**
*   **TetraSDF Network**: A neural network defined by `num_layers`, `num_hidden`, `levels`, `r_min`, `r_max`, and `T` [S1, unofficial].
    *   `num_layers`: Number of hidden layers (default 4) [S1, unofficial].
    *   `num_hidden`: Number of hidden units per layer (default 12) [S1, unofficial].
    *   `levels`: Multi-resolution levels (default 4) [S1, unofficial].
    *   `r_min`, `r_max`: Resolution bounds for the tetrahedral grid, dependent on the chosen resolution preset (small, medium, large) [S1, unofficial].
    *   `T`: A parameter related to the tetrahedral grid structure (default 21) [S1, unofficial].
    *   `use_A`: Boolean flag for using a preconditioner (default True) [S1, unofficial].
*   **StanfordDataset**: A custom dataset class that provides point-cloud samples and ground truth SDF labels [S1, unofficial].
*   **DataLoader**: A standard iterator that shuffles data and yields batches of size `BATCH_SIZE` [S1, unofficial].

**Parameters**
*   `BATCH_SIZE`: 1000 [S1, unofficial].
*   `lr` (Adam learning rate): $1 \times 10^{-3}$ [S1, unofficial].
*   `eikonal_weight`: $5 \times 10^{-3}$ [S1, unofficial].
*   `EPOCH`: 10 [S1, unofficial].
*   `T_max` (for scheduler): $EPOCH \times \frac{\text{len(training\_data)}}{BATCH\_SIZE}$ [S1, unofficial].
*   Resolution presets:
    *   `small`: $r_{min}=2, r_{max}=32$ [S1, unofficial].
    *   `medium`: $r_{min}=4, r_{max}=64$ [S1, unofficial].
    *   `large`: $r_{min}=8, r_{max}=128$ [S1, unofficial].

**Numbered Steps**
1.  **Initialization**:
    *   Set random seeds for Python, NumPy, and PyTorch [S1, unofficial].
    *   Create the `TetraSDF` network instance with specified hyperparameters [S1, unofficial].
    *   Load the training dataset (`StanfordDataset`) and create a `DataLoader` with `BATCH_SIZE=1000` and `shuffle=True` [S1, unofficial].
    *   Initialize the Adam optimizer with `lr=1e-3` [S1, unofficial].
    *   Initialize the `CosineAnnealingLR` scheduler with `T_max` calculated as above [S1, unofficial].
    *   Define the L1 Loss criterion (`nn.L1Loss`) [S1, unofficial].
2.  **Epoch Loop**: Iterate for `EPOCH` times (default 10) [S1, unofficial].
    *   **Resampling**: If `epoch == 0`, call `train_dataloader.dataset.resample()` to resample the dataset [S1, unofficial].
3.  **Batch Loop**: Iterate through each batch in `train_dataloader` [S1, unofficial].
    *   **Data Loading**: Retrieve `inputs` (points) and `labels` (ground truth SDF) from the batch and move them to the GPU [S1, unofficial].
    *   **Gradient Zeroing**: Reset accumulated gradients to zero [S1, unofficial].
    *   **Forward Pass**: Compute predicted SDF values: `pred_sdf = net.sdf(inputs)[:, 0]` [S1, unofficial].
    *   **L1 Loss Calculation**: Compute $L_{L1} = \frac{1}{N} \sum_{i=1}^N | \hat{y}_i - y_i |$ where $\hat{y}_i$ is `pred_sdf` and $y_i$ is `labels` [S1, unofficial].
    *   **Eikonal Loss Calculation**:
        *   Set `points = inputs.detach().clone().requires_grad_(True)` [S1, unofficial].
        *   Compute the gradient of the SDF output with respect to the input points: `J = torch.autograd.grad(net.sdf(points).sum(), points, create_graph=True, retain_graph=True)[0]` [S1, unofficial].
        *   Compute the Eikonal loss: $L_{eik} = \mathbb{E} [ (\| \nabla \phi(\mathbf{x}) \|_2 - 1)^2 ]$ using `eik = (J.norm(p=2, dim=1) - 1.0).pow(2).mean()` [S1, unofficial].
    *   **Weight Norm Loss Calculation**:
        *   Compute $L_{wn} = \frac{1}{M} \sum_{i=1}^M \mathbb{E} [ (1 - \| \mathbf{W}_i \|_2)^2 ]$ where $\mathbf{W}_i$ are the layer weights in `net.fc` [S1, unofficial].
    *   **Total Loss**: Sum the losses: `loss = L1_Loss + eikonal_weight * L_eik + L_wn` [S1, unofficial].
    *   **Backward Pass**: Compute gradients of the total loss with respect to all network parameters [S1, unofficial].
    *   **Optimizer Step**: Update network parameters using the Adam algorithm [S1, unofficial].
    *   **Scheduler Step**: Update the learning rate using cosine annealing [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
from torch.utils.data import DataLoader

# Standard math operations
def l1_loss(pred, target):
    return torch.mean(torch.abs(pred - target))

def weight_norm_loss(fc_layer):
    total = 0.0
    count = 0
    for param in fc_layer.parameters():
        if param.dim() > 1:
            # Compute L2 norm per output neuron (row-wise for shape (out, in))
            row_norms = torch.norm(param, p=2, dim=1)
            total += torch.mean((1.0 - row_norms) ** 2)
            count += 1
    return total / count if count > 0 else 0.0

def adam_step(params, grads, state, lr, beta1=0.9, beta2=0.999, eps=1e-8):
    for p, g in zip(params, grads):
        if p.grad is None: continue
        g = g.data
        state[p] = state.get(p, {'m': 0.0, 'v': 0.0})
        m, v = state[p]['m'], state[p]['v']
        state[p]['m'] = beta1 * m + (1 - beta1) * g
        state[p]['v'] = beta2 * v + (1 - beta2) * g ** 2
        m_hat = state[p]['m'] / (1 - beta1 ** (state[p].get('step', 0) + 1))
        v_hat = state[p]['v'] / (1 - beta2 ** (state[p].get('step', 0) + 1))
        p.data -= lr * m_hat / (torch.sqrt(v_hat) + eps)
        state[p]['step'] = state[p].get('step', 0) + 1

def cosine_annealing_step(optimizer, epoch, T_max, lr_min=0):
    lr = lr_min + (1 - optimizer.param_groups[0]['lr']) * (1 + math.cos(math.pi * epoch / T_max)) / 2
    for group in optimizer.param_groups:
        group['lr'] = lr

BATCH_SIZE = 1000
EPOCH = 10
eikonal_weight = 5e-3

for epoch in range(EPOCH):
    running_loss = 0.0
    running_l1_loss = 0.0
    
    if epoch == 0:
        train_dataloader.dataset.resample()
        
    for i, data in enumerate(train_dataloader, 0):
        inputs, labels = data
        inputs = inputs.cuda()
        labels = labels.cuda()
        
        optimizer.zero_grad()
        
        pred_sdf = net.sdf(inputs)[:, 0]
        
        l1_loss_val = l1_loss(pred_sdf, labels)
        
        points = inputs.detach().clone().requires_grad_(True)
        J = torch.autograd.grad(net.sdf(points).sum(), points, create_graph=True, retain_graph=True)[0]
        eik = (J.norm(p=2, dim=1) - 1.0).pow(2).mean()
        
        wn_loss_val = weight_norm_loss(net.fc)
        
        loss = l1_loss_val + eikonal_weight * eik + wn_loss_val
        
        loss.backward()
        
        adam_step(net.parameters(), net.parameters(), optimizer.state_dict(), lr=1e-3)
        
        cosine_annealing_step(optimizer, epoch, T_max)
        
        running_loss += loss.item()
        running_l1_loss += l1_loss_val.item()
```

**Edge Cases and Pitfalls**
*   **Sharp Slopes**: Without the Eikonal loss, the estimated SDF values develop sharp slopes [S1, unofficial]. This is mitigated by adding $L_{eik}$ with a weight of $5 \times 10^{-3}$ [S1, unofficial].
*   **Precision**: The use of `detach().clone().requires_grad_(True)` is critical for computing the Eikonal loss correctly without interfering with the main gradient flow for the L1 loss [S1, unofficial].

## Operating characteristics
*   **Time Complexity**: Not given in the sources [S1, unofficial].
*   **Memory Complexity**: Not given in the sources [S1, unofficial].
*   **Measured Performance**: Not given in the sources [S1, unofficial].
*   **Scaling**: Not given in the sources [S1, unofficial].
*   **Strengths**: Combines L1 regression, Eikonal regularization, and weight normalization for robust SDF learning [S1, unofficial].
*   **Limitations**: Requires specific dataset resampling on the first epoch [S1, unofficial].
*   **When to use**: When training a TetraSDF model for 3D shape representation [S1, unofficial].

## Background
TetraSDF was developed by NAVER Corp. and released as an open-source project [S1, unofficial]. The training procedure is part of the TetraSDF framework, which uses a multi-resolution tetrahedral grid for SDF representation [S1, unofficial]. The method is designed to work with Stanford 3D scanning models [S1, unofficial].

## Sources
[S1] naver-ai/TetraSDF, https://github.com/naver-ai/TetraSDF, fetched 2026-10-27 (unofficial)
