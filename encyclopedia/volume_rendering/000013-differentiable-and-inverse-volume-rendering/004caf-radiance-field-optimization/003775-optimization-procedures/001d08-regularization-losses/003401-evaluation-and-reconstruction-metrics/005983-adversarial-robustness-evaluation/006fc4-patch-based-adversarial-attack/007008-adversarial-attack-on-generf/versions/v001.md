# Adversarial Attack on GeNeRF

Also called: GeNeRF Adversarial Attack, Targeted Adversarial Attack on Generalizable Neural Radiance Fields

## Overview
A targeted adversarial attack method for Generalizable Neural Radiance Fields (GeNeRFs) that modifies source images (views) to force the rendered output to match a manually created 'adversarial ground truth' image containing hallucinated objects, deleted objects, or modified shapes [S1]. Unlike standard classification attacks that perturb a single image, this method exploits the multi-view nature of GeNeRFs by simultaneously modifying a subset or all of the input source views [S1]. It distinguishes itself from sibling methods like [Fast Gradient Sign Method (FGSM)](21941-fast-gradient-sign-method-fgsm.md) by integrating a residual loss regularizer that preserves image quality in non-attacked regions, ensuring gradients from unmodified areas are considered during optimization [S1].

## How it works
**Inputs:**
1. A pretrained GeNeRF model (e.g., IBRNet) [S1].
2. A specific camera pose and a set of $N$ source images (views) $I = \{I_1, I_2, ..., I_N\}$ [S1].
3. A manually modified "adversarial ground truth" image $I_{adv}$, created by altering a baseline render of the target pose (adding hallucinated objects, deleting existing ones, or modifying shapes) [S1].

**Outputs:**
1. Perturbed source images $I' = \{I'_1, I'_2, ..., I'_N\}$ [S1].
2. The resulting rendered image $I_{render}$ which approximates $I_{adv}$ [S1].

**Data Structures:**
- Source images: Tensors of shape $(H, W, C)$, where $H, W$ are height and width, and $C$ is channels (typically 3 for RGB). Values are normalized to $[0, 1]$ or $[-1, 1]$ depending on the GeNeRF implementation [S1].
- Perturbation buffer: Tensor of same shape as source images, storing the additive noise $\delta$ [S1].

**Parameters:**
- $\epsilon$ (epsilon): Maximum intensity of noise/perturbation allowed per pixel. Typical value: $0.01$ [S1].
- $T$: Number of optimization iterations. Typical value: $1000$ [S1].
- $N_{src}$: Number of source images provided to the GeNeRF. Tested values: $10, 8, 6, 5, 4$ [S1].
- $N_{att}$: Number of source images subjected to perturbation. Tested values: $1$ up to $N_{src}$ [S1].
- $P_{size}$: Patch size for patch-based attacks (e.g., $2\times2, 5\times5, 10\times10, 20\times20$) [S1].

**Algorithm Steps:**
1. **Baseline Rendering:** Render a baseline image $I_{base}$ at the target camera pose using the original source images $I$ and the pretrained GeNeRF model $\mathcal{M}$ [S1].
   $$ I_{base} = \mathcal{M}(I, \text{pose}) $$
2. **Adversarial Ground Truth Creation:** Manually modify $I_{base}$ to create $I_{adv}$. Modifications include:
   - Modifying the shape of existing objects.
   - Deleting existing objects and substituting with background pixels.
   - Adding new hallucinated objects [S1].
3. **Initialization:** Initialize the perturbation $\delta$ for the selected $N_{att}$ source images to zero or small random noise. If using patch-based attacks, initialize patches at the center of the images [S1].
4. **Optimization Loop:** For $t = 1$ to $T$:
   a. **Forward Pass:** Compute the perturbed source images $I'_k = I_k + \delta_k$ for attacked views $k \in \{1, ..., N_{att}\}$, keeping unattacked views unchanged. Render the current image $I_{render}$:
      $$ I_{render} = \mathcal{M}(I', \text{pose}) $$
   b. **Residual Loss Calculation:** Compute the residual loss $L_{res}$ between $I_{adv}$ and $I_{render}$ using the Average L2 distance:
      $$ L_{res} = \frac{1}{H \cdot W \cdot C} \sum_{h,w,c} (I_{adv}[h,w,c] - I_{render}[h,w,c])^2 $$
      This loss acts as a regularizer to prevent deterioration of image quality in non-attacked regions by ensuring gradients from those regions are considered [S1].
   c. **Back-propagation:** Back-propagate the gradients of $L_{res}$ through the differentiable GeNeRF pipeline to update $\delta$ [S1].
   d. **Update Rule:** For low-intensity attacks, apply the iterative FGSM with momentum algorithm [S1]. Let $g_t$ be the gradient of $L_{res}$ with respect to the perturbation $\delta$. Update momentum $m_t$ and perturbation $\delta_t$:
      $$ m_t = \beta m_{t-1} + \frac{g_t}{\|g_t\|_1} $$
      $$ \delta_t = \delta_{t-1} + \alpha \cdot \text{sign}(m_t) $$
      Where $\beta$ is the momentum coefficient and $\alpha$ is the step size [S1].
   e. **Projection:** Clip $\delta_t$ to ensure pixel values remain within valid bounds and respect $\epsilon$:
      $$ \delta_t = \text{clip}(\delta_t, -\epsilon, \epsilon) $$
      $$ I'_k = \text{clip}(I_k + \delta_t, \text{min\_val}, \text{max\_val}) $$
5. **Evaluation:** Compute the final attack success metric: Average L2 distance between $I_{render}$ and $I_{adv}$ [S1].

**Edge Cases and Pitfalls:**
- **Manual Modification Bias:** Manual modifications may disturb real image structure (causing strong edges) or be subjective. This bias is unavoidable but was mitigated by generating three distinct types of modifications [S1].
- **Patch Placement:** In patch-based attacks, placing patches at the center ensures they are not closely positioned to regions already modified, testing far-reaching effects [S1].
- **Success Thresholds:** An attack is considered successful if the average pixel distance is below $0.015$; above $0.020$ is unsuccessful. These thresholds may vary by scene [S1].

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of iterations $T$ and the number of source images $N_{src}$, as each iteration requires a forward pass through the GeNeRF and back-propagation [S1].
- **Memory Complexity:** Proportional to the number of source images $N_{src}$ and their resolution, as gradients must be stored for all attacked views [S1].
- **Measured Performance:** Tested on 10 different scenes, with each attack repeated 10 times to average stochastic nature [S1]. Attacks were successful in most cases when a significant majority of source views were targeted [S1].
- **Scaling:** The quality of the generated image generally improves with an increase in the number of source images $N_{src}$ [S1].
- **Strengths:** Exploits multi-view consistency; residual loss preserves non-attacked regions; effective against GeNeRFs like IBRNet [S1].
- **Limitations:** Requires manual creation of adversarial ground truth; subjective modifications; vulnerable when majority of views are attacked [S1].
- **When to use:** Evaluating robustness of GeNeRFs against targeted manipulations; security assessment of multi-view reconstruction systems [S1].
- **When not to use:** When computational resources are limited for iterative optimization; when untargeted attacks are sufficient for evaluation [S1].

## Background
Originated in the paper "Targeted Adversarial Attacks on Generalizable Neural Radiance Fields" [S1]. The method addresses the gap in adversarial attack research for GeNeRFs, as prior work was restricted to low-intensity and untargeted attacks [S1]. It builds on the differentiable nature of GeNeRFs like IBRNet, allowing gradients to flow from the rendered output back to the input source images [S1]. The approach highlights the robustness of GeNeRFs when most views are untouched but underscores vulnerability when a significant majority are targeted [S1].

## Sources
[S1] Targeted Adversarial Attacks on Generalizable Neural Radiance Fields, https://doi.org/10.1109/iccvw60793.2023.00399, fetched 2026-10-29
