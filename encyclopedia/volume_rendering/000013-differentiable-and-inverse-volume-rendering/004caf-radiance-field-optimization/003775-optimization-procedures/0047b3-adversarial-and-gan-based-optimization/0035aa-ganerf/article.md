# GANeRF

Also called: Adversarial Loss for GANerf, Adversarial NeRF Training, GAN-based NeRF Optimization, GANeRF Adversarial Training Loop, GANeRF

## Overview
GANeRF optimizes a 3D radiance field representation using a 2D adversarial loss, where a discriminator learns the distribution of image patches and provides gradient feedback to the NeRF to enforce 3D-consistent photorealism [S1]. It solves the problem of quality degradation in novel view synthesis caused by imperfect input data, such as limited coverage, image distortion, or illumination changes [S1]. Unlike standard NeRF optimization which relies solely on photometric reconstruction losses, GANeRF distinguishes itself by directly backpropagating gradients from a 2D patch discriminator into the 3D scene representation [S1]. It differs from [GNeRF](7359-gnerf.md) by focusing on patch-based discrimination for local realism, and from [Test-time Adversarial Optimization](19464-test-time-adversarial-optimization.md) by its integration directly into the training loop rather than a post-processing stage [G].

## How it works
**Inputs and Outputs:**
- **Inputs:** A set of posed input images capturing a static 3D scene; a base NeRF architecture (e.g., Nerfacto [S1]).
- **Outputs:** An optimized 3D radiance field representation capable of synthesizing photorealistic novel views.

**Data Structures:**
The implementation relies on the Nerfstudio framework structures [S2, unofficial]:
- `GanerfPipelineConfig`: Extends `VanillaPipelineConfig` to include discriminator components.
- `GanerfModelConfig`: Extends the base model to include discriminator loss computation hooks.
- `GanerfTrainerConfig`: Extends the trainer to handle the adversarial training loop.
- **Discriminator:** A discriminator network that distinguishes between real and fake image patches [S2, unofficial].
- **Patches:** Extracted from rendered RGB images and ground truth images. Patches are reshaped from $(B, H, W, 3)$ to $(B, 3, H, W)$ and potentially unfolded to match the discriminator's resolution [S2, unofficial].

**Parameters:**
- `adv_loss_mult`: Multiplier for the adversarial loss. Default: 0.0003 [S2, unofficial].
- `vgg_loss_mult`: Multiplier for VGG perceptual loss. Default: 0.0003 [S2, unofficial].
- `gp_loss_mult`: Multiplier for gradient penalty loss. Default: 0.2 [S2, unofficial].

**Numbered Steps:**
1. **Initialization:** Set up the NeRF model and the 2D discriminator. Initialize optimizers for both the NeRF parameters $\Theta$ and the discriminator parameters $\eta$ [S2, unofficial].
2. **Iteration Start:** For each training iteration, retrieve a batch of rays and corresponding ground truth images [S2, unofficial].
3. **NeRF Forward Pass:** Render the scene using the current NeRF parameters $\Theta$ to generate predicted RGB images $\hat{I}$ [S1].
4. **Patch Extraction:** Extract patches from the rendered images $\hat{I}$ and the ground truth images $I$ [S2, unofficial]. Reshape patches to $(B, 3, H_p, W_p)$ and normalize if required (e.g., $2.0 \cdot x - 1.0$) [S2, unofficial].
5. **Discriminator Forward Pass (Fake):** Pass the fake patches $\hat{P}$ through the discriminator $D$ to get predictions $pred_{fake} = D(\hat{P}; \eta)$ [S2, unofficial].
6. **Discriminator Forward Pass (Real):** Pass the real patches $P$ through the discriminator $D$ to get predictions $pred_{real} = D(P; \eta)$ [S2, unofficial].
7. **Compute Discriminator Loss:** Calculate the discriminator loss $L_D$ using the real and fake predictions. This includes the standard adversarial loss and a gradient penalty (R1) to stabilize training [S2, unofficial].
   - $L_{adv} = \text{mean}(\text{softplus}(-pred_{fake}))$ [S2, unofficial].
   - Gradient penalty is computed on real patches [S2, unofficial].
8. **Update Discriminator:**
   - Zero gradients for the discriminator optimizer [S2, unofficial].
   - Compute $L_D$ [S2, unofficial].
   - Backpropagate $L_D$ to update $\eta$.
   - Verify gradients are finite [S2, unofficial].
   - Step the discriminator optimizer [S2, unofficial].
9. **Compute NeRF Loss:**
   - Calculate the standard reconstruction loss (e.g., L1) between $\hat{I}$ and $I$ [S2, unofficial].
   - Calculate the adversarial loss for the NeRF: $L_{adv\_nerf} = \text{mean}(\text{softplus}(-D(\hat{P}; \eta)))$ [S2, unofficial].
   - Calculate VGG perceptual loss if enabled [S2, unofficial].
   - Combine losses: $L_{total} = L_{rec} + \lambda_{adv} L_{adv\_nerf} + \lambda_{vgg} L_{vgg}$ [S2, unofficial].
10. **Update NeRF:**
    - Zero gradients for the NeRF optimizer.
    - Backpropagate $L_{total}$ through the discriminator (if using the discriminator's output directly) and into the NeRF parameters $\Theta$ [S1].
    - Step the NeRF optimizer [S2, unofficial].

**Formulas:**
- **Adversarial Loss (for NeRF):**
  $$ L_{adv} = \mathbb{E}_{\hat{P} \sim P_{fake}}[\text{softplus}(-D(\hat{P}; \eta))] $$
  where $\text{softplus}(x) = \log(1 + e^x)$ [S2, unofficial].
- **Gradient Penalty (R1):**
  $$ L_{gp} = \frac{1}{2} \mathbb{E}_{P \sim P_{real}}[\|\nabla_P D(P; \eta)\|_2^2] $$
  where $\nabla_P$ is the gradient of the discriminator output with respect to the input patch [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def softplus(x):
    return torch.log(1 + torch.exp(x))

def train_iteration(model, discriminator, batch, config):
    """
    Performs one training iteration for GANeRF.
    model: The NeRF model (generator).
    discriminator: The 2D patch discriminator.
    batch: Dictionary containing 'rays' and 'gt_images'.
    config: Configuration object with adv_loss_mult, vgg_loss_mult, gp_loss_mult.
    """
    # 1. Render scene
    rays = batch['rays']
    gt_images = batch['gt_images']
    rendered_outputs = model(rays)
    rendered_images = rendered_outputs['rgb'] # Shape: (B, H, W, 3)

    # 2. Extract patches
    # get_patches handles reshaping to (B, 3, Hp, Wp) and normalization
    real_patches, fake_patches = model.get_patches(rendered_outputs, batch)

    # 3. Discriminator Training
    if config.adv_loss_mult > 0.0:
        discriminator.zero_grad()

        pred_real = discriminator(real_patches)
        pred_fake = discriminator(fake_patches)

        # Standard GAN loss for discriminator
        d_loss_real = softplus(-pred_real).mean()
        d_loss_fake = softplus(pred_fake).mean()

        # R1 Gradient Penalty
        real_patches.requires_grad_(True)
        pred_real_gp = discriminator(real_patches)
        gradients = torch.autograd.grad(
            outputs=[pred_real_gp.sum()],
            inputs=[real_patches],
            create_graph=True,
            only_inputs=True
        )[0]
        r1_penalty = gradients.square().sum([1, 2, 3]).mean()
        gp_loss = config.gp_loss_mult * r1_penalty / 2

        d_loss = d_loss_real + d_loss_fake + gp_loss
        d_loss.backward()
        discriminator.optimizer.step()

    # 4. NeRF Training
    model.zero_grad()

    # Reconstruction Loss (L1)
    rec_loss = torch.mean(torch.abs(rendered_images - gt_images))

    # Adversarial Loss for NeRF
    pred_fake_nerf = discriminator(fake_patches)
    adv_loss = softplus(-pred_fake_nerf).mean()

    # VGG perceptual loss computation (not given in sources)
    # vgg_loss = ...

    # Total Loss
    total_loss = rec_loss + config.adv_loss_mult * adv_loss + config.vgg_loss_mult * vgg_loss

    # Backprop and step NeRF
    total_loss.backward()
    model.optimizer.step()

    return total_loss, adv_loss, d_loss
```

**Edge Cases and Pitfalls:**
- **Limited Coverage:** In regions with sparse input observations, the discriminator may overfit to the observed distribution, requiring careful tuning of loss multipliers to maintain 3D consistency [S1].
- **Gradient Verification:** Gradients must be verified as finite after the discriminator update to prevent NaN propagation into the NeRF parameters [S2, unofficial].

## Subtopics

### [Conditional Adversarial Loss](17047-conditional-adversarial-loss.md)
Conditional Adversarial Loss is a regularization term used in differentiable volume rendering optimization to enforce photorealism and suppress blurry or distorted artifacts in rendered novel views. It distinguishes itself from standard reconstruction losses by employing a conditional discriminator that evaluates image pairs (rendered vs. target) from the same viewpoint, thereby preventing GAN mode collapse and ensuring 3D consistency across edited scenes.

### FENeRF
FENeRF uses semantic masks to edit the volume field via GAN inversion to enhance 3D GAN perception technology editability [S3].

## Operating characteristics
- **Measured Performance:** Evaluated on challenging indoor scenes from the ScanNet++ and Tanks and Temples datasets [S1]. GANeRF achieves significant improvements in perceptual metrics like LPIPS (reductions between 28-48%) while maintaining consistently better PSNR and SSIM scores compared to baselines like Nerfacto [S1].
- **Strengths:** Mitigates quality degradation from imperfect input data (limited coverage, distortion, illumination changes) [S1]. Enforces 3D-consistent photorealism [S1].
- **Limitations:** Requires careful tuning of adversarial loss multipliers [S2, unofficial]. May introduce artifacts if the discriminator is not well-trained [S1].
- **When to use:** When photorealism and texture detail are critical, and the input data is imperfect or limited [S1].
- **When not to use:** When computational resources are extremely limited, or when strict geometric accuracy is prioritized over photorealism [S1].

## Background
GANeRF was introduced by Barbara Roessle, Norman Müller, Lorenzo Porzi, Samuel Rota Bulò, Peter Kontschieder, and Matthias Nießner [S1]. It builds on the Nerfacto model [S1]. The method leverages generative adversarial networks (GANs) to improve NeRF quality in challenging real-world scenarios [S1]. It is related to concurrent work like NeRFLiX but differs in its adversarial formulation [S1].

## Sources
[S1] GANeRF: Leveraging Discriminators to Optimize Neural Radiance Fields, https://doi.org/10.1145/3618402, fetched 2026-10-26
[S2] barbararoessle/ganerf, https://github.com/barbararoessle/ganerf, fetched 2026-10-27 (unofficial)
[S3] Overview of NeRF technology and applications, https://www.jcad.cn/en/article/doi/10.3724/SP.J.1089.2023-00759, fetched 2026-10-31
