# Contribution-based Pruning

Also called: Contribution Pruning, Contribution-Prune

## Overview
Contribution-based Pruning is a density control strategy in 3D Gaussian Splatting that removes redundant Gaussian primitives by jointly evaluating their opacity and spatial scale attributes [S3, unofficial]. It addresses the problem of superfluous or low-impact Gaussians accumulating during densification, which increases computational load and storage costs without improving rendering fidelity [S1][S3, unofficial]. Unlike [Opacity Pruning](14627-opacity-pruning.md), which relies solely on transparency, or [Size-based Pruning](17737-size-based-pruning.md), which targets geometric extent alone, this method combines both metrics to identify Gaussians that are simultaneously invisible and geometrically insignificant [S3, unofficial].

## How it works
**Inputs and Outputs:**
- Input: The current set of 3D Gaussians $\mathcal{G}$, each with attributes including opacity $o_g$ and scale parameters (from which maximum scale $s_{\max}(g)$ is derived).
- Output: A pruned set of Gaussians $\mathcal{G}_{\text{new}}$ with redundant primitives removed.

**Data Structures:**
- Gaussian Set $\mathcal{G}$: A collection of Gaussian primitives.
- Vote Counter $V_g$: An integer accumulator for each Gaussian $g$, initialized to 0.
- Pruning Candidates Sets: $\mathcal{C}_{\text{opacity}}^{(t)}$ and $\mathcal{C}_{\text{scale}}^{(t)}$.

**Parameters:**
- $\tau$: Pruning threshold (quantile). Determines the cutoff for "low" opacity and scale. Typical value: 0.2 [S3, unofficial].
- $I_{\text{prune}}$: Pruning interval (number of iterations between pruning checks).
- $v$: Vote threshold. A Gaussian is removed if its accumulated votes exceed $I_{\text{prune}} \cdot v$.

**Steps:**
1. **Compute Attributes:** At iteration $t$, for each Gaussian $g \in \mathcal{G}$, retrieve its current opacity $o_g$ and compute its maximum scale $s_{\max}(g)$ across all dimensions.
2. **Determine Candidates:** Calculate the $\tau$-quantile for opacity $Q_\tau(o)$ and maximum scale $Q_\tau(s_{\max})$ across the entire set $\mathcal{G}$. Identify candidates for pruning based on opacity and scale:
   $$ \mathcal{C}_{\text{opacity}}^{(t)} = \{g \in \mathcal{G} \mid o_g < Q_\tau(o)\} $$
   $$ \mathcal{C}_{\text{scale}}^{(t)} = \{g \in \mathcal{G} \mid s_{\max}(g) < Q_\tau(s_{\max})\} $$
3. **Intersect Candidates:** The set of pruning candidates $\mathcal{C}_{\text{prune}}^{(t)}$ is the intersection of opacity and scale candidates:
   $$ \mathcal{C}_{\text{prune}}^{(t)} = \mathcal{C}_{\text{opacity}}^{(t)} \cap \mathcal{C}_{\text{scale}}^{(t)} $$
4. **Accumulate Votes:** For each Gaussian $g$, update its vote count $V_g^{(t)}$. If $g \in \mathcal{C}_{\text{prune}}^{(t)}$, increment the vote by 1. Otherwise, the vote remains unchanged (or is not incremented).
   $$ V_g^{(t+1)} = V_g^{(t)} + \mathbf{1}_{\{g \in \mathcal{C}_{\text{prune}}^{(t)}\}} $$
   where $\mathbf{1}$ is the indicator function.
5. **Prune:** If the current iteration $t$ is a multiple of $I_{\text{prune}}$, permanently remove all Gaussians $g$ where $V_g^{(t)} > I_{\text{prune}} \cdot v$. The updated set is $\mathcal{G}_{\text{new}} = \mathcal{G} \setminus \{g \mid V_g^{(t)} > I_{\text{prune}} \cdot v\}$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def contribution_based_pruning(Gaussians, tau, I_prune, v):
    """
    Prunes Gaussians based on joint opacity and scale contribution.
    
    Args:
        Gaussians: List of Gaussian objects, each having .opacity and .scale (array/vector).
        tau: Quantile threshold (e.g., 0.2).
        I_prune: Pruning interval (iterations).
        v: Vote threshold multiplier.
        
    Returns:
        Pruned list of Gaussians.
    """
    # Initialize vote counters if not present
    for g in Gaussians:
        if not hasattr(g, 'vote_count'):
            g.vote_count = 0
            
    # Determine if it is time to prune
    current_iter = get_current_iteration() # [G] Helper to track iteration
    if current_iter % I_prune != 0:
        # Update votes but do not prune yet
        update_votes(Gaussians, tau)
        return Gaussians
        
    # Step 1 & 2: Identify candidates
    opacities = [g.opacity for g in Gaussians]
    max_scales = [max(g.scale) for g in Gaussians] # max across dimensions
    
    # Calculate quantiles
    q_opacity = quantile(opacities, tau)
    q_scale = quantile(max_scales, tau)
    
    candidates = []
    for g in Gaussians:
        if g.opacity < q_opacity and max(g.scale) < q_scale:
            candidates.append(g)
            
    # Step 4: Accumulate votes
    for g in Gaussians:
        if g in candidates:
            g.vote_count += 1
            
    # Step 5: Prune
    threshold_votes = I_prune * v
    pruned_gaussians = [g for g in Gaussians if g.vote_count <= threshold_votes]
    
    return pruned_gaussians

def update_votes(Gaussians, tau):
    """Helper to update votes without pruning."""
    opacities = [g.opacity for g in Gaussians]
    max_scales = [max(g.scale) for g in Gaussians]
    
    q_opacity = quantile(opacities, tau)
    q_scale = quantile(max_scales, tau)
    
    for g in Gaussians:
        if g.opacity < q_opacity and max(g.scale) < q_scale:
            g.vote_count += 1
```

**Edge Cases and Pitfalls:**
- **Early Training Noise:** Opacity and scale values can fluctuate significantly during early training. Immediate removal based on a single iteration's metrics may remove valid Gaussians. The voting mechanism mitigates this by requiring consistent low contribution over multiple iterations [S3, unofficial].
- **Threshold Sensitivity:** A larger $\tau$ results in more aggressive pruning (lower quantile cutoff), potentially removing more Gaussians but risking quality degradation if set too high [S3, unofficial].
- **Single-Criterion Failure:** Pruning based solely on opacity or scale leads to substantial performance degradation compared to using both jointly [S3, unofficial].

## Subtopics

### Accumulated Vote Pruning
A variant that explicitly formalizes the vote accumulation mechanism to mitigate noisy fluctuations in opacity or scale during early training [S3, unofficial]. Instead of removing candidates immediately, it accumulates votes $V_g^{(t)}$ for Gaussians selected as pruning candidates $\mathcal{C}_{\text{prune}}^{(t)}$ and permanently prunes them only when $V_g^{(t)} > I_{\text{prune}} \cdot v$ [S3, unofficial]. This ensures that only Gaussians consistently exhibiting low contribution are removed [S3, unofficial].

### Distribution-based Pruning
A strategy that restricts pruning to the initial 25k iterations to prevent excessive removal during later stages when finer details are critical [S3, unofficial]. It uses specific parameters: $I_{\text{prune}} = 1000$, $v = 0.6$, and redundancy threshold $\tau = 0.2$ [S3, unofficial]. This allows the model to retain representation capacity while reducing unnecessary primitives in early training [S3, unofficial].

### θ+Opacity Rule Pruning
A pruning rule based on theta (coverage) and opacity that localizes sacrifice at city scale, serving as a complementary export to score-based methods [S4, unofficial]. It is designed to remove perceptually redundant mass [S4, unofficial]. At extreme budgets (e.g., 10% primitives), it achieves the best LPIPS scores (0.547 vs 0.557 for score export), keeping street markings and facades legible where other methods blur or collapse [S4, unofficial].

### Iterative Pruning
Removes Gaussian points in multiple sequential iterations (e.g., Iter1-3 with rate 0.4, Iter4 with rate 0.2) to reduce model size while maintaining visual quality [S5]. This approach allows for fine-tuning between pruning steps, adapting to the remaining point distribution and achieving a better quality-size tradeoff compared to single-pass pruning [S5]. It can remove 87% of points with an average PSNR drop of only 0.23 dB [S5].

### GTC-guided Shrinking
A strategy that prunes Gaussian-tile pairs based on a Gaussian Tile Contribution (GTC) metric, guided by a scaling factor $\lambda_G$ (typical value 0.2) [S6, unofficial]. It uses a scene-adaptive threshold $\tau$ determined by the GTC metric, unlike fixed thresholds in other methods [S6, unofficial]. It preserves high-frequency regions (dense, structurally meaningful Gaussians) while removing redundant pairs in low-frequency regions, achieving a 20% FPS improvement with 1% PSNR degradation [S6, unofficial].

### Greedy Selection
A selection algorithm that deletes Gaussians one by one, choosing the one that causes the smallest increase in the unpenalized deletion cost, until the target count is reached [S7, unofficial]. It satisfies the count constraint by construction and does not require a count penalty coefficient [S7, unofficial]. The cost increment for retaining member $j$ is calculated as $\Delta_{\text{add}}(j) = 2(Rz)_j + R_{jj}$ [S7, unofficial].

### Spatial Gaussian Allocation Analysis
Analysis of where Gaussians are redistributed by occlusion-aware filters, comparing occlusion-blind count, occlusion-aware count, and average-contribution criterion [S2, unofficial]. The average-contribution criterion concentrates Gaussians on central regions, using more Gaussians (0.97M) than the blind count (0.73M) on the kitchen scene to maintain quality [S2, unofficial].

### Opacity + Gradient Based Pruning
A combined strategy applying both opacity-based and gradient-based pruning to effectively remove floaters and redundant Gaussians [S8]. Used by methods like Trimming the Fat and ELMGS, it can remove 75% of Gaussians while maintaining high visual quality [S8].

## Operating characteristics
**Time and Memory Complexity:**
- The pruning step involves computing quantiles over the Gaussian set, which is $O(N \log N)$ or $O(N)$ depending on the quantile algorithm, where $N$ is the number of Gaussians [S3, unofficial].
- Vote accumulation is $O(N)$ per iteration [S3, unofficial].
- Memory overhead is minimal, requiring only an integer vote counter per Gaussian [S3, unofficial].

**Measured Performance:**
- On Mobile-GS (baseline 0.56M Gaussians, 27.22 PSNR, 109 FPS), applying pruning with $\tau=0.2$ reduces Gaussians to 0.47M, maintains 27.12 PSNR, and increases FPS to 127 [S3, unofficial].
- Pruning based solely on opacity or scale results in lower PSNR (26.84 and 26.87 respectively) compared to the joint method (27.12) [S3, unofficial].

**Strengths and Limitations:**
- **Strengths:** Reduces storage and computational costs significantly while maintaining high rendering fidelity [S3, unofficial]. The voting mechanism prevents premature removal of valid Gaussians during training fluctuations [S3, unofficial].
- **Limitations:** Requires careful tuning of $\tau$, $I_{\text{prune}}$, and $v$ [S3, unofficial]. Aggressive pruning (high $\tau$) can lead to visible artifacts in low-frequency regions if not balanced with other strategies like GTC-guided shrinking [S6, unofficial].

**When to Use:**
- Use when computational resources are limited (e.g., mobile devices) or when storage size is a constraint [S3, unofficial].
- Use in conjunction with other pruning methods (e.g., MaskGaussian, Mini-Splatting) for further reduction [S3, unofficial].

**When Not to Use:**
- Avoid if the scene requires very high fidelity in low-frequency regions and the voting threshold is not carefully tuned, as it may remove smooth blending Gaussians [S6, unofficial].
- Not recommended to rely on opacity or scale alone, as this degrades performance [S3, unofficial].

## Background
Contribution-based Pruning was introduced as part of the Mobile-GS method for real-time Gaussian Splatting on mobile devices [S3, unofficial]. It addresses the need for efficient rendering on hardware with limited computational power and memory [S3, unofficial]. The method builds on the observation that Gaussians with low opacity and small scale contribute minimally to the final image [S3, unofficial]. It differs from earlier pruning methods that might remove Gaussians based on a single attribute or without considering the temporal consistency of their contribution [S3, unofficial].

## Sources
[S1] A Survey on 3D Gaussian Splatting, https://doi.org/10.1145/3807511, fetched 2026-10-27
[S2] Counting Contributions, Not Views: Occlusion-Aware Densification for 3D Gaussian Splatting, https://arxiv.org/abs/2610.27108, fetched 2026-10-26 (unofficial)
[S3] Mobile-GS: Real-time Gaussian Splatting for Mobile Devices, https://doi.org/10.48550/arXiv.2603.11531, fetched 2026-10-29 (unofficial)
[S4] Budgeted-GS: Real-Time Large-Scale Gaussian Splatting via Factoring LOD, https://arxiv.org/abs/2610.03162, fetched 2026-10-29 (unofficial)
[S5] A 129 FPS Full HD Real-Time Accelerator for 3D Gaussian Splatting., https://doi.org/10.1109/tvcg.2026.3683714, fetched 2026-10-27
[S6] FilterGS: Traversal-Free Parallel Filtering and Adaptive Shrinking for Large-Scale LoD 3D Gaussian Splatting, https://doi.org/10.48550/arXiv.2603.23891, fetched 2026-10-29 (unofficial)
[S7] SplatQ: Interaction-Aware Pruning of 3D Gaussian Splatting via Quadratic Selection, https://arxiv.org/abs/2610.13503, fetched 2026-10-29 (unofficial)
[S8] Compression in 3D Gaussian Splatting: A Survey of Methods, Trends, and Future Directions, https://doi.org/10.1109/TCSVT.2026.3676048, fetched 2026-10-26
