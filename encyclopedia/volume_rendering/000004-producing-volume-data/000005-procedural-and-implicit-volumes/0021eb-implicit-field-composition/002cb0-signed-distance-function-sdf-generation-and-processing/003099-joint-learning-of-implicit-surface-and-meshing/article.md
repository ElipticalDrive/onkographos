# Joint Learning of Implicit Surface and Meshing

Also called: End-to-end SDF and Mesh Learning, Joint SDF-Mesh Optimization

## Overview
This method jointly optimizes a Signed Distance Function (SDF) and a reconstructed mesh in an end-to-end pipeline, predicting surface meshes directly from input images without relying on explicit mesh training data or proxy meshes [S1]. It differs from sibling methods like [SDF to Density Conversion](1282-sdf-to-density-conversion.md) by producing a geometric mesh output rather than a volumetric density field, and from [GSurf SDF Learning from Gaussians](20862-gsurf-sdf-learning-from-gaussians.md) by using image-conditioned latent vectors for shape representation rather than Gaussian primitives.

## How it works
**Inputs and Outputs:**
*   **Input:** One or more images of the target object.
*   **Output:** A reconstructed surface mesh $M$.

**Data Structures:**
1.  **Latent Vector ($\lambda$):** A fixed-size vector encoding the shape identity, produced by the encoder.
2.  **Regular Grid ($G$):** A 3D grid of query points $\mathbf{x}$ where the SDF is evaluated.
3.  **SDF Field ($S_G$):** The scalar field values $f_\theta(G)$ predicted by the implicit network.
4.  **Mesh ($M$):** The triangle mesh extracted via the SDC network.

**Architecture Components:**
1.  **Image Encoder:** A ResNet-18 network [S1] that maps input images to the latent vector $\lambda$.
2.  **Implicit Shape Network (DeepSDF):** A network conditioned on $\lambda$ that predicts the SDF value $d(\mathbf{x})$ for any point $\mathbf{x}$ [S1].
3.  **SDC Network:** A network that takes the SDF field $S_G$ as input and outputs the mesh vertices and connectivity [S1].

**Training Schedule:**
1.  **Phase 1 (Independent):** Train the DeepSDF network (with the image encoder) and the SDC network independently.
2.  **Phase 2 (Joint Fine-tuning):** Fine-tune both networks jointly for 200 epochs [S1]. The input is an image, and the output is the mesh $M$.

**Loss Function:**
The joint optimization minimizes the combined loss $\mathcal{L}_{\text{SVR}}$:
$$
\mathcal{L}_{\text{SVR}} = \mathcal{L}_{\text{SDF}} + \alpha_3 \mathcal{L}_{\text{SDR}} + \alpha_4 \mathcal{L}_{\text{D}}
$$
*   $\mathcal{L}_{\text{SDF}}$: Loss for reconstructing the implicit representation (defined in Eq. 8 of [S1]).
*   $\mathcal{L}_{\text{SDR}}$: Loss for regularizing the implicit representation (defined in Eq. 7 of [S1]).
*   $\mathcal{L}_{\text{D}}$: Loss for training the SDC network (defined in Eq. 3 of [S1]).
*   $\alpha_3, \alpha_4$: Weighting hyperparameters.

**Pitfalls:**
*   **Normal Consistency Instability:** The normal consistency loss $\mathcal{L}_N$ is explicitly omitted during joint training. Including it causes instability because the evaluated SDF values may be noisy [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Initialization
encoder = ResNet18()
deep_sdf = DeepSDF() # Conditioned on latent vector
sdc_net = SDCNetwork()

# Phase 1: Independent Training
# (Details of independent training steps are not provided in the source)
# Train encoder + deep_sdf on image-to-SDF task
# Train sdc_net on SDF-to-mesh task

# Phase 2: Joint Fine-tuning
optimizer = Optimizer(params=encoder.parameters() + deep_sdf.parameters() + sdc_net.parameters())

for epoch in range(200):
    for image in training_images:
        # 1. Encode image to latent vector
        lambda_vec = encoder(image)
        
        # 2. Define query grid G
        G = create_regular_grid()
        
        # 3. Predict SDF values on grid
        # DeepSDF takes query points and latent vector
        sdf_values = deep_sdf(G, lambda_vec)
        
        # 4. Reconstruct mesh using SDC
        mesh = sdc_net(sdf_values)
        
        # 5. Compute Loss
        # L_SDF and L_SDR depend on ground truth SDF/shape (Eq 7, 8 in [S1])
        loss_sdf = compute_sdf_loss(sdf_values, ground_truth_sdf)
        loss_sdr = compute_sdf_regularization_loss(sdf_values, ground_truth_sdf)
        
        # L_D depends on ground truth mesh (Eq 3 in [S1])
        loss_d = compute_sdc_loss(mesh, ground_truth_mesh)
        
        # Combined Loss
        loss = loss_sdf + alpha_3 * loss_sdr + alpha_4 * loss_d
        
        # 6. Backpropagate
        optimizer.zero_grad()
        loss.backward()
        optimizer.step()
```

## Subtopics

### [SDF Fusion](22631-sdf-fusion.md)
SDF Fusion fuses multi-view depth and normal data into a Truncated Signed Distance Function (TSDF) volume and a weighted color volume by raycasting from camera to voxel centers, accumulating signed distances and colors via weighted averaging. It is distinguished by its application in joint learning pipelines where SMPL body priors provide rough features and multi-view inputs provide detailed features, which are fused to jointly learn the SDF for 3D reconstruction.

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Strengths:** Eliminates the need for explicit mesh data or proxy meshes during training, allowing for end-to-end optimization of both the implicit field and the mesh geometry [S1].
*   **Limitations:** The method is sensitive to noisy SDF evaluations, necessitating the exclusion of normal consistency losses which can destabilize training [S1].
*   **When to use:** When predicting meshes directly from images is required without access to explicit mesh annotations or proxy generation steps.

## Background
This method was introduced in the paper "Self-Supervised Dual Contouring" [S1]. It builds on the DeepSDF architecture [S1] and the SDC (Self-Supervised Dual Contouring) network [S1]. The approach addresses the limitations of existing data-driven meshing methods that rely on explicit mesh data or axiomatically produced proxy meshes [S1].

## Sources
[S1] Self-Supervised Dual Contouring, https://doi.org/10.1109/cvpr52733.2024.00448, fetched 2026-10-26
