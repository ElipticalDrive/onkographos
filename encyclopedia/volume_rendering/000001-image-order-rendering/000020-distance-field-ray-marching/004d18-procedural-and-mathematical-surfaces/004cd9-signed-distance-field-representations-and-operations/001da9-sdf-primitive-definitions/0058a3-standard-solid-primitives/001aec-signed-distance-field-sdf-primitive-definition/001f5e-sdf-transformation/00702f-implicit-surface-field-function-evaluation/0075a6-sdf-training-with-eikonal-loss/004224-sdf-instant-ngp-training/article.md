# SDF-Instant-NGP Training

Also called: SDDF Training, SDF baseline training

## Overview
This method defines the training procedure for a Signed Distance Function (SDF) baseline network, specifically referred to as SDF-Instant-NGP in the source literature [S1]. It is distinguished from its parent method, [SDF Training with Eikonal Loss](2553-sdf-training-with-eikonal-loss.md), by the explicit inclusion of a sign loss term applied to free-space samples to enforce positive SDF values outside the surface, and by the specific hyperparameter configuration (batch size, learning rate, epochs) used for the SDF baseline in the RaDe-GS context [S1].

## How it works

### Inputs and Outputs
*   **Inputs**:
    *   A set of near-surface samples $\mathcal{X}_{\text{near}}$ with corresponding distance labels $y_i \in \mathbb{R}$ [S1].
    *   A set of free-space positions $\mathcal{X}_{\text{free}}$ [S1].
    *   A differentiable SDF model $f(\mathbf{x}; \theta)$ [S1].
*   **Outputs**:
    *   Trained network parameters $\theta$ that minimize the combined loss function [S1].

### Data Structures
The training data consists of two sets of 3D points: $\mathcal{X}_{\text{near}}$ containing near-surface points with distance labels $y_i \in \mathbb{R}$, and $\mathcal{X}_{\text{free}}$ containing free-space positions that provide only sign supervision [S1].

### Numbered Steps
1.  **Sample Generation**: Generate two types of training samples: SDF samples $\mathbf{x}_i \in \mathcal{X}_{\text{near}}$ with distance labels $y_i$, and free space positions $\mathbf{x}_j \in \mathcal{X}_{\text{free}}$ [S1].
2.  **Forward Pass**: For each batch of samples, compute the predicted SDF values $f(\mathbf{x})$ and their gradients $\nabla f(\mathbf{x})$ with respect to the input coordinates using automatic differentiation [S1].
3.  **Loss Calculation**: Compute the three components of the loss function: SDF loss ($l_{\text{SDF}}$) for near-surface samples, sign loss ($l_{\text{sign}}$) for free-space samples, and Eikonal loss ($l_{\text{eik}}$) for all samples [S1].
4.  **Optimization**: Update the network parameters using the Adam optimizer to minimize the total loss [S1].
5.  **Iteration**: Repeat steps 2-4 for 300 epochs [S1].

### Formulas
The total loss is the sum of three terms [S1]:

$$l_{\text{SDF}} = \frac{1}{\|\mathcal{X}_{\text{near}}\|} \sum_{\mathbf{x}_i \in \mathcal{X}_{\text{near}}} |f(\mathbf{x}_i) - y_i| \quad (63)$$

$$l_{\text{sign}} = \frac{1}{\|\mathcal{X}_{\text{free}}\|} \sum_{\mathbf{x}_j \in \mathcal{X}_{\text{free}}} \exp(-\lambda f(\mathbf{x}_j)) \quad (64)$$

$$l_{\text{eik}} = \frac{1}{\|\mathcal{X}_{\text{near}}\| + \|\mathcal{X}_{\text{free}}\|} \sum_{\mathbf{x}_k \in \mathcal{X}_{\text{near}} \cup \mathcal{X}_{\text{free}}} \| |\nabla f(\mathbf{x}_k)| - 1 \| \quad (65)$$

Where:
*   $\|\mathcal{X}_{\text{near}}\|$ and $\|\mathcal{X}_{\text{free}}\|$ are the number of samples in each set [S1].
*   $f(\mathbf{x})$ is the SDF model output [S1].
*   $y_i$ is the ground truth distance label [S1].
*   $\nabla f(\mathbf{x})$ is the gradient of the SDF with respect to the input position [S1].
*   $\lambda$ is a parameter for the sign loss. Not given in the sources.

### Parameters
*   **Batch size**: 60 [S1].
*   **Optimizer**: Adam [S1].
*   **Learning rate**: 0.0001 [S1].
*   **Epochs**: 300 [S1].
*   **$\lambda$**: Value not specified in the source section [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.optim as optim

def compute_sdf_loss(pred_near, labels_near):
    return torch.mean(torch.abs(pred_near - labels_near))

def compute_sign_loss(pred_free, lambda_param):
    return torch.mean(torch.exp(-lambda_param * pred_free))

def compute_eikonal_loss(pred_near, inputs_near, pred_free, inputs_free):
    grad_near = torch.autograd.grad(
        pred_near, inputs_near, grad_outputs=torch.ones_like(pred_near),
        create_graph=False, retain_graph=True, only_inputs=True
    )[0]
    grad_free = torch.autograd.grad(
        pred_free, inputs_free, grad_outputs=torch.ones_like(pred_free),
        create_graph=False, retain_graph=False, only_inputs=True
    )[0]
    grad_norm_near = torch.norm(grad_near, dim=1, keepdim=True)
    grad_norm_free = torch.norm(grad_free, dim=1, keepdim=True)
    return torch.mean(torch.abs(grad_norm_near - 1.0)) + torch.mean(torch.abs(grad_norm_free - 1.0))

def train_sdf_baseline(model, near_samples, near_labels, free_samples, lambda_param,
                       batch_size=60, lr=0.0001, epochs=300):
    optimizer = optim.Adam(model.parameters(), lr=lr)
    for epoch in range(epochs):
        near_indices = torch.randperm(near_samples.size(0))
        free_indices = torch.randperm(free_samples.size(0))
        for i in range(0, near_samples.size(0), batch_size):
            near_idx = near_indices[i:i+batch_size]
            x_near = near_samples[near_idx]
            y_near = near_labels[near_idx]
            
            free_idx = free_indices[i:i+batch_size]
            x_free = free_samples[free_idx]
            
            optimizer.zero_grad()
            pred_near = model(x_near)
            pred_free = model(x_free)
            
            loss_sdf = compute_sdf_loss(pred_near, y_near)
            loss_sign = compute_sign_loss(pred_free, lambda_param)
            loss_eik = compute_eikonal_loss(pred_near, x_near, pred_free, x_free)
            
            total_loss = loss_sdf + loss_sign + loss_eik
            total_loss.backward()
            optimizer.step()
```

### Edge Cases and Pitfalls
The source does not specify the value of $\lambda$, the sampling strategy for $\mathcal{X}_{\text{free}}$, or how to handle precision issues and degenerate inputs [S1].

## Subtopics

### Sphere Loss
*   **What it is**: A method that uses spherical priors or constraints to regularize the SDF.
*   **Difference**: Focuses on geometric priors rather than just point-wise supervision.
*   **When to choose**: When scene geometry can be approximated by spheres or when spherical priors improve convergence.
*   **Link**: [Sphere Loss](2750-sphere-loss.md)

### MLP Distillation
*   **What it is**: A technique where a smaller network is trained to mimic a larger, pre-trained SDF network.
*   **Difference**: Focuses on model compression and inference speed rather than initial training from scratch.
*   **When to choose**: When deployment constraints require a smaller model.
*   **Link**: [MLP Distillation](4829-mlp-distillation.md)

### SDDF with Ellipsoidal Priors and Neural Residuals
*   **What it is**: The primary method in the source paper, which uses ellipsoidal priors and neural residuals to learn a Signed Directional Distance Function.
*   **Difference**: This is the advanced method; SDF-Instant-NGP is the baseline used for comparison.
*   **When to choose**: For higher accuracy and robustness, especially in complex scenes.
*   **Link**: [SDDF with Ellipsoidal Priors and Neural Residuals](10638-sddf-with-ellipsoidal-priors-and-neural-residuals.md)

## Operating characteristics
The method trains for 300 epochs [S1]. The source does not report time or memory complexity, measured performance metrics, specific limitations, or usage guidelines for this baseline [S1].

## Background
The method is described in the paper "Learning Scene-Level Signed Directional Distance Function With Ellipsoidal Priors and Neural Residuals" [S1]. It is presented as a baseline (SDF-Instant-NGP) for the proposed SDDF method in the RaDe-GS framework [S1]. It is used for training a standard SDF network to serve as a comparison point for the more advanced SDDF method [S1].

## Sources

[S1] Learning Scene-Level Signed Directional Distance Function With Ellipsoidal Priors and Neural Residuals., https://doi.org/10.1109/tpami.2026.3688658, fetched 2026-10-26
