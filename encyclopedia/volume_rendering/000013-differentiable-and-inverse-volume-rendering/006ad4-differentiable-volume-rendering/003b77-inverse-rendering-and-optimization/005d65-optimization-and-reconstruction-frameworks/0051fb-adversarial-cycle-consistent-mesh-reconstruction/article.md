# Adversarial Cycle-Consistent Mesh Reconstruction

Also called: Adversarial Cycle-Consistent Mesh Reconstruction Loss, Pix2Vex Training Loss and Optimization

## Overview
Adversarial Cycle-Consistent Mesh Reconstruction is an optimization framework for single-view image-to-geometry reconstruction that jointly trains networks for image-to-mesh ($p2v$), image-to-volume ($a2b$), mesh-to-image ($b2a$), and a discriminator ($D$) [S1, unofficial]. It distinguishes itself from standard differentiable rendering approaches by incorporating cycle consistency between the mesh and volume domains and explicit mesh regularization to ensure geometric validity [S1, unofficial].

## How it works
**Inputs and Outputs:**
The method takes a single input image $a \in A$ and outputs a reconstructed mesh $b^{00} \in B$ and an intermediate volume representation $b^0 \in B$ [S1, unofficial].

**Data Structures and Network Architectures:**
The implementation relies on specific network topologies defined by convolutional blocks $C_k$ and fully connected layers [S1, unofficial].
*   **Convolution-LeakyReLU Layer ($C_k$):** A convolutional layer with $k$ filters of size $4 \times 4$ and stride 2, followed by a LeakyReLU activation with a negative slope of 0.2 [S1, unofficial]. From the fifth convolutional layer onwards, a dropout rate of 50% is applied [S1, unofficial].
*   **Discriminator ($D$):** Architecture: $C_{64}-C_{128}-C_{256}-C_{512}-C_{1}$ followed by a sigmoid function [S1, unofficial].
*   **Image-to-Image Translation ($a2b, b2a$):** These use Pix2Pix networks with the first two or more residual layers, followed by a sigmoid function [S1, unofficial].
*   **Image-to-Mesh ($p2v$ / Pix2Vex):** Based on the Pix2Pix network [S1, unofficial]. The first half consists of symmetric residual network blocks: $C_{64}-C_{128}-C_{256}-C_{512}-C_{512}-C_{512}-C_{512}$ [S1, unofficial]. This is followed by two fully connected layers ($n^2/2 \rightarrow n^2/2 + 3 \cdot v$ and $n^2/2 + 3 \cdot v \rightarrow 3 \cdot v$) and a sigmoid function [S1, unofficial]. Here, $n$ is the image resolution dimension and $v$ is the number of vertices [S1, unofficial].

**Variables and Intermediate States:**
Given an input image $a \in A$:
*   $b^0 = a2b(a)$: The volume representation predicted directly from the image [S1, unofficial].
*   $b^{00} = SR \circ p2v(a)$: The mesh predicted by $p2v$ and rendered by the Soft Rasterizer ($SR$) [S1, unofficial].
*   $a^0 = b2a(b^0)$: The image reconstructed from the volume representation $b^0$ [S1, unofficial].
*   $a^{00} = b2a(b^{00})$: The image reconstructed from the mesh representation $b^{00}$ [S1, unofficial].

**Loss Functions:**
The total loss is defined as:
$$ L = \min_{p2v} \min_{a2b} \min_{b2a} \max_{D} \sum_{i=1}^{5} (\alpha_i \cdot L_i) + L_{reg} $$
where $\alpha_i \in [0, 1]$ are weights, $L_i$ are the five specific loss components, and $L_{reg}$ is the regularization loss [S1, unofficial].

The five loss components are:
$$ L_1 = E_{a\sim A} [\log D(a, b^{00})] + E_{a\sim A} [\log(1 - D(a, b^0))] + E_{a\sim A} [\|b^{00} - b^0\|_1] $$
$$ L_2 = E_{a\sim A} [\log D(a, b^{00})] + E_{a\sim A} [\log(1 - D(a^{00}, b^{00}))] + E_{a\sim A} [\|a^{00} - a\|_1] $$
$$ L_3 = E_{a\sim A} [\log D(a, b^0)] + E_{a\sim A} [\log(1 - D(a^{00}, b^0))] + E_{a\sim A} [\|a^0 - a\|_1] + E_{a\sim A} [\|b^{00} - b^0\|_1] $$
$$ L_4 = E_{a\sim A} [\log D(a, b^{00})] + E_{a\sim A} [\log(1 - D(a^0, b^{00}))] + E_{a\sim A} [\|a^0 - a\|_1] + E_{a\sim A} [\|b^{00} - b^0\|_1] $$
$$ L_5 = E_{a\sim A} [\log D(a, b^0)] + E_{a\sim A} [\log(1 - D(a^0, b^0))] + E_{a\sim A} [\|a^0 - a\|_1] $$

**Regularization Loss ($L_{reg}$):**
Applied to the reconstructed mesh to enforce regularity and reduce curvature [S1, unofficial]. It consists of three terms:
1.  **Normal Angle Similarity:** L2 norm of the difference between normals of adjacent faces [S1, unofficial].
2.  **Edge Length Similarity:** L1 norm of the difference between edge lengths [S1, unofficial].
3.  **Vertex Regularity:** L1 norm of the distance from each vertex to the mean position of its adjacent vertices [S1, unofficial].

**Training Procedure:**
The training alternates in the following order:
1.  Train discriminator $D$.
2.  Train translation from $B$ to $A$ ($b2a$).
3.  Train components performing translation from $A$ to $B$ ($p2v+SR$, $a2b$) [S1, unofficial].
For each section, the five losses $L_i$ are trained separately using the Adam optimizer with an initial learning rate of $10^{-6}$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.optim as optim

# Networks are assumed defined: p2v, a2b, b2a, D
# SR is the Soft Rasterizer

NUM_EPOCHS = 1000
BATCH_LOADER = iter([])  # Iterator yielding batches of input images a
n = 128  # Image resolution dimension
v = 162  # Number of vertices
lr = 1e-6
alpha_weights = [1.0, 1.0, 1.0, 1.0, 1.0]  # Weights alpha_i in [0, 1]

optimizer_p2v = optim.Adam(p2v.parameters(), lr=lr)
optimizer_a2b = optim.Adam(a2b.parameters(), lr=lr)
optimizer_b2a = optim.Adam(b2a.parameters(), lr=lr)
optimizer_D = optim.Adam(D.parameters(), lr=lr)

def compute_regularization_loss(mesh_vertices, mesh_faces):
    # 1. Normal angle similarity (L2 norm)
    # 2. Edge length similarity (L1 norm)
    # 3. Distance to mean vertex of adjacent vertices (L1 norm)
    # Standard mesh geometry operations [G]
    pass

def compute_loss_L1(a, b0, b00):
    return torch.log(D(a, b00) + 1e-8).mean() + torch.log(1 - D(a, b0) + 1e-8).mean() + torch.abs(b00 - b0).mean()

def compute_loss_L2(a, a00, b00):
    return torch.log(D(a, b00) + 1e-8).mean() + torch.log(1 - D(a00, b00) + 1e-8).mean() + torch.abs(a00 - a).mean()

def compute_loss_L3(a, a00, b0, b00, a0):
    return torch.log(D(a, b0) + 1e-8).mean() + torch.log(1 - D(a00, b0) + 1e-8).mean() + torch.abs(a0 - a).mean() + torch.abs(b00 - b0).mean()

def compute_loss_L4(a, a0, b00, b0):
    return torch.log(D(a, b00) + 1e-8).mean() + torch.log(1 - D(a0, b00) + 1e-8).mean() + torch.abs(a0 - a).mean() + torch.abs(b00 - b0).mean()

def compute_loss_L5(a, a0, b0):
    return torch.log(D(a, b0) + 1e-8).mean() + torch.log(1 - D(a0, b0) + 1e-8).mean() + torch.abs(a0 - a).mean()

def compute_all_losses(a, b0, b00, a0, a00):
    return [
        compute_loss_L1(a, b0, b00),
        compute_loss_L2(a, a00, b00),
        compute_loss_L3(a, a00, b0, b00, a0),
        compute_loss_L4(a, a0, b00, b0),
        compute_loss_L5(a, a0, b0)
    ]

for epoch in range(NUM_EPOCHS):
    for a in BATCH_LOADER:
        b0 = a2b(a)
        mesh = p2v(a)
        b00 = mesh
        a0 = b2a(b0)
        a00 = b2a(b00)

        losses = compute_all_losses(a, b0, b00, a0, a00)
        reg_loss = compute_regularization_loss(mesh.vertices, mesh.faces)

        # 1. Train Discriminator D
        optimizer_D.zero_grad()
        loss_D = sum(alpha_weights[i] * losses[i] for i in range(5)) + reg_loss
        loss_D.backward()
        optimizer_D.step()

        # 2. Train translation from B to A (b2a)
        optimizer_b2a.zero_grad()
        loss_b2a = sum(alpha_weights[i] * losses[i] for i in range(5)) + reg_loss
        loss_b2a.backward()
        optimizer_b2a.step()

        # 3. Train components performing translation from A to B (p2v+SR, a2b)
        optimizer_p2v.zero_grad()
        optimizer_a2b.zero_grad()
        loss_p2v_a2b = sum(alpha_weights[i] * losses[i] for i in range(5)) + reg_loss
        loss_p2v_a2b.backward()
        optimizer_p2v.step()
        optimizer_a2b.step()
```

**Stabilization of Smooth Z-Buffer:**
The smooth z-buffer used in the Soft Rasterizer can cause pixels not covered by any triangle to appear as if they have visible triangles because low weights close to zero can cause high values in the weighted SoftMin [S1, unofficial]. To stabilize this, two triangles covering the background are added and the visibility value is reduced with increasing distance from triangles [S1, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
Not given in the sources.

**Measured Performance:**
Experiments were conducted with an image resolution of $n \times n = 128 \times 128$ and a number of vertices $v = 162$ [S1, unofficial].

**Strengths and Limitations:**
The method combines adversarial training for realism, cycle consistency for domain alignment, and mesh regularization for geometric validity [S1, unofficial]. It requires stabilization for uncovered pixels in the smooth z-buffer and relies on a fixed vertex count [S1, unofficial].

**When to Use:**
Use when single-view image-to-geometry reconstruction is required and the output needs to be a valid, regular mesh rather than just a volume or point cloud [S1, unofficial].

## Background
The method was introduced in the paper "Pix2Vex: Image-to-Geometry Reconstruction using a Smooth Differentiable Renderer" [S1, unofficial]. It builds upon differentiable rendering techniques and adversarial training frameworks [S1, unofficial]. The smooth z-buffer stabilization technique relates to work on blurred depth tests for non-photorealistic rendering [S1, unofficial].

## Sources
[S1] Pix2Vex: Image-to-Geometry Reconstruction using a Smooth Differentiable Renderer, https://doi.org/10.48550/arxiv.1903.11149, fetched 2026-10-26 (unofficial)
