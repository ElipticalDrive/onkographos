# Adversarial Training (GAN)

Also called: Discriminator Training, GAN Loss, Joint Adversarial-Rasterization Training, LSGAN Adversarial Training

## Overview
Adversarial Training integrates a discriminator network into the optimization loop of a generative volume rendering model to distinguish between real ground-truth images and synthesized renders, using the discriminator's feedback to refine the generator's output [S1, S2]. Unlike standard supervised training that relies solely on pixel-wise reconstruction losses, this method enforces perceptual realism and reduces artifacts by learning the distribution of real data, making it distinct from sibling methods like [StyleNeRF](8659-stylenerf.md) which focus on style transfer, or [HyperNeRFGAN](8765-hypernerfgan.md) which may use different generative priors [S2].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Real ground-truth images $I_{gt}$, generated/fake images $G(z)$ or $R_\theta(\cdot)$, camera parameters $\theta$, and latent codes $z$ [S1, S2, S3].
*   **Outputs:** Updated network parameters for the generator (synthesizer) $\theta$ and discriminator $\phi$ [S1, S2].

**Data Structures:**
*   **Generator/Synthesizer ($G$ or $R_\theta$):** A neural network that maps inputs to rendered images [S1, S2].
*   **Discriminator ($D$ or $D_\phi$):** A neural network that outputs a scalar score indicating the likelihood of an image being real, optionally equipped with multi-scale branches [S2] or a pose estimation branch [S3].

**Numbered Steps:**
1.  **Initialization:** Initialize generator parameters $\theta$ and discriminator parameters $\phi$ [G]. If using multi-scale discrimination, initialize $N_s$ branches [S2].
2.  **Forward Pass (Generator):** Sample latent codes $z$ and camera parameters $\theta$. Compute the generated image $I_{fake} = G(z, \theta)$ (or $R_\theta(S_t^{(i)})$) [S2, S3].
3.  **Discriminator Update:** Compute discriminator scores for real images $D(I_{gt})$ and fake images $D(I_{fake})$ [S1, S2]. Calculate Discriminator Loss $L_{GAN,D}$ based on the specific GAN variant (e.g., LSGAN or non-saturating with R1) [S1, S2, S3]. Update $\phi$ by minimizing $L_{GAN,D}$ [S1, S2].
4.  **Generator Update:** Compute discriminator scores for fake images $D(I_{fake})$ [S1, S2]. Calculate Generator Loss $L_{GAN,G}$ [S1, S2]. If using contrastive pose learning, compute pose embeddings $p_I$ from $I_{fake}$ and $p_\theta$ from $\theta$, then calculate $L_{pose}$ [S3]. If using 3D Gaussian regularization, compute $L_{center}$ and $L_{knn}$ based on anchor Gaussian positions $\mu^0_A$ [S3]. Calculate total Generator Loss $L_G = L_{GAN,G} + \lambda_{pose} L_{pose} + \lambda_{center} L_{center} + \lambda_{knn} L_{knn}$ [S1, S3]. Update $\theta$ by minimizing $L_G$ [S1, S2, S3].
5.  **Iteration:** Repeat steps 2-4 for the desired number of epochs [G].
6.  **Inference:** Remove the discriminator $D_\phi$ to avoid inference overhead; use only $R_\theta$ [S2].

**Formulas:**

*   **LSGAN Adversarial Loss (Synthesizer/Discriminator Joint):**
    $$
    L_{adv}^\phi = \sum_{i=1}^{N_s} \left[ \sum_{t=1}^{T} D_\phi(R_\theta(S_t^{(i)}))^2 + (D_\phi(I_{gt}^{(i)}) - 1)^2 \right]
    $$
    Where $N_s$ is the total number of discriminator scales, $I_{gt}^{(i)}$ is the scale-adjusted ground-truth image, and $S_t^{(i)}$ is the rasterized raw image at resolution level $t$ and scale $i$ [S2].

*   **Non-Saturating Adversarial Loss with R1 Regularization:**
    $$
    L_{adv} = \mathbb{E}_{z \sim P_z, \theta \sim P_\theta} [f(d(g(z, \theta)))] + \mathbb{E}_{I_r \sim P_{I_r}} [f(-d(I_r)) + \lambda \|\nabla d(I_r)\|^2]
    $$
    Where $f(t) = -\log(1 + \exp(-t))$ is the softplus function, $d(\cdot)$ is the discriminator, $g(\cdot)$ is the generator, and $\lambda$ is the R1 regularization strength [S3].

*   **Standard Log-Loss (Alternative Formulation):**
    $$
    L_{GAN,D} = - \log(D(x)) - \log(1 - D(G(z)))
    $$
    $$
    L_{GAN,G} = - \log(D(G(z)))
    $$
    Where $x$ is real data, $G(z)$ is generated data, and $D(\cdot)$ is the discriminator score [S1].

*   **Contrastive Pose Loss:**
    $$
    L_{pose} = -\log \left( \frac{\exp(\text{sim}(p_I, p_\theta^+) / \tau)}{\sum_{b=1}^{B} \exp(\text{sim}(p_I, p_\theta^{(b)}) / \tau)} \right)
    $$
    Where $\text{sim}(\cdot, \cdot)$ is cosine similarity, $\tau$ is temperature, $p_I$ is the pose embedding from the image, $p_\theta^+$ is the positive pose embedding from parameters, and $B$ is batch size [S3].

*   **3D Gaussian Regularization Losses:**
    $$
    L_{center} = \frac{1}{N} \sum_{j=1}^N \|\mu^0_{A,j}\|^2
    $$
    $$
    L_{knn} = \frac{1}{NK} \sum_{j=1}^N \sum_{k=1}^K \|\mu^0_{A,j} - \text{KNN}(\mu^0_{A,j}, k)\|^2
    $$
    Where $\mu^0_{A,j}$ is the position of the $j$-th anchor Gaussian at the coarsest level, and $\text{KNN}(\cdot, k)$ returns the position of the $k$-th nearest neighbor [S3].

*   **Total Loss:**
    $$
    L = L_{adv} + \lambda_{pose} L_{pose} + \lambda_{center} L_{center} + \lambda_{knn} L_{knn}
    $$
    [S3]

**Parameters:**
*   $N_s$: Total number of discriminator scales. Typical value: 5 [S2].
*   $\lambda$: R1 regularization strength [S3].
*   $\tau$: Temperature scaling parameter in contrastive loss [S3].
*   $K$: Number of nearest neighbors for $L_{knn}$ [S3].
*   $\lambda_{pose}, \lambda_{center}, \lambda_{knn}$: Strengths of corresponding objectives [S3].
*   $L$: Number of frequency bands in positional encoding (inherited from parent) [G].
*   $T$: Number of resolution levels in the synthesizer [S2].

**Edge Cases and Pitfalls:**
*   **Inference Overhead:** The discriminator must be removed at test time to avoid adding computational complexity [S2].
*   **Precision:** The softplus function $f(t)$ is used to avoid saturation in the non-saturating loss formulation [S3].
*   **Data Augmentation:** Adaptive data augmentation is recommended for limited datasets (e.g., AFHQ-Cat) to prevent overfitting [S3].
*   **Degenerate Input:** If point cloud information is insufficient, the synthesizer may produce artifacts; adversarial supervision helps generalize to unseen inputs [S2].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def softplus(t):
    return -math.log(1 + math.exp(-t))

def cosine_similarity(a, b):
    return np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b))

def compute_knn_distances(points, k):
    # Returns distances to k nearest neighbors for each point
    n = len(points)
    dists = np.zeros((n, n))
    for i in range(n):
        for j in range(n):
            dists[i, j] = np.linalg.norm(points[i] - points[j])
    knn_dists = np.sort(dists, axis=1)[:, 1:k+1]
    return knn_dists

def train_adversarial_volume_rendering(generator, discriminator, real_images, latent_sampler, pose_encoder, config):
    """
    Trains a volume rendering generator with adversarial supervision.
    
    Args:
        generator: Synthesizer network R_theta.
        discriminator: Discriminator network D_phi.
        real_images: Iterator of ground-truth image batches.
        latent_sampler: Sampler for latent codes z and camera parameters theta.
        pose_encoder: MLP encoding camera parameters theta to pose embeddings.
        config: Dictionary containing hyperparameters.
    """
    optimizer_G = GradientDescent(generator.parameters(), lr=config.lr_G)
    optimizer_D = GradientDescent(discriminator.parameters(), lr=config.lr_D)

    lambda_r1 = config.lambda_r1
    lambda_pose = config.lambda_pose
    lambda_center = config.lambda_center
    lambda_knn = config.lambda_knn
    tau = config.tau
    Ns = config.Ns
    K = config.K

    for epoch in range(config.epochs):
        for real_batch in real_images:
            # 1. Sample latent codes and camera parameters
            z, theta = latent_sampler.sample(batch_size=len(real_batch))

            # 2. Generate fake images
            fake_batch = generator(z, theta)

            # 3. Discriminator Update
            optimizer_D.zero_grad()

            if Ns > 1:
                # LSGAN style loss for multi-scale
                d_real_scales = discriminator.multi_scale(real_batch)
                d_fake_scales = discriminator.multi_scale(fake_batch)
                loss_d = 0
                for i in range(Ns):
                    loss_d += np.sum(d_fake_scales[i]**2) + np.sum((d_real_scales[i] - 1)**2)
            else:
                # Non-saturating with R1 regularization
                d_real = discriminator(real_batch)
                d_fake = discriminator(fake_batch)

                # R1 Regularization: lambda * ||grad D(real)||^2
                grad_real = gradient(d_real, real_batch)
                r1_reg = lambda_r1 * 0.5 * np.sum(grad_real**2)

                loss_d = np.mean(softplus(-d_real)) + np.mean(softplus(d_fake)) + r1_reg

            loss_d.backward()
            optimizer_D.step()

            # 4. Generator Update
            optimizer_G.zero_grad()

            fake_batch = generator(z, theta)
            d_fake = discriminator(fake_batch)

            # Generator Adversarial Loss
            loss_g_adv = np.mean(softplus(d_fake))

            loss_g = loss_g_adv

            # 5. Add Contrastive Pose Loss
            if config.use_pose_contrast:
                p_theta = pose_encoder(theta)
                p_I_fake = discriminator.get_pose_embedding(fake_batch)
                p_I_real = discriminator.get_pose_embedding(real_batch)

                sim_matrix = np.zeros((len(theta), len(theta)))
                for i in range(len(theta)):
                    for j in range(len(theta)):
                        sim_matrix[i, j] = cosine_similarity(p_I_fake[i], p_theta[j]) / tau

                pos_sim = np.diag(sim_matrix)
                neg_sim = sim_matrix - np.eye(len(theta)) * 1e9
                loss_pose = -np.mean(np.log(np.exp(pos_sim) / np.sum(np.exp(neg_sim), axis=1)))

                loss_g += lambda_pose * loss_pose

            # 6. Add 3D Gaussian Regularization
            if config.use_gaussian_reg:
                mu_A = generator.get_coarsest_anchor_positions()
                loss_center = np.mean(np.linalg.norm(mu_A, axis=1)**2)

                knn_dists = compute_knn_distances(mu_A, K)
                loss_knn = np.mean(knn_dists**2)

                loss_g += lambda_center * loss_center + lambda_knn * loss_knn

            loss_g.backward()
            optimizer_G.step()

    # Remove discriminator for inference
    del discriminator
```

## Subtopics

### [Lighting Prediction Network Training](19342-lighting-prediction-network-training.md)
A training strategy for predicting lighting conditions in street scenes that addresses depth rendering ambiguity and encourages photorealism through adversarial supervision. It distinguishes itself from standard differentiable rendering pipelines by combining a multi-term hybrid loss (including sky regression, reconstruction, and regularization) with a PatchGAN discriminator, trained in a two-stage process (pre-training followed by joint adversarial training) to stabilize convergence.

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Requires memory for both generator and discriminator networks during training; the discriminator is removed at test time, so inference memory matches the generator alone [S2].
*   **Performance:** Produces more realistic textures and fewer artifacts than pure L1/L2 loss methods, particularly in high-frequency details [S2]. FID scores are commonly used to evaluate quality [S3].
*   **Strengths:** Improves generalization on unseen inputs and handles insufficient point cloud information better than non-adversarial methods [S2].
*   **Limitations:** Not given in the sources.

## Background
Adversarial training in volume rendering and novel view synthesis was introduced to address the blurriness and artifacts common in supervised learning approaches [S1, S2]. The method draws from Generative Adversarial Networks (GANs) proposed by Goodfellow et al. [S1]. It has been applied to super-resolution of volumetric data [S1], point-based graphics [S2], and 3D Gaussian splatting [S3].

## Sources
[S1] Volumetric Isosurface Rendering with Deep Learning-Based Super ..., https://ieeexplore.ieee.org/abstract/document/8918030, fetched 2026-10-31
[S2] CE-NPBG: Connectivity Enhanced Neural Point-Based Graphics for Novel View Synthesis in Autonomous Driving Scenes, https://doi.org/10.1109/cvprw67362.2025.00240, fetched 2026-10-29
[S3] GSGAN: Adversarial Learning for Hierarchical Generation of 3D Gaussian Splats, https://doi.org/10.52202/079017-2171, fetched 2026-10-29
