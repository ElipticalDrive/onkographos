# Geodesic Loss Approximation for 2D/3D Registration

Also called: Geodesic Loss, ProST Geodesic Loss

## Overview
This method trains a deep neural network to approximate the geodesic gradient in SE(3), enabling end-to-end learning of a convex similarity metric for 2D/3D registration [S1]. It differs from standard supervised registration by optimizing the network parameters $\phi$ such that the gradient of the learned loss $L_N$ with respect to pose $\theta$ aligns directionally with the true geodesic gradient $\partial L_G / \partial \theta$, rather than minimizing pixel-wise error directly [S1]. This approach extends the capture range of initialization compared to traditional image-based methods like Grad-NCC [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** 3D volume $V$, pose parameter $\theta \in SE(3)$, fixed target image $I_f$, target pose $\theta_t$ (for geodesic calculation), and network parameters $\phi$.
*   **Outputs:** Network parameters $\phi$ (trained), pose parameter $\theta$ (optimized during application).

**Data Structures:**
*   **Pose $\theta$:** Represented in SE(3). The gradient $\partial L / \partial \theta$ is decomposed into translation part $v^t$ and rotation part $v^r$. The rotation vector is transformed into Rodrigues angle-axis representation [S1].
*   **Gradients:** $\frac{\partial L_N}{\partial \theta}$ and $\frac{\partial L_G}{\partial \theta}$ are 6-dimensional vectors (3 translation, 3 rotation) [S1].

**Numbered Steps:**

1.  **Geodesic Gradient Calculation:** Compute the geodesic gradient $\frac{\partial L_G(\theta, \theta_t)}{\partial \theta}$ using the implementation from [10] (cited in [S1]), where $L_G$ is the square of the geodesic distance in SE(3) [S1].
2.  **Forward Pass (Network):** Pass $V$, $\theta$, and $I_f$ through the deep neural network (ProST-based architecture) to obtain embedded features $e_m$ (from volume) and $e_f$ (from target image) [S1].
3.  **Network Loss Calculation:** Compute the network loss $L_N$ as the mean squared error between embedded features:
    $$L_N = \|e_m - e_f\|^2$$
    [S1]
4.  **Network Gradient Calculation:** Compute the gradient of the network loss with respect to pose $\theta$ using back-propagation through the differentiable volume rendering layer (ProST) [S1]:
    $$\frac{\partial L_N}{\partial \theta} = \frac{\partial L_N}{\partial I_m} \cdot \frac{\partial I_m}{\partial G_S} \cdot \frac{\partial G_S}{\partial G_T} \cdot \frac{\partial G_T}{\partial T(\theta)} \cdot \frac{\partial T(\theta)}{\partial \theta}$$
    [S1]
5.  **Directional Distance Calculation:** Decompose $\frac{\partial L_N}{\partial \theta}$ and $\frac{\partial L_G}{\partial \theta}$ into translation ($v_1^t, v_2^t$) and rotation ($v_1^r, v_2^r$) components [S1]. Compute the directional distance metric $M_{dist}$:
    $$M_{dist}^{trans} = \left\| \frac{v_1^t}{\|v_1^t\|} - \frac{v_2^t}{\|v_2^t\|} \right\|^2$$
    $$M_{dist}^{rot} = \left\| \frac{v_1^r}{\|v_1^r\|} - \frac{v_2^r}{\|v_2^r\|} \right\|^2$$
    $$M_{dist} = M_{dist}^{trans} + M_{dist}^{rot}$$
    [S1]
6.  **Double Backward Pass:** Perform a second back-propagation pass to compute $\frac{\partial M_{dist}}{\partial \phi}$ for updating network parameters $\phi$ [S1].
7.  **Network Update:** Update $\phi$ using gradient descent on $M_{dist}$ [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import numpy as np

def compute_geodesic_gradient(theta, theta_t):
    """
    Computes the geodesic gradient dL_G/d_theta.
    Implementation based on [10] as cited in [S1].
    Returns a 6D vector: [v_trans, v_rot] in Rodrigues angle-axis.
    """
    # Placeholder for the specific implementation from [10]
    # This function must calculate the gradient of the squared geodesic distance
    # in SE(3) between current pose theta and target pose theta_t.
    raise NotImplementedError("Implementation from [10] required")

def compute_network_gradient(theta, V, I_f, network, phi):
    """
    Computes dL_N/d_theta via back-propagation through the network.
    """
    # Forward pass
    e_m, e_f = network(V, theta, I_f, phi)
    L_N = torch.mean((e_m - e_f)**2)
    
    # Backward pass to get gradient w.r.t theta
    # This relies on the differentiable ProST layer
    L_N.backward()
    grad_theta = theta.grad.clone()
    
    # Clear gradients for next step
    theta.grad.zero_()
    return grad_theta, L_N

def compute_directional_distance(grad_N, grad_G):
    """
    Computes M_dist between network gradient and geodesic gradient.
    """
    # Split into translation (first 3) and rotation (last 3)
    v1_t, v1_r = grad_N[:3], grad_N[3:]
    v2_t, v2_r = grad_G[:3], grad_G[3:]
    
    # Normalize
    v1_t_norm = v1_t / torch.norm(v1_t)
    v2_t_norm = v2_t / torch.norm(v2_t)
    v1_r_norm = v1_r / torch.norm(v1_r)
    v2_r_norm = v2_r / torch.norm(v2_r)
    
    # Compute squared Euclidean distance
    M_trans = torch.norm(v1_t_norm - v2_t_norm)**2
    M_rot = torch.norm(v1_r_norm - v2_r_norm)**2
    
    return M_trans + M_rot

def train_network(V, I_f, theta, theta_t, network, phi, lr_phi):
    """
    Single training step to update network parameters phi.
    """
    # 1. Compute Geodesic Gradient
    grad_G = compute_geodesic_gradient(theta, theta_t)
    
    # 2. Compute Network Gradient and Loss
    grad_N, L_N = compute_network_gradient(theta, V, I_f, network, phi)
    
    # 3. Compute Directional Distance
    M_dist = compute_directional_distance(grad_N, grad_G)
    
    # 4. Double Backward Pass to get dM_dist/d_phi
    # Note: grad_N depends on phi, so we need gradients of grad_N w.r.t phi
    # This requires retaining graph for grad_N
    M_dist.backward()
    
    # 5. Update phi
    with torch.no_grad():
        for param in network.parameters():
            if param.grad is not None:
                param -= lr_phi * param.grad
                param.grad.zero_()
                
    return M_dist, L_N

def register(V, I_f, theta_init, network, lr_theta, lr_ncc, max_iter, ncc_switch_iter):
    """
    Application phase: Optimize theta using trained network, then switch to Grad-NCC.
    """
    theta = theta_init.clone().requires_grad_(True)
    
    for i in range(max_iter):
        # Forward pass through network to get L_N
        e_m, e_f = network(V, theta, I_f, network.parameters())
        L_N = torch.mean((e_m - e_f)**2)
        
        # Backward pass to get dL_N/d_theta
        L_N.backward()
        grad_N = theta.grad.clone()
        
        # Update theta
        with torch.no_grad():
            theta -= lr_theta * grad_N
            theta.grad.zero_()
            
        # Check convergence using STD of L_N (simplified check)
        if i > ncc_switch_iter:
            # Switch to Grad-NCC optimization
            # Implement Grad-NCC loss L_GNCC
            # L_GNCC = ... (PyTorch implementation of Grad-NCC [11])
            # L_GNCC.backward()
            # theta -= lr_ncc * theta.grad
            pass
            
    return theta
```

**Parameters:**
*   **Learning Rate $\phi$:** Not given in the sources. [G] Typical values for deep learning optimization (e.g., 0.001).
*   **Learning Rate $\theta$:** Not given in the sources. [G] Typical values for optimization (e.g., 0.01).
*   **Target Pose $\theta_t$:** Used for geodesic gradient calculation. Not given in the sources. [G] Typically a random perturbation or ground truth during training.
*   **NCC Switch Iteration:** Not given in the sources. [G] Determined by monitoring STD of $L_N$.

**Edge Cases and Pitfalls:**
*   **Zero Gradients:** If $\|v_1^t\|$ or $\|v_1^r\|$ is zero, normalization fails. [G] Add epsilon to denominator.
*   **Precision:** Rodrigues angle-axis representation must be consistent between geodesic gradient calculation and network gradient decomposition [S1].
*   **Convergence Detection:** Monitoring STD of $L_N$ is used to switch from network-based optimization to Grad-NCC [S1].

## Subtopics

### Geodesic Loss ($L_G$)
*   **What it is:** The square of the geodesic distance in SE(3), used as a target for convexity in registration [S1].
*   **Differentiation:** Provides a convex similarity metric with a large capture range [S1].
*   **When to choose:** When initialization is far from ground truth and traditional methods fail [S1].
*   **Link:** [Geodesic Loss](https://arxiv.org/abs/2003.10987v1)

### Grad-NCC Loss ($L_{GNCC}$)
*   **What it is:** Gradient-based Normalized Cross Correlation, implemented as a PyTorch loss function [S1].
*   **Differentiation:** More sensitive to local textures than the network-based metric [S1].
*   **When to choose:** For fine-tuning after the network-based optimization has converged [S1].
*   **Link:** [Grad-NCC](https://arxiv.org/abs/2003.10987v1)

## Operating characteristics
*   **Time Complexity:** Training requires double backward passes, increasing computational cost per iteration [S1]. Application phase is similar to standard back-propagation through the network [S1].
*   **Memory Complexity:** Requires storing intermediate gradients for the double backward pass [S1].
*   **Measured Performance:** Tested on 1536x1536 images (downsampled to 128x128) with isotropic pixel spacing of 0.194 mm/pixel and source-to-detector distance of 1020 mm [S1]. Trained on 17 full body CT scans [S1].
*   **Scaling:** Not given in the sources.
*   **Strengths:** Extends capture range of initialization, less sensitive to local textures than traditional methods [S1].
*   **Limitations:** Requires training phase with geodesic gradient calculation [S1].
*   **When to use:** 2D/3D registration with large initial pose errors [S1].
*   **When not:** When computational resources for training are unavailable [G].

## Background
*   **Origin:** Proposed in "Generalizing Spatial Transformers to Projective Geometry with Applications to 2D/3D Registration" [S1].
*   **History:** Builds on geodesic loss for registration [13, 9] and geodesic gradient implementation [10] [S1].
*   **Usage:** 2D/3D registration in medical imaging (CT to DRR) [S1].

## Sources
[S1] Generalizing Spatial Transformers to Projective Geometry with Applications to 2D/3D Registration, https://arxiv.org/abs/2003.10987v1, fetched 2026-10-26
