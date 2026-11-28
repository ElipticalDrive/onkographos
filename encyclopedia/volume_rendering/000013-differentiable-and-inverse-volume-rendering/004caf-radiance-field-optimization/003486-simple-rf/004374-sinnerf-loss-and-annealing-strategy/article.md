# SinNeRF Loss and Annealing Strategy

Also called: SinNeRF Total Loss and Annealing

## Overview
This method defines the composite objective function and its temporal scheduling for optimizing the SinNeRF radiance field. It solves the problem of balancing global structural coherence with local high-frequency detail reconstruction during single-image NeRF training. It sets itself apart from sibling methods like [Hard Sample Mining](7722-hard-sample-mining-for-nerf-optimization.md) by explicitly decoupling global structure supervision from local texture guidance via distinct loss terms ($\mathcal{L}_{\text{cls}}$ and $\mathcal{L}_{\text{adv}}$) and annealing their weights inversely to the training stride.

## How it works
**Inputs and Outputs**
*   **Inputs:** Current training iteration $t$, total training iterations $T$, initial weights $\lambda_1, \lambda_2, \lambda_3$, and computed loss components $\mathcal{L}_{\text{pix}}$, $\mathcal{L}_{\text{geo}}$, $\mathcal{L}_{\text{adv}}$, $\mathcal{L}_{\text{cls}}$.
*   **Outputs:** Scalar total loss $\mathcal{L}_{\text{total}}$ for backpropagation.

**Data Structures**
*   Loss weights: A vector or dictionary mapping loss keys to scalar floats.
*   Annealing schedule: Linear interpolation parameters derived from $t/T$.

**Formulas**
The total loss is defined as [S1]:
$$
\mathcal{L}_{\text{total}} = \mathcal{L}_{\text{pix}} + \lambda_1 \mathcal{L}_{\text{geo}} + \lambda_2 \mathcal{L}_{\text{adv}} + \lambda_3 \mathcal{L}_{\text{cls}}
$$
Where:
*   $\mathcal{L}_{\text{pix}}$: Pixel-wise reconstruction loss (e.g., L1 or L2).
*   $\mathcal{L}_{\text{geo}}$: Geometric regularization loss.
*   $\mathcal{L}_{\text{adv}}$: Adversarial loss for local texture guidance.
*   $\mathcal{L}_{\text{cls}}$: Classification loss (using DINO-ViT) for global structure prior.
*   $\lambda_1, \lambda_2, \lambda_3$: Time-dependent weighting factors.

**Parameters**
*   Initial weights [S1]: $\lambda_1 = 8$, $\lambda_2 = 0.1$, $\lambda_3 = 0$.
*   Annealing targets [S1]: $\lambda_2$ decreases to $0$; $\lambda_3$ increases to $0.1$.

**Algorithm**
1.  Compute base losses $\mathcal{L}_{\text{pix}}$, $\mathcal{L}_{\text{geo}}$, $\mathcal{L}_{\text{adv}}$, $\mathcal{L}_{\text{cls}}$ for the current batch/patch.
2.  Calculate the annealing coefficient $\alpha = t / T$ (clamped to $[0, 1]$).
3.  Update weights linearly [S1]:
    *   $\lambda_2(t) = 0.1 \times (1 - \alpha)$
    *   $\lambda_3(t) = 0.1 \times \alpha$
    *   $\lambda_1$ remains constant at $8$.
4.  Compute $\mathcal{L}_{\text{total}}$ using the formula above.

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_sinnerf_loss(l_pix, l_geo, l_adv, l_cls, t, T):
    # Initial weights
    lambda_1 = 8.0
    lambda_2_init = 0.1
    lambda_3_init = 0.0
    lambda_3_target = 0.1

    # Linear annealing schedule
    # alpha goes from 0 to 1 as t goes from 0 to T
    alpha = min(max(t / T, 0.0), 1.0)
    
    # Update weights
    # lambda_2 decreases from 0.1 to 0
    lambda_2 = lambda_2_init * (1.0 - alpha)
    
    # lambda_3 increases from 0 to 0.1
    lambda_3 = lambda_3_init + (lambda_3_target - lambda_3_init) * alpha
    
    # Total loss calculation
    l_total = l_pix + (lambda_1 * l_geo) + (lambda_2 * l_adv) + (lambda_3 * l_cls)
    
    return l_total
```

**Edge Cases and Pitfalls**
*   **Stride Dependency:** The annealing strategy is coupled with the sampling stride. Early training uses large strides (covering major regions), justifying high $\lambda_3$. Late training uses small strides (high-frequency details), requiring high $\lambda_2$. If stride scheduling is decoupled from loss annealing, the model may over-smooth or produce artifacts [S1].
*   **Zero Weighting:** At $t=0$, $\lambda_3=0$, meaning no global structure prior is applied initially. At $t=T$, $\lambda_2=0$, meaning adversarial texture guidance is dropped. Ensure the respective loss terms are computed even if their weight is zero to avoid graph errors, or conditionally skip computation if performance is critical [G].

## Subtopics

### Sigmoid Narrowing Factor
A technique used in related Visual Error Tomography (VET) frameworks to transition the alpha renderer to traditional depth testing [S2]. It uses a sigmoid narrowing factor $f$ to control the smoothness of this transition. Large values of $f$ cause abrupt changes during training, impacting quality. The recommended value is $f = 0.01$ [S2]. On the Redwood Chair scene, switching after 400 epochs (final at 600) with $f=10^{-2}$ yields an LPIPS of 0.186, compared to 0.200 for $f=100$ and 0.191 for $f=10^{-3}$ [S2].

## Operating characteristics
*   **Complexity:** The addition of $\mathcal{L}_{\text{cls}}$ (DINO-ViT) and $\mathcal{L}_{\text{adv}}$ (Discriminator) significantly increases computational cost per iteration compared to standard NeRF, requiring resizing patches to $224 \times 224$ for DINO-ViT [S1].
*   **Performance:** The annealing strategy allows the model to first capture global geometry and then refine textures. Without this, the model may struggle to converge on complex scenes from a single image [S1].
*   **Limitations:** The method relies on the availability of a pre-trained DINO-ViT and a GAN discriminator, increasing memory footprint and implementation complexity. The linear annealing is simple but may not be optimal for all scene complexities; however, it is the standard setting provided in the source [S1].

## Background
SinNeRF was introduced by [S1] to address the limitations of training Neural Radiance Fields from a single image, where lack of multi-view consistency leads to artifacts. The method builds on the original NeRF architecture [S1] but incorporates external priors (geometric, adversarial, and semantic) to constrain the solution space. The specific loss weighting strategy was designed to mimic a coarse-to-fine optimization process, aligning with the decreasing stride used in ray sampling.

## Sources
[S1] SinNeRF: Training Neural Radiance Fields on Complex Scenes from a Single Image, https://doi.org/10.1007/978-3-031-20047-2_42, fetched 2026-10-26
[S2] VET: Visual Error Tomography for Point Cloud Completion and High-Quality Neural Rendering, https://doi.org/10.1145/3610548.3618212, fetched 2026-10-26
