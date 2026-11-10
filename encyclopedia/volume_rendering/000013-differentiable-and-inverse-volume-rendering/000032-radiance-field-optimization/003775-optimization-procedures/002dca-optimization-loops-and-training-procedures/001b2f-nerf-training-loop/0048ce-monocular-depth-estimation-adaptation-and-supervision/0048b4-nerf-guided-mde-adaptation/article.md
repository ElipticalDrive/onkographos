# NeRF-guided MDE Adaptation

Also called: MDE Adaptation via NeRF, Scene-specific MDE fine-tuning

## Overview
This method adapts a pre-trained Monocular Depth Estimation (MDE) network to the specific absolute geometry of a target scene using a NeRF as a geometric prior [S1, unofficial]. It resolves ambiguities inherent in monocular priors, such as convexity/concavity confusion and flat surface orientation errors, by aligning the MDE output with the NeRF's rendered depth [S1, unofficial]. Unlike sibling methods that primarily use depth to regularize the NeRF, this approach explicitly fine-tunes the MDE network itself to become scene-aware [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Inputs:** A set of seen views $S$, an initial pre-trained MDE network $G_\phi$, and a NeRF model $\mathcal{N}$ (typically initialized from sparse inputs).
- **Outputs:** An adapted MDE network $G_\phi'$ and an optimized NeRF $\mathcal{N}'$.

**Data Structures:**
- **NeRF $\mathcal{N}$:** Standard volumetric representation. For a ray $r$, depth $\bar{D}(r)$ is computed as the expected distance: $\bar{D}(r) = \sum_i w_i y_i$, where $w_i$ is the transmittance-weighted contribution of sample $i$ at distance $y_i$ [Parent].
- **MDE Output $D^*$:** A pseudo-depth map extracted from the MDE network for an input image $I$.
- **Confidence Mask $M(p)$:** A scalar value per pixel $p \in [0, 1]$ indicating reliability. Not given in the sources how this is computed initially, but it is used to weight losses [S1, unofficial].

**Parameters:**
- **Learning Rates:** Adam optimizer. NeRF LR: $1 \cdot 10^{-2}$. MDE LR: $1 \cdot 10^{-5}$ [S1, unofficial].
- **Scheduler:** Cosine warmup [S1, unofficial].
- **Patch Size $P$:** Local patches are used for fitting to avoid skewing by dominant objects [Parent].

**Steps:**
1. **Initialization:** Initialize NeRF $\mathcal{N}$ and freeze MDE weights initially. Extract initial monocular depth $D^*_{init}$ from the pre-trained MDE for each seen view $I_i$ [S1, unofficial].
2. **NeRF Optimization (Standard):** Render depth $\bar{D}_i$ from NeRF for each seen view $I_i$.
3. **Alignment (Scale & Shift):** For each patch $p \in P$ in each seen view $I_i$, compute scale $w_i$ and shift $q_i$ via least squares to align the MDE prior $D^*_i$ with the NeRF depth $\bar{D}_i$:
   $$ \min_{w_i, q_i} \sum_{p \in P} \| (w_i \cdot \bar{D}_i(p) + q_i) - D^*_i(p) \|^2 $$
   The stop-gradient operator $\text{sg}(\cdot)$ is applied to the NeRF depth $\bar{D}_i$ in the loss terms to prevent gradients from flowing into the NeRF during MDE adaptation steps [S1, unofficial].
4. **MDE Adaptation Loss ($L_{MDE}$):** Update MDE weights to minimize the difference between the aligned NeRF depth and the MDE output. The loss includes an absolute alignment term and a patch-wise aligned term [S1, unofficial]:
   $$ L_{MDE} = \sum_{I_i \in S} \sum_{p \in P} \left( \| \text{sg}(\bar{D}_i(p)) - D^*_i(p) \| + \| (w_i \cdot \text{sg}(\bar{D}_i(p)) + q_i) - D^*_i(p) \| \right) $$
5. **Regularization ($L_{reg}$):** To preserve local smoothness of the adapted MDE and prevent overfitting to noisy NeRF depths, a regularization term is added using the *initial* pre-trained MDE output $D^*_{init}$ [S1, unofficial]:
   $$ L_{reg} = \sum_{I_i \in S} \sum_{p \in P} \| (w_i \cdot \text{sg}(D^*_{init}(p)) + q_i) - D^*_i(p) \| $$
6. **Confidence-Weighted Joint Training:** In advanced variants (e.g., DäRF), confidence masks $M_i(p)$ are introduced to weight the losses, mitigating error propagation from inaccurate depths [S1, unofficial]. The losses for seen views $L_{seen}$ and unseen views $L_{unseen}$ are redefined as [S1, unofficial]:
   $$ L_{seen} = \sum_{I_i \in S} \sum_{p \in P} M_i(p) \| (w_i \cdot \text{sg}(D^*_i(p)) + q_i) - \bar{D}_i(p) \| $$
   $$ L_{unseen} = \sum_{I_l \in U} \sum_{p \in P} M_l(p) \| (w_l \cdot \text{sg}(\bar{D}^*_l(p)) + q_l) - \bar{D}_l(p) \| $$
   The MDE loss with confidence is [S1, unofficial]:
   $$ L_{MDE} = \sum_{I_i \in S} \sum_{p \in P} M_i(p) \left( \| \text{sg}(\bar{D}_i(p)) - \bar{D}^*_i(p) \| + \| (w_i \cdot \text{sg}(\bar{D}_i(p)) + q_i) - \bar{D}^*_i(p) \| \right) $$
   Note: In the confidence-weighted formulation, $\bar{D}^*$ denotes the MDE output potentially adapted or refined, while $D^*$ in earlier equations referred to the raw extraction [S1, unofficial].
7. **Simultaneous Update:** Train NeRF and MDE simultaneously. MDE provides geometric priors to NeRF; NeRF provides absolute geometry to MDE [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def nerf_guided_mde_adaptation(nerf, mde, seen_views, optimizer_nerf, optimizer_mde, lr_nerf, lr_mde):
    # Initialize MDE with pre-trained weights
    mde.eval() # Freeze initially or use separate optimizer
    
    for iteration in range(max_iterations):
        for view in seen_views:
            # 1. Render NeRF depth
            # Assume render_depth returns expected distance D_bar
            D_bar = nerf.render_depth(view.image)
            
            # 2. Extract Monocular Depth
            # D_star is the output of the current MDE network
            D_star = mde(view.image)
            
            # 3. Compute Alignment Parameters (w, q) via Least Squares
            # Fit D_star to D_bar on patches to handle local geometry
            w, q = compute_least_squares_alignment(D_star, D_bar)
            
            # 4. Compute Losses
            # L_MDE: Adapt MDE to NeRF's absolute geometry
            # sg() stops gradients from flowing into NeRF
            loss_mde = 0
            for p in patches:
                # Term 1: Absolute alignment (no scale/shift)
                term1 = abs(sg(D_bar[p]) - D_star[p])
                # Term 2: Patch-wise alignment (with scale/shift)
                term2 = abs((w * sg(D_bar[p]) + q) - D_star[p])
                loss_mde += term1 + term2
            
            # L_reg: Regularization using initial MDE output to preserve smoothness
            # D_star_init is pre-computed from initial pre-trained MDE
            loss_reg = 0
            for p in patches:
                term_reg = abs((w * sg(D_star_init[p]) + q) - D_star[p])
                loss_reg += term_reg
                
            # 5. Update MDE
            optimizer_mde.zero_grad()
            (loss_mde + lambda_reg * loss_reg).backward()
            optimizer_mde.step()
            
            # 6. Update NeRF (Standard NeRF Loss + Depth Consistency)
            # L_seen uses the adapted MDE output as prior
            loss_nerf = compute_nerf_loss(view, D_bar, D_star, w, q)
            optimizer_nerf.zero_grad()
            loss_nerf.backward()
            optimizer_nerf.step()
```

## Subtopics

### Overall Training with Confidence
This variant incorporates confidence masks $M(p)$ to weight the depth consistency losses, preventing error propagation from inaccurate depths [S1, unofficial]. It redefines $L_{seen}$, $L_{unseen}$, and $L_{MDE}$ by multiplying the loss terms by $M_i(p)$ or $M_l(p)$ [S1, unofficial]. It is chosen when the NeRF or MDE outputs are noisy, ensuring that unreliable depth estimates do not corrupt the joint optimization [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Adds overhead to standard NeRF training due to MDE forward passes and least-squares fitting per patch. The simultaneous training requires balancing two optimizers [S1, unofficial].
- **Memory Complexity:** Requires storing the initial MDE output $D^*_{init}$ for regularization and potentially confidence maps [S1, unofficial].
- **Performance:** Improves NeRF reconstruction quality (PSNR, SSIM) on sparse inputs by providing better geometric priors [S1, unofficial]. On ScanNet (9-10 views), it achieves higher PSNR than baselines like NerfingMVS [S1, unofficial].
- **Strengths:** Resolves monocular ambiguities (concavity/convexity) and aligns scale/shift automatically [S1, unofficial].
- **Limitations:** Relies on the quality of the initial MDE and NeRF initialization. If the NeRF is poor, it may mislead the MDE [S1, unofficial].

## Background
Originated in the DäRF paper (2023) by [S1, unofficial]. It addresses the limitation of monocular depth estimation lacking scene-specific absolute depth priors and multi-view consistency [S1, unofficial]. It builds on the principle of using NeRF as a geometric prior for MDE adaptation [S1, unofficial].

## Sources
[S1] DaRF: Boosting Radiance Fields from Sparse Inputs with Monocular Depth Adaptation, https://doi.org/10.48550/arxiv.2305.19201, fetched 2026-10-29 (unofficial)
