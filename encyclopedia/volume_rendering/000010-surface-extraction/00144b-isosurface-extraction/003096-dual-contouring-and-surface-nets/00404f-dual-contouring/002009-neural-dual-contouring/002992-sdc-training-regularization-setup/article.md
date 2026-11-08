# SDC Training & Regularization Setup

Also called: Curriculum DeepSDF Training, SDC Training Pipeline

## Overview
This method defines the self-supervised training pipeline for Self-Supervised Dual Contouring (SDC), utilizing Curriculum DeepSDF as the Deep Implicit Network (DIN) [S1]. It solves the problem of reconstructing implicit surfaces from rendered images of known 3D shapes by jointly training an encoder and an implicit field [S1]. It is distinguished from standard DIN training by the late-stage application of a mesh-based regularization term that aligns the predicted Signed Distance Field (SDF) with the geometry extracted via SDC, rather than relying solely on external SDF supervision [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Rendered images of shapes from the ShapeNet dataset [S1].
*   **Output:** A trained DIN (DeepSDF) and a ResNet-18 encoder that map image features and query points to SDF values [S1].

**Data Structures:**
*   **Query Points:** A set of 400,000 points sampled on a regular grid, specifically chosen to be close to the surface [S1].
*   **Latent Vector:** An encoding of the input image produced by the ResNet-18 encoder [S1].
*   **Mesh Grid:** A regular $64^3$ grid used for SDC mesh construction during regularization [S1].

**Parameters:**
*   **Total Epochs:** 2000 [S1].
*   **Regularization Start Epoch:** 1800 (last 200 epochs) [S1].
*   **Regularization Weight ($\epsilon$):** 10 [S1].
*   **Optimizer:** ADAM [S1].
*   **Hardware:** Ampere-A100 GPU [S1].

**Steps:**
1.  **Encoder Training:** Train a ResNet-18 network to encode the rendering of each shape into a latent vector [S1].
2.  **Implicit Network Training:** Jointly train the ResNet-18 and DeepSDF networks over 2000 epochs using the ADAM optimizer [S1].
3.  **Query Sampling:** For each training step, sample 400,000 query points close to the surface [S1].
4.  **Forward Pass:** Concatenate the latent vector with the query point coordinates and compute the SDF prediction [S1].
5.  **Regularization Application:** For the final 200 epochs of the 2000-epoch training run, apply the SDC regularization term [S1].
6.  **Mesh Construction:** Construct the mesh using SDC on a regular $64^3$ grid [S1].
7.  **Regularization Sampling:** Sample points on a regular grid closer to the extracted surface [S1].
8.  **Loss Combination:** Compute the regularization loss with weight $\epsilon = 10$ and add it to the training objective [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Self-contained training loop for SDC
encoder = ResNet18()
din = DeepSDF()
optimizer = Adam(list(encoder.parameters()) + list(din.parameters()))
total_epochs = 2000
reg_start_epoch = 1800
epsilon = 10
grid_resolution = 64

for epoch in range(1, total_epochs + 1):
    # 1. Encode input rendering into latent vector
    image = next_training_image_iterator()  # Source does not specify dataset loader details
    latent = encoder(image)

    # 2. Sample 400,000 query points close to the surface
    query_points = sample_points_near_surface(count=400000)  # Source does not specify exact sampling distribution

    # 3. Concatenate latent vector with query points and compute SDF prediction
    inputs = concatenate(latent, query_points)
    sdf_predictions = din(inputs)

    # 4. Compute base DIN loss against target values
    base_loss = compute_din_loss(sdf_predictions, query_points)  # Source does not specify exact loss function

    # 5. Apply regularization for the last 200 epochs
    if epoch > reg_start_epoch:
        # Construct mesh using SDC on a 64^3 grid
        mesh = extract_mesh_sdc(sdf_predictions, grid_resolution=64)

        # Sample points on a regular grid closer to the extracted surface
        reg_points = sample_points_near_mesh(mesh)  # Source does not specify exact sampling strategy

        # Compute SDF at regularization points
        reg_inputs = concatenate(latent, reg_points)
        reg_sdf = din(reg_inputs)

        # Compute distance to the extracted mesh for these points
        distances_to_mesh = compute_distance_to_mesh(reg_points, mesh)  # Standard geometry operation

        # Compute regularization loss and combine with base loss
        reg_loss = compute_discrepancy(reg_sdf, distances_to_mesh)
        total_loss = base_loss + epsilon * reg_loss
    else:
        total_loss = base_loss

    # 6. Backpropagate and update weights
    optimizer.zero_grad()
    total_loss.backward()
    optimizer.step()
```

**Edge Cases and Pitfalls:**
*   **Precision:** The regularization relies on accurate mesh extraction from the $64^3$ grid; insufficient resolution may lead to noisy regularization signals [S1].
*   **Timing:** Regularization is applied only in the last 200 epochs to allow the curriculum learning strategy to first establish the general SDF structure [S1].

## Subtopics

### [Deep Implicit Network (DIN) Regularizer](4781-deep-implicit-network-din-regularizer.md)
The specific mesh-based objective used during the final training phase to ensure geometric consistency with the SDC-extracted mesh.

## Operating characteristics
*   **Time Complexity:** Training takes approximately 10 hours per shape on an Ampere-A100 GPU [S1].
*   **Memory Complexity:** Not given in the sources.
*   **Performance:** SDC and NDC have similar inference times, while UNDC requires twice the time per shape [S1].
*   **Strengths:** Avoids test-time optimization; constructs meshes using a regular $64^3$ grid [S1].
*   **Limitations:** Requires joint training of ResNet-18 and DeepSDF; training duration is approximately 10 hours per shape [S1].
*   **When to use:** Reconstructing implicit surfaces from rendered images of known 3D shapes [S1].

## Background
This method originates from the Self-Supervised Dual Contouring paper [S1]. It builds on Curriculum DeepSDF [56] and uses the ShapeNet dataset [9] for training. The approach was developed to enable self-supervised learning of implicit surfaces, leveraging the SDC method for mesh extraction [S1].

## Sources
[S1] Self-Supervised Dual Contouring, https://doi.org/10.1109/cvpr52733.2024.00448, fetched 2026-10-26
