# GANeRF

Also called: Adversarial Loss for GANerf, Adversarial NeRF Training, GAN-based NeRF Optimization, GANeRF Adversarial Training Loop, GANeRF

## Overview
GANeRF optimizes a 3D radiance field representation by integrating a 2D adversarial loss that enforces photorealism and 3D consistency [S1]. It solves the problem of quality degradation in novel view synthesis (NVS) caused by imperfect input data, such as limited coverage or illumination changes, by leveraging a per-scene 2D discriminator to guide the NeRF [S1]. Unlike sibling methods like GNeRF which may use global image discrimination or non-adversarial refinement networks [S1], GANeRF uses direct backpropagation from a patch-based discriminator into the 3D scene representation to reduce reconstruction imperfections [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Posed input images capturing a static 3D scene; a base NeRF architecture (e.g., Nerfacto [S1]).
*   **Outputs:** A refined 3D radiance field representation capable of synthesizing photorealistic novel views [S1].

**Data Structures:**
The implementation relies on the Nerfstudio framework structures [S2, unofficial]:
*   `GanerfPipelineConfig`: Extends `VanillaPipelineConfig` [S2, unofficial].
*   `GanerfModelConfig`: Contains the NeRF model and discriminator configuration [S2, unofficial].
*   `GanerfTrainerConfig`: Contains training hyperparameters including `adv_loss_mult`, `vgg_loss_mult`, and `gp_loss_mult` [S2, unofficial].
*   **Discriminator:** A 2D patch discriminator $D$ with resolution `img_resolution` [S2, unofficial].

**Parameters:**
*   `adv_loss_mult`: Multiplier for adversarial loss. Default: 0.0003 [S2, unofficial].
*   `vgg_loss_mult`: Multiplier for VGG perceptual loss. Default: 0.0003 [S2, unofficial].
*   `gp_loss_mult`: Multiplier for gradient penalty loss. Default: 0.2 [S2, unofficial].

**Algorithm Steps:**
1.  **Initialization:** Setup the NeRF model, discriminator, and optimizers [S2, unofficial].
2.  **Iteration Start:** Zero all optimizer gradients using `self.optimizers.zero_grad_all()` [S2, unofficial].
3.  **Check Adversarial Mode:** Evaluate if `self.config.adv_loss_mult > 0.0` [S2, unofficial].
4.  **Discriminator Training (if active):**
    a.  Extract real image patches from the ground truth batch and fake image patches from the NeRF rendering outputs [S2, unofficial].
    b.  Compute discriminator loss dictionary via `pipeline.get_discriminator_train_loss_dict` [S2, unofficial].
    c.  Sum all values in the dictionary into `d_loss` using `functools.reduce(torch.add, d_loss_dict.values())` [S2, unofficial].
    d.  Backpropagate `d_loss` via `d_loss.backward()` [S2, unofficial].
    e.  Verify gradients are finite using `has_finite_gradients` [S2, unofficial].
    f.  Step the discriminator optimizer [S2, unofficial].
5.  **NeRF Training:**
    a.  Compute NeRF loss dictionary via `pipeline.get_nerf_train_loss_dict` [S2, unofficial].
    b.  Sum values into `loss` [S2, unofficial].
    c.  If discriminator was trained, merge the discriminator loss dict into the NeRF loss dict before returning [S2, unofficial].
    d.  Backpropagate `loss` to update NeRF parameters [S2, unofficial].
6.  **Generator Refinement (Optional):** A 2D generator conditioned on multi-resolution NeRF renderings can be adversarially trained to refine outputs to match the real distribution [S1].

**Formulas:**
The adversarial loss for the NeRF (generator) is computed as [S2, unofficial]:
$$
\text{adv\_loss} = \text{mean}(\text{softplus}(-\text{pred\_fake}))
$$
where $\text{pred\_fake}$ is the discriminator's output for fake patches [S2, unofficial].

The gradient penalty (R1 penalty) for the discriminator is computed as [S2, unofficial]:
$$
\text{r1\_penalty} = \frac{1}{2} \mathbb{E}_{x \sim P_{real}} [\|\nabla_x D(x)\|_2^2]
$$
Implemented via `gradients.square().sum([1, 2, 3]).mean()` [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def train_iteration(self, batch, outputs):
    # 1. Zero gradients
    self.optimizers.zero_grad_all()

    # 2. Check if adversarial loss is active
    if self.config.adv_loss_mult > 0.0:
        # 3. Extract patches
        real_patches, fake_patches = self.get_patches(outputs, batch)

        # 4. Discriminator Loss
        pred_real = self.run_discriminator(real_patches)
        pred_fake = self.run_discriminator(fake_patches)
        
        # Standard GAN loss terms (implied by get_discriminator_train_loss_dict)
        d_loss_dict = self.compute_d_loss(real_patches, fake_patches)
        
        # Gradient Penalty
        gp_loss = self.compute_simple_gradient_penalty(real_patches)
        d_loss_dict["gp_loss"] = gp_loss * self.config.gp_loss_mult
        
        # Sum discriminator loss
        d_loss = functools.reduce(torch.add, d_loss_dict.values())
        
        # Update Discriminator
        d_loss.backward()
        assert has_finite_gradients(self.discriminator.parameters())
        self.discriminator_optimizer.step()

    # 5. NeRF Loss
    nerf_loss_dict = self.get_nerf_train_loss_dict(outputs, batch)
    
    # Merge discriminator loss if active
    if self.config.adv_loss_mult > 0.0:
        # Add adversarial loss to NeRF dict
        adv_loss = self.compute_adv_loss(fake_patches)
        nerf_loss_dict["adv_loss"] = adv_loss * self.config.adv_loss_mult
        # Add VGG perceptual loss if configured
        if self.config.vgg_loss_mult > 0.0:
            vgg_loss = self.compute_vgg_loss(outputs, batch)
            nerf_loss_dict["vgg_loss"] = vgg_loss * self.config.vgg_loss_mult
            
    loss = functools.reduce(torch.add, nerf_loss_dict.values())
    
    # Update NeRF
    loss.backward()
    self.neurf_optimizer.step()
```

## Subtopics

### FENeRF
Uses semantic masks to edit the volume field via GAN inversion to enhance 3D GAN perception technology editability [S3].

## Operating characteristics
*   **Time Complexity:** Adds overhead from discriminator forward/backward passes and patch extraction [S2, unofficial].
*   **Memory Complexity:** Requires storing discriminator parameters and intermediate gradients for patches [S2, unofficial].
*   **Performance:** Evaluated on challenging indoor scenes from ScanNet++ and Tanks and Temples [S1]. Achieves significant improvements in perceptual metrics like LPIPS (reductions between 28-48%) while maintaining better PSNR and SSIM scores compared to baselines [S1].
*   **Strengths:** Mitigates quality degradation from imperfect input data (limited coverage, distortion, illumination changes) [S1]. Enforces 3D-consistent photorealism [S1].
*   **Limitations:** Requires careful tuning of adversarial loss multipliers [S2, unofficial]. Sensitive to discriminator training stability [S2, unofficial].

## Background
GANeRF was introduced by Barbara Roessle, Norman Müller, Lorenzo Porzi, Samuel Rota Bulò, Peter Kontschieder, and Matthias Nießner [S1]. It builds on Neural Radiance Fields (NeRF) [Mildenhall et al. 2020] and specifically the Nerfacto model [Tancik et al. 2023] [S1]. It was evaluated on ScanNet++ [Yeshwanth et al. 2023] and Tanks and Temples [Knapitsch et al. 2017] datasets [S1].

## Sources
[S1] GANeRF: Leveraging Discriminators to Optimize Neural Radiance Fields, https://doi.org/10.1145/3618402, fetched 2026-10-26
[S2] barbararoessle/ganerf, https://github.com/barbararoessle/ganerf, fetched 2026-10-27 (unofficial)
[S3] Overview of NeRF technology and applications, https://www.jcad.cn/en/article/doi/10.3724/SP.J.1089.2023-00759, fetched 2026-10-31
