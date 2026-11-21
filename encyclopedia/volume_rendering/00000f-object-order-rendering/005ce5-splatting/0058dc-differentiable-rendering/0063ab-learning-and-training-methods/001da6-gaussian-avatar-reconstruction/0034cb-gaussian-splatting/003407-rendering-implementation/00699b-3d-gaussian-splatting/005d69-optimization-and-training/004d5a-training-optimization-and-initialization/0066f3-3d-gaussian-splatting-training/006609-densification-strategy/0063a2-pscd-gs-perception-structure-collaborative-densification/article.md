# PSCD-GS (Perception-Structure Collaborative Densification)
Also called: Role-Aware Perception-Structure Collaborative Densification

## Overview
PSCD-GS is a densification strategy for 3D Gaussian Splatting that replaces single-score heuristics with a continuous perception–structure role representation to guide densification, cloning/splitting selection, and split placement [S1, unofficial]. It augments the baseline 3DGS gradient and scale metrics with complementary perceptual (texture/appearance) and structural (geometry/boundaries) evidence, allowing the renderer to distinguish between regions needing appearance refinement versus geometric reconstruction [S1, unofficial]. Unlike sibling methods that rely on a single complexity or edge score, PSCD-GS retains the overlapping responses of both cue types to condition candidate selection and split initialization [S1, unofficial].

## How it works
**Inputs:** Training views (images $I_v$ and poses), current depth maps $Z_{kv}$ rendered from the current Gaussian set $\mathcal{G}_k$, and the current set of Gaussians $\mathcal{G}_k$.
**Outputs:** Updated Gaussian set $\mathcal{G}_{k+1}$ with new primitives created via cloning or splitting based on role-conditioned decisions.

**Data Structures:**
*   **Gaussian $g_i$:** Mean $\mu_i$, covariance $\Sigma_i$ (scale $s_i$, rotation $q_i$), opacity $\alpha_i$, SH coefficients $\mathbf{c}_i$.
*   **Evidence Maps:** Perceptual magnitude $A_v$, Structural magnitude $B_{kv}$, Structural tensor $R_{vB,k}$.
*   **Calibration State:** Exponential moving average quantiles $q^P_{k}, q^Q_{k}$ (low/high) for perceptual and structural scores.

**Parameters:**
*   $\tau_g$: Gradient threshold for densification eligibility.
*   $\tau_s$: Scene-scale-dependent threshold for cloning vs. splitting.
*   $\lambda_e, \lambda_d$: Structural mixture weights ($\lambda_e + \lambda_d = 1$).
*   $\tau_z$: Max reprojection depth error for validity.
*   $m_{min}$: Floor value for cross-view reliability ($>0$).
*   $\tau_C$: Visibility weight normalization threshold.
*   $K_{vis}$: View support threshold.
*   $\sigma_X$: Dispersion scale parameter for cross-view agreement.
*   $\beta_q$: EMA decay factor for quantile updates.
*   $\epsilon_{cal}$: Epsilon for calibration stability.
*   $\sigma_z, \sigma_h$: Dispersion scales for depth and structural difference.
*   $VGG-16$: Frozen ImageNet-pretrained network for perceptual features.

**Step 1: Baseline Eligibility**
Compute the average gradient magnitude $\bar{g}_i$ and maximum scale $s_{max,i}$ for each Gaussian $g_i$ visible in steps $T_i$ [S1, unofficial]:
$$ \bar{g}_i = \frac{1}{|T_i| + \epsilon} \sum_{t \in T_i} \left\| \frac{\partial L_t}{\partial \mu_i^{2D}} \right\|^2, \quad s_{max,i} = \|s_i\|_\infty $$
A Gaussian is eligible for densification if $\bar{g}_i > \tau_g$ [S1, unofficial].

**Step 2: Construct Perceptual Evidence**
For each training view $I_v$, extract features using a frozen VGG-16 network. Compute the perceptual magnitude map $A_v$ using selected layers $L$ and weights $\beta_l$ [S1, unofficial]:
$$ R^v_A(u) = RNorm_{\Omega_v} \left( \frac{\sum_{l \in L} \beta_l U_{pl} \left[ \sum_c (\nabla \phi_{l,c}(I_v))^2 \right]}{\sum_{l \in L} \beta_l U_{pl} \left[ \sum_c \nabla \phi_{l,c}(I_v) \nabla \phi_{l,c}(I_v)^T \right]} \right) $$
where $U_{pl}$ is spatial resizing, $\phi_{l,c}$ are channel-normalized feature maps, and $RNorm_{\Omega_v}$ is robust image-space normalization (clipping at fixed quantiles within view $v$ and mapping to $[0,1]$) [S1, unofficial].

**Step 3: Construct Structural Evidence**
Compute single-view luminance edge response $E_v$ and log-depth gradient response $D_{kv}$ [S1, unofficial]:
$$ E_v = RNorm_{\Omega_v}(\|\nabla Y_v\|_2), \quad D_{kv} = RNorm_{\Omega_v}(\|\nabla \log(Z_{kv} + \epsilon_z)\|_2) $$
Combine them into a raw structural response $H_{kv}$ [S1, unofficial]:
$$ H_{kv} = \lambda_e E_v + \lambda_d D_{kv} $$
Compute cross-view reliability $M_{kv}$ by back-projecting pixels from view $v$ to neighbors $v' \in N(v)$ using depth $Z_{kv}$ [S1, unofficial]:
$$ \Delta z_{vv'}(u) = \frac{Z_{kv}(u') - z_{v \to v'}}{\max(Z_{kv}(u'), z_{v \to v'}, \epsilon_z)} $$
$$ a_{vv'}(u) = \chi_{vv'}(u) \exp\left(-\frac{\Delta z_{vv'}(u)}{\sigma_z}\right) \exp\left(-\frac{\|H_{kv}(u) - H_{kv'}(u')\|}{\sigma_h}\right) $$
$$ M_{kv}(u) = \frac{\sum_{v' \in N(v)} a_{vv'}(u)}{\sum_{v' \in N(v)} \chi_{vv'}(u) + \epsilon} $$
where $\chi_{vv'}(u)=1$ if reprojection is inside neighbor image, depths are valid, and $\Delta z < \tau_z$, else 0. If no valid correspondence exists, $M_{kv}(u)=0$. The final structural magnitude $B_{kv}$ applies a floor $m_{min}$ to prevent removal of thin structures/occlusions [S1, unofficial]:
$$ B_{kv} = RNorm_{\Omega_v}(H_{kv}[m_{min} + (1 - m_{min})M_{kv}]) $$
Store the structural direction tensor $R_{vB,k}$ for split placement [S1, unofficial]:
$$ R_{vB,k}(u) = [m_{min} + (1 - m_{min})M_{kv}(u)] (\lambda_e g_v^E g_v^{E^T} + \lambda_d g_v^{D,k} g_v^{D,k^T}) $$
where $g_v^E, g_v^{D,k}$ are normalized gradient vectors.

**Step 4: Transfer Evidence to Gaussians**
For each visible Gaussian $g_i$, aggregate evidence from contributing pixels using alpha-compositing weights $\omega_{iv}$ [S1, unofficial]. Compute view support $\nu_{iv}$ and cross-view agreement $V_i^X$ [S1, unofficial]:
$$ \nu_{iv} = \min\left(1, \frac{C_v^i}{\tau_C}\right) \sum_{v'} \frac{\omega_{iv'}}{\sum_{v''} \omega_{iv''} + \epsilon} $$
$$ \mu_i^X = \frac{\sum_v \omega_{iv} x_{i,v}^X}{\sum_v \omega_{iv} + \epsilon}, \quad V_i^X = \frac{\sum_v \omega_{iv} (x_{i,v}^X - \mu_i^X)^2}{\sum_v \omega_{iv} + \epsilon} $$
Compute the raw evidence score $I_e^X$ (where $X \in \{A, B\}$) [S1, unofficial]:
$$ I_e^X_i = \min\left(1, \frac{\sum_v \nu_{iv}}{K_{vis}} \exp\left(-\frac{V_i^X}{2(\sigma_X + \epsilon)}\right)\right) \mu_i^X $$
If a Gaussian has no reliable view, set raw evidence to 0 [S1, unofficial].

**Step 5: Calibrate Scores**
Update quantiles using EMA and calibrate to $[0,1]$ [S1, unofficial]:
$$ q^X_{k} = \beta_q q^X_{k-1} + (1-\beta_q) \text{Quantile}_p \{I_e^X\}_{i \in \mathcal{G}_k} $$
$$ Calk^X(x) = \text{clip}\left(\frac{x - q^X_{low,k}}{q^X_{high,k} - q^X_{low,k} + \epsilon_{cal}}, 0, 1\right) $$
$$ P_i = Calk^A(I_e^A_i), \quad Q_i = Calk^B(I_e^B_i) $$
If the quantile span collapses, retain previous range; if no valid range, set branch to 0 [S1, unofficial].

**Step 6: Determine Roles and Densify**
Map scores to continuous memberships $m_i^P, m_i^Q$ using sigmoid functions with parameters $a^P, a^Q, \tau_P, \tau_Q$ [S1, unofficial]:
$$ m_i^P = \sigma[a^P(P_i - \tau_P)], \quad m_i^Q = \sigma[a^Q(Q_i - \tau_Q)] $$
Compute role memberships:
$$ \rho_i^{PQ} = m_i^P m_i^Q \quad (\text{Co-dominant}) $$
$$ \rho_i^{P\bar{Q}} = (1-m_i^Q)m_i^P \quad (\text{Perception-dominant}) $$
$$ \rho_i^{\bar{P}Q} = m_i^Q(1-m_i^P) \quad (\text{Structure-dominant}) $$
$$ \rho_i^{\bar{P}\bar{Q}} = (1-m_i^P)(1-m_i^Q) \quad (\text{Low-response}) $$
Use these roles to condition:
1.  **Candidate Selection:** Prioritize Gaussians with high $\rho_i^{PQ}$ or specific dominant roles based on loss.
2.  **Operation Selection:** Use $\tau_s$ and role to choose cloning (small scale, same center) or splitting (large scale, sampled from covariance) [S1, unofficial].
3.  **Split Placement:** Use the directional evidence from $R_{vB,k}$ or perceptual gradients to initialize child positions, rather than random sampling [S1, unofficial].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def pscd_densification(G, training_views, depth_maps, iteration):
    # 1. Baseline Eligibility
    eligible_gaussians = []
    for g in G:
        if g.visible_in_steps:
            g.avg_grad = compute_avg_gradient(g, training_views)
            g.max_scale = compute_max_scale(g)
            if g.avg_grad > tau_g:
                eligible_gaussians.append(g)
    
    if not eligible_gaussians:
        return G

    # 2. Construct Evidence Maps
    perceptual_maps = {}
    structural_maps = {}
    structural_tensors = {}
    
    for v in training_views:
        # Perceptual
        features = vgg16_extract(v.image) # Frozen
        perceptual_maps[v.id] = compute_perceptual_map(features, layers=L, weights=beta_l)
        
        # Structural
        Y_v = luminance(v.image)
        Z_kv = depth_maps[v.id]
        E_v = robust_normalize(laplacian(Y_v))
        D_kv = robust_normalize(laplacian(log(Z_kv + eps_z)))
        H_kv = lambda_e * E_v + lambda_d * D_kv
        
        # Cross-view reliability
        M_kv = compute_cross_view_reliability(H_kv, Z_kv, training_views, tau_z, sigma_z, sigma_h)
        B_kv = robust_normalize(H_kv * (m_min + (1 - m_min) * M_kv))
        
        # Tensor for direction
        g_E = normalize_gradient(E_v)
        g_D = normalize_gradient(D_kv)
        R_vB = (m_min + (1 - m_min) * M_kv) * (lambda_e * outer(g_E, g_E) + lambda_d * outer(g_D, g_D))
        
        structural_maps[v.id] = B_kv
        structural_tensors[v.id] = R_vB

    # 3. Transfer to Gaussians & Calibrate
    raw_scores_A = {}
    raw_scores_B = {}
    
    for g in eligible_gaussians:
        # Aggregate perceptual evidence
        score_A = aggregate_evidence(g, perceptual_maps, training_views, tau_C, K_vis, sigma_X)
        raw_scores_A[g.id] = score_A
        
        # Aggregate structural evidence
        score_B = aggregate_evidence(g, structural_maps, training_views, tau_C, K_vis, sigma_X)
        raw_scores_B[g.id] = score_B

    # Update Quantiles
    q_A_low, q_A_high = update_quantiles(raw_scores_A, q_A_prev_low, q_A_prev_high, beta_q)
    q_B_low, q_B_high = update_quantiles(raw_scores_B, q_B_prev_low, q_B_prev_high, beta_q)
    
    # Calibrate
    scores_P = {}
    scores_Q = {}
    for g in eligible_gaussians:
        scores_P[g.id] = clip((raw_scores_A[g.id] - q_A_low) / (q_A_high - q_A_low + eps_cal), 0, 1)
        scores_Q[g.id] = clip((raw_scores_B[g.id] - q_B_low) / (q_B_high - q_B_low + eps_cal), 0, 1)

    # 4. Role Assignment & Densification
    new_gaussians = []
    for g in eligible_gaussians:
        m_P = sigmoid(a_P * (scores_P[g.id] - tau_P))
        m_Q = sigmoid(a_Q * (scores_Q[g.id] - tau_Q))
        
        # Determine role
        role = determine_role(m_P, m_Q)
        
        # Decide operation
        if g.max_scale < tau_s:
            # Clone
            child = clone(g)
            # Adjust position based on role? (Source implies direction guides split, clone is same center)
            new_gaussians.append(child)
        else:
            # Split
            # Use structural tensor or perceptual gradient for direction
            direction = get_evidence_direction(g, structural_tensors, perceptual_maps)
            children = split(g, direction=direction)
            new_gaussians.extend(children)
            
        # Remove parent
        g.mark_for_removal()
        
    return compact(G + new_gaussians)
```

## Operating characteristics
*   **Complexity:** Adds overhead of VGG-16 feature extraction and cross-view reprojection for structural reliability at each densification event. VGG-16 is frozen and efficient. Cross-view pooling is $O(N \cdot |N(v)|)$.
*   **Memory:** Requires storing evidence maps and tensors per view during densification events. Quantile states are minimal.
*   **Performance:** Improves reconstruction quality in texture-rich and geometrically complex regions by providing more targeted densification signals than gradient magnitude alone [S1, unofficial].
*   **Strengths:** Handles regions with high texture but low geometry (perception-dominant) and vice versa (structure-dominant) differently. Prevents artificial evidence from constant fields via calibration guards [S1, unofficial].
*   **Limitations:** Requires depth maps for structural cues. Sensitive to depth accuracy. More hyperparameters than baseline 3DGS ($\lambda_e, \lambda_d, m_{min}, \sigma_X$, etc.) [S1, unofficial].

## Background
PSCD-GS was introduced to address the limitations of single-score densification in 3D Gaussian Splatting, where gradients and scales alone cannot distinguish between appearance and geometry needs [S1, unofficial]. It builds on the baseline adaptive density control of 3DGS [Parent] and integrates concepts from perceptual loss (LPIPS/VGG) and multi-view geometry [S1, unofficial].

## Sources
[S1] PSCD-GS: Role-Aware Perception-Structure Collaborative Densification for 3D Gaussian Splatting, https://doi.org/10.21203/rs.3.rs-10466466/v1, fetched 2026-10-26 (unofficial)
