# VDN-NeRF

Also called: View-Dependence Normalization, View-Dependence Normalization NeRF, Ours

## Overview
VDN-NeRF is a method for training neural radiance fields (NeRFs) to achieve high-quality geometry under non-Lambertian surface and dynamic lighting conditions [S1]. It resolves the shape-radiance ambiguity by normalizing view-dependent radiance variations through self-distillation of invariant features encoded within the NeRF itself [S1]. Unlike methods that explicitly model complex underlying factors of view-dependence (such as specularities or dynamic illumination), VDN-NeRF aligns the view-dependence of different scenes to a uniform level, allowing a single optimal network capacity to ensure good view synthesis and prevent geometric degradation [S1].

## How it works
VDN-NeRF builds on [Neural Radiance Fields](4972-neural-radiance-fields.md) and [Surface-SDF Representation](10778-surface-sdf-representation.md) by introducing a parallel feature stream and a distillation network to normalize view-dependent color variations [S1].

### Inputs and Outputs
*   **Inputs:** A set of posed 2D images $I_k$ of a scene.
*   **Outputs:** A trained SDF network $f$, a radiance function $c$, a feature function $F$, and a distillation network $\psi$. The system produces rendered images and depth maps for novel views.

### Data Structures and Network Architectures
1.  **SDF Network $f$:** Predicts the Signed Distance Function $f(x)$ for a point $x \in \mathbb{R}^3$. The zero-level set $S = \{x \mid f(x) = 0\}$ represents the scene geometry [S1].
2.  **Radiance Function $c$:** A 4-layer Multi-Layer Perceptron (MLP) with a hidden dimension of 256. It maps coordinate $x$ and viewing direction $v$ to an RGB value: $c: \mathbb{R}^3 \times S^2 \rightarrow \mathbb{R}^3$ [S1].
3.  **Feature Function $F$:** A 4-layer MLP with a hidden dimension of 256, identical in architecture to $c$. It maps $(x, v)$ to a feature vector [S1].
4.  **Distillation Network $\psi$:** A monocular depth prediction network. It uses a DenseNet161 backbone (specifically WaveletMonodepth) and takes a rendered image $I_{kr}$ as input to predict rendered depth $\hat{d}_{rk}$ and intermediate feature maps [S1].

### Volume Rendering Formulas
The density $\rho(t)$ at a point $p(t)$ along a ray parameterized by $t$ is derived from the SDF $f$ and a scaling factor $\Phi_s$:
$$
\rho(t) = \max\left(\frac{s}{\Phi_s(f(p(t)))} \frac{-d}{dt}(f(p(t))), 0\right)
$$
where $s$ is a scaling factor [S1].

The accumulated transmittance $T(t)$ is:
$$
T(t) = \exp\left(-\int_0^t \rho(\tau) d\tau\right)
$$

The weight function $w(t)$ for volume rendering is:
$$
w(t) = T(t)\rho(t)
$$

The standard color rendering $C$ for a ray $(o, v)$ is:
$$
C(o, v) = \int_0^{+\infty} w(t) c(p(t), v) dt
$$

The feature rendering $\Psi(o, v)$ is defined similarly using the feature function $F$:
$$
\Psi(o, v) = \int_0^{+\infty} w(t) F(p(t), v) dt
$$

### Training Steps
1.  **Instantiate Networks:** Initialize $f$, $c$, $F$, and $\psi$. Pretrain $\psi$ on in-domain data to ensure efficient convergence [S1].
2.  **Sample Points:** For each image $I_k$ in a batch of size $n$, sample points $p_i$ along rays using hierarchical sampling (as in NeuS) [S1].
3.  **Compute Features and Radiance:** For each sample point $p_i$ and viewing direction $v_i$, compute the SDF value $f(p_i)$, density $\rho_i$, and the outputs of the radiance function $c(p_i, v_i)$ and feature function $F(p_i, v_i)$ [S1].
4.  **Volume Render:** Compute the rendered color and the rendered feature $\Psi(o_i, v_i)$ for each ray [S1].
5.  **Distillation Prediction:** Pass the rendered image $I_{kr}$ (or the input image $I_k$) through the distillation network $\psi$ to obtain the normalized feature map $\bar{\psi}_l(I_k)$ from the $l$-th layer. The operational choice sets $l=1$, using features from the first Conv block of the distillation encoder [S1].
6.  **Compute Losses:** Calculate the joint training objective $\mathcal{L}$:
    $$
    \mathcal{L} = \lambda_{\text{color}} \mathcal{L}_{\text{color}} + \lambda_{\text{vdn}} \mathcal{L}_{\text{vdn}} + \lambda_{\text{reg}} \mathcal{L}_{\text{reg}}
    $$
    *   **Color Loss ($\mathcal{L}_{\text{color}}$):** L1 loss between rendered color and input image $I_k$. If masks $M_k$ are available, the loss is weighted by the mask:
        $$
        \mathcal{L}_{\text{color}} = \frac{1}{n} \sum_{i=1}^n \| C(o_i, v_i) - I_{k,i} \|_1 \cdot M_{k,i}
        $$
        Otherwise, use the standard L1 loss:
        $$
        \mathcal{L}_{\text{color}} = \frac{1}{n} \sum_{i=1}^n \| C(o_i, v_i) - I_{k,i} \|_1
        $$
    *   **View-Dependence Normalization Loss ($\mathcal{L}_{\text{vdn}}$):** L1 loss between the rendered feature $\Psi(o_i, v_i)$ and the normalized distillation feature $\bar{\psi}_l(I_k)[p_i]$:
        $$
        \mathcal{L}_{\text{vdn}} = \frac{1}{n} \sum_{i=1}^n \| \Psi(o_i, v_i) - \bar{\psi}_l(I_k)[p_i] \|_1
        $$
    *   **Regularization Loss ($\mathcal{L}_{\text{reg}}$):** Eikonal loss regularizing the gradient of the SDF network to be unit length:
        $$
        \mathcal{L}_{\text{reg}} = \frac{1}{mn} \sum_{i,j} (\|\nabla f(p_{i,j})\| - 1)^2
        $$
        where $m$ is the number of point samples per ray [S1].
7.  **Update:** Minimize $\mathcal{L}$ using gradient descent. Optionally, if the discrepancy between $\psi$'s prediction and self-supervision is small, $\psi$ can be frozen to reduce training time [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def train_vdn_nerf(images, poses, batch_size=512, num_steps=300000):
    # Initialize networks
    f = SDFNetwork()          # Predicts f(x)
    c = RadianceNetwork()     # 4-layer MLP, dim 256
    F = FeatureNetwork()      # 4-layer MLP, dim 256
    psi = DistillationNetwork() # WaveletMonodepth with DenseNet161
    
    # Pretrain psi on in-domain data
    pretrain_distillation_network(psi, in_domain_data)
    
    optimizer = Optimizer([f.parameters(), c.parameters(), F.parameters(), psi.parameters()])
    
    for step in range(num_steps):
        # Sample batch of images
        batch_images, batch_poses = sample_batch(images, poses, batch_size)
        
        total_loss = 0
        for img, pose in zip(batch_images, batch_poses):
            # 1. Sample points along rays using hierarchical sampling
            points, directions, t_vals = sample_rays(pose, hierarchical=True)
            
            # 2. Compute SDF and Density
            sdf_vals = f(points)
            # Compute density rho(t) from SDF gradient and scaling factor Phi_s
            # rho(t) = max( (s / Phi_s(f(p))) * (-d/dt f(p)), 0 )
            densities = compute_density_from_sdf(sdf_vals, points, t_vals)
            
            # 3. Compute Transmittance and Weights
            transmittance, weights = compute_transmittance_and_weights(densities)
            
            # 4. Compute Radiance and Feature Outputs
            radiance_outputs = c(points, directions)
            feature_outputs = F(points, directions)
            
            # 5. Volume Render Color and Feature
            rendered_color = sum(weights * radiance_outputs)
            rendered_feature = sum(weights * feature_outputs)
            
            # 6. Distillation Network Prediction
            # psi takes the rendered image or input image to predict depth features
            # We use the input image I_k for distillation feature extraction
            distillation_features = psi(img)
            # Extract normalized feature from layer l=1 (first Conv block)
            normalized_feature = distillation_features[layer_l=1]
            
            # 7. Compute Losses
            # Color Loss (L1)
            color_loss = mean_l1(rendered_color, img)
            
            # VDN Loss (L1 between rendered feature and normalized distillation feature)
            # Note: normalized_feature must be aligned with point samples p_i
            vdn_loss = mean_l1(rendered_feature, normalized_feature)
            
            # Eikonal Regularization
            # Sample additional points for gradient regularization
            reg_points = sample_random_points()
            grad_f = compute_gradient(f, reg_points)
            eikonal_loss = mean_squared_error(norm(grad_f) - 1.0)
            
            # Joint Loss
            loss = lambda_color * color_loss + lambda_vdn * vdn_loss + lambda_reg * eikonal_loss
            
            # 8. Backpropagate
            optimizer.zero_grad()
            loss.backward()
            optimizer.step()
            
            total_loss += loss.item()
            
        if step % 1000 == 0:
            print(f"Step {step}, Loss: {total_loss}")
            
        # Optional: Freeze psi if discrepancy is small
        if check_distillation_discrepancy(psi) < threshold:
            psi.requires_grad = False
```

### Parameters and Edge Cases
*   **Hyperparameters:** $\lambda_{\text{color}}$, $\lambda_{\text{vdn}}$, $\lambda_{\text{reg}}$ are weighting factors (values not explicitly given in sources, but standard practice applies). Batch size is 512 [S1].
*   **Layer Selection:** The operational choice for the distillation layer index $l$ is 1 (first Conv block of the encoder) [S1].
*   **Pitfall - Distillation Update:** If the distillation network's prediction discrepancy is small, it can be frozen to save time. However, if it is not frozen, it must be updated jointly [S1].
*   **Pitfall - Mask Handling:** If foreground masks $M_k$ are available, the color loss is masked. If not, the standard L1 loss is used. The source notes that masks are defined as $M_k$ for input and $\hat{M}_k = \sum w_{k,j}$ for rendered [S1].
*   **Precision:** The method relies on the stability of the depth prediction network $\psi$. Pretraining $\psi$ is critical for convergence [S1].

## Operating characteristics
*   **Time Complexity:** Training takes approximately 300,000 steps, converging in about 11 hours on a single NVIDIA RTX 3090 GPU [S1].
*   **Memory Complexity:** Uses 4-layer MLPs with hidden dimension 256 for $c$ and $F$, plus a DenseNet161 backbone for $\psi$. Memory usage is comparable to standard SDF-based NeRFs (like NeuS) with the added overhead of the distillation network features [S1].
*   **Performance:** On the BlendedMVS dataset, VDN-NeRF achieves an IoU of 0.854, L1 Chamfer Distance of 0.397, L2 Chamfer Distance of 0.827, Normal Consistency of 0.845, and f-score of 0.834 [S1].
*   **Strengths:** Improves geometry quality under non-Lambertian surfaces and dynamic lighting without changing the volume rendering pipeline. It aligns the optimal network capacity across scenes, preventing shape-radiance ambiguity from degrading geometry [S1].
*   **Limitations:** Requires pretraining the distillation network. The method assumes that depth-prediction features are sufficiently view-invariant, which may not hold for all extreme view-dependent phenomena [S1].
*   **When to Use:** When high-quality geometry is required for scenes with specularities, dynamic lighting, or non-Lambertian materials, and when using a baseline NeRF/SDF method [S1].

## Background
VDN-NeRF was proposed by Bingfan Zhu, Yanchao Yang, Xulong Wang, Youyi Zheng, and Leonidas Guibas from Zhejiang University, The University of Hong Kong, and Stanford University [S1]. It was published in CVPR 2023 (arXiv:2303.17968v1, March 31, 2023) [S1]. The work addresses the fundamental issue in NeRF reconstruction where geometry is often poor due to shape-radiance ambiguity, particularly under varying lighting and non-Lambertian surfaces [S1]. It builds upon the NeuS method [S1].

## Sources
[S1] VDN-NeRF: Resolving Shape-Radiance Ambiguity via View-Dependence Normalization, https://doi.org/10.1109/CVPR52729.2023.00012, fetched 2026-10-29
