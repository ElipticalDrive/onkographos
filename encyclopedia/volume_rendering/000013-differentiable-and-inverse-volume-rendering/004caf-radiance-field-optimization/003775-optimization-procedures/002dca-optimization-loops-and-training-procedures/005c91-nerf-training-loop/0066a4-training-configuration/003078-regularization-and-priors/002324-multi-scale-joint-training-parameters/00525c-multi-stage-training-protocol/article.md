# Multi-stage Training Protocol

Also called: Pretrain and Joint Train Strategy

## Overview
Multi-stage training protocols decompose the optimization of complex neural radiance fields or inverse rendering models into sequential phases to stabilize convergence and manage conflicting gradients. Unlike single-stage end-to-end training, which often suffers from instability when optimizing heterogeneous components (e.g., geometry, appearance, and lighting) simultaneously, this method decouples the process into distinct steps such as pretraining, joint training, and fine-tuning. It addresses the problem of initializing dependent modules with reasonable values before introducing more complex interactions, ensuring that the model components act as expected during the final end-to-end optimization.

## How it works
The protocol generally follows a progressive scheme where specific submodules are trained, frozen, or jointly optimized in a defined order. The exact steps vary by implementation, but the core logic involves warming up coarse representations before refining high-frequency details or coupled variables.

### Inputs and Outputs
*   **Inputs:** Camera poses, images, and optionally auxiliary supervision (e.g., depth, normals, albedo).
*   **Outputs:** Optimized neural field parameters (MLP weights, grid features, or volume densities) and associated attributes (lighting, material properties).

### Data Structures
Data structures depend on the specific variant (e.g., Direct Prediction Module vs. UV Volumes). Common structures include:
*   **Neural Networks:** MLPs for direct prediction or lighting estimation.
*   **Volumetric/Grid Representations:** 3D volumes (e.g., $128^3$) or UV-metric grids for storing features.
*   **Loss Weights:** Scalars $\lambda$ for balancing multi-task losses.

### Steps
The general multi-stage protocol consists of the following phases:

1.  **Pretrain Direct Prediction Module:**
    *   Train the albedo, normal, and depth branches of the Direct Prediction Module independently.
    *   **Goal:** Ensure these branches produce reasonable initial values, as they are dependencies for subsequent modules (e.g., Lighting Joint Prediction Module).
    *   **Loss:** Standard reconstruction or supervision losses for albedo, normal, and depth.

2.  **Joint Train Direct and Lighting Modules:**
    *   Jointly optimize the Direct Prediction Module and the Lighting Joint Prediction Module.
    *   **Loss:** Multi-task loss $L$ combining albedo, normal, depth, lighting, visibility, and regularization terms:
        $$ L = \lambda_A L_{albedo} + \lambda_N L_{normal} + \lambda_D L_{depth} + \lambda_L L_{light} + \lambda_{visible} L_{visible} + \lambda_{reg} L_{reg} + \lambda_{rerender} L_{rerender} $$
        where $\lambda$ are hyperparameters balancing the terms [S1].

3.  **Train Joint Re-prediction Module:**
    *   Freeze the weights of the Direct Prediction Module and Lighting Joint Prediction Module.
    *   Train the Joint Re-prediction Module to refine albedo, normals, and depth.
    *   **Goal:** Improve the consistency of intermediate outputs without disturbing the already learned lighting and direct appearance features.

4.  **End-to-End Finetune:**
    *   Unfreeze all modules and jointly finetune the entire network end-to-end.
    *   **Loss:** Multi-task loss computed on both the joint prediction outputs $(\hat{A}, \hat{N}, \hat{D}, \hat{L})$ and the intermediate outputs $(\tilde{A}, \tilde{N}, \tilde{D})$.
    *   **Goal:** Refine the global consistency of the model.

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def multi_stage_training(model, data_loader, config):
    # Step 1: Pretrain Direct Prediction Module
    # Train albedo, normal, and depth branches of Direct Prediction Module
    for epoch in range(config.pretrain_epochs):
        optimizer_direct.zero_grad()
        outputs = model.direct_prediction_module(data)
        loss = compute_direct_loss(outputs, data)
        loss.backward()
        optimizer_direct.step()
    
    # Step 2: Joint Train Direct and Lighting Modules
    # Freeze nothing yet, optimize both modules
    for epoch in range(config.joint_epochs):
        optimizer_joint.zero_grad()
        direct_out = model.direct_prediction_module(data)
        lighting_out = model.lighting_joint_prediction_module(direct_out, data)
        loss = compute_multi_task_loss(direct_out, lighting_out, data)
        loss.backward()
        optimizer_joint.step()
        
        # Apply learning rate decay if applicable
        adjust_learning_rate(optimizer_joint, epoch, config.lr_decay)

    # Step 3: Train Joint Re-prediction Module
    # Freeze weights of Direct and Lighting modules
    freeze_model(model.direct_prediction_module)
    freeze_model(model.lighting_joint_prediction_module)
    
    for epoch in range(config.repred_epochs):
        optimizer_repred.zero_grad()
        # Only re-prediction module gradients flow
        repred_out = model.joint_reprediction_module(data)
        loss = compute_repred_loss(repred_out, data)
        loss.backward()
        optimizer_repred.step()

    # Step 4: End-to-End Finetune
    # Unfreeze all modules
    unfreeze_model(model.direct_prediction_module)
    unfreeze_model(model.lighting_joint_prediction_module)
    unfreeze_model(model.joint_reprediction_module)
    
    for epoch in range(config.finetune_epochs):
        optimizer_e2e.zero_grad()
        # Forward pass through all modules
        direct_out = model.direct_prediction_module(data)
        lighting_out = model.lighting_joint_prediction_module(direct_out, data)
        repred_out = model.joint_reprediction_module(data)
        
        # Loss on both joint predictions and intermediate outputs
        loss = compute_e2e_loss(direct_out, lighting_out, repred_out, data)
        loss.backward()
        optimizer_e2e.step()
        
        adjust_learning_rate(optimizer_e2e, epoch, config.lr_decay)
```

### Parameters
*   **Learning Rate Decay:** Exponential decay is often used. For example, in UV Volume variants, decay rates $\gamma$ are applied [S2].
*   **Loss Weights:** $\lambda_A, \lambda_N, \lambda_D, \lambda_L, \lambda_{visible}, \lambda_{reg}, \lambda_{rerender}$ balance the multi-task loss [S1].
*   **Ray Sampling:** 64 points sampled along each camera ray between near and far bounds [S2].
*   **Optimizer:** Adam optimizer [S2].

### Edge Cases and Pitfalls
*   **Dependency Violation:** If the Direct Prediction Module is not pretrained, the Lighting Joint Prediction Module may receive unreasonable inputs, leading to poor convergence [S1].
*   **Freezing Timing:** Freezing modules too early may prevent necessary co-adaptation; freezing too late may lead to instability. The protocol specifies freezing after Step 2 and unfreezing in Step 4 [S1].
*   **UV Volume Warm-start:** In UV-based methods, UV volumes must be learned first to warm-start coarse geometry before optimizing the Neural Texture Stack (NTS) for high-frequency signals [S2].

## Subtopics

### [MobileNeRF Staged Training](15503-mobilenerf-staged-training.md)
MobileNeRF Staged Training is a multi-stage optimization protocol designed to train the MobileNeRF neural field representation efficiently on mobile hardware constraints. It addresses the challenge of jointly optimizing geometry (vertices), appearance (MLPs), and acceleration structures (grids) by decoupling them into sequential phases. Unlike standard NeRF training which optimizes all parameters simultaneously, this method fixes geometry and acceleration structures in later stages to stabilize convergence and reduce computational overhead during the final refinement of appearance features.

### [Multi-resolution Feature Grid Pre-train](17802-multi-resolution-feature-grid-pre-train.md)
Multi-resolution Feature Grid Pre-train is a method that constructs a multi-resolution ground feature plane representation for large urban scenes, capturing density and appearance separately to reduce memory footprint and guide subsequent NeRF sampling. It differs from standard NeRF training by decoupling the initial scene representation into a compact, low-rank tensor factorization (the grid branch) that is pre-trained before joint optimization with a full 3D NeRF branch. This approach leverages the planar nature of urban environments to achieve robust performance with sparse view data.#### UV Volume Training Strategy
The UV Volume Training Strategy uses two optimization strategies for UV volumes and the Neural Texture Stack (NTS) branch. UV volumes are learned primarily in early stages to warm-start coarse geometry, while NTS is optimized later to fit high-frequency signals. The learning rate for UV volumes is calculated as $l_{uv} = 1 \times 10^{-3} \beta^{epoch}$, where $\beta = 1 \times 10^{-1}$. The learning rate for NTS is calculated as $l_{nts} = 5 \times 10^{-4} \beta^{epoch}$, where $\beta = 1 \times 10^{-1}$. Ray sampling uses 64 points along each camera ray between near and far bounds, with scene bounds estimated based on the SMPL model. The Adam optimizer is used. Training on 26-view videos of 100 frames at 960 × 540 resolution takes approximately 20 hours (200k iterations) on a single NVIDIA A100 GPU.

## Operating characteristics
*   **Time Complexity:** Multi-stage training increases total training time compared to single-stage methods due to sequential phases. For example, UV Volume training takes ~20 hours for 200k iterations [S2].
*   **Memory Complexity:** Depends on the representation (e.g., $128^3$ volume or UV grids). Pretraining stages may require less memory if only subsets of parameters are active.
*   **Strengths:** Stabilizes convergence for complex models with dependent components; allows for specialized optimization strategies for different parts of the network (e.g., geometry vs. appearance).
*   **Limitations:** Increased training time; requires careful tuning of stage transitions and learning rates; potential for suboptimal solutions if stages are not well-coordinated.
*   **When to Use:** When training complex inverse rendering models or neural fields with heterogeneous components (e.g., lighting, material, geometry) that conflict during joint optimization.
*   **When Not to Use:** For simple NeRFs where single-stage training converges reliably; when computational resources are extremely limited and training time is a critical constraint.

## Background
The multi-stage training protocol originated in the context of inverse rendering and neural radiance field optimization, where models often comprise multiple interconnected modules (e.g., direct prediction, lighting estimation, re-prediction). The approach was formalized in works such as "Learning Indoor Inverse Rendering with 3D Spatially-Varying Lighting" [S1] and "UV Volumes for Real-time Rendering of Editable Free-view Human Performance" [S2]. It addresses the instability observed in end-to-end training of such complex systems by progressively introducing model complexity.

## Sources
[S1] Learning Indoor Inverse Rendering with 3D Spatially-Varying Lighting, https://doi.org/10.1109/iccv48922.2021.01231, fetched 2026-10-26
[S2] UV Volumes for Real-time Rendering of Editable Free-view Human Performance, https://doi.org/10.1109/cvpr52729.2023.01595, fetched 2026-10-26
