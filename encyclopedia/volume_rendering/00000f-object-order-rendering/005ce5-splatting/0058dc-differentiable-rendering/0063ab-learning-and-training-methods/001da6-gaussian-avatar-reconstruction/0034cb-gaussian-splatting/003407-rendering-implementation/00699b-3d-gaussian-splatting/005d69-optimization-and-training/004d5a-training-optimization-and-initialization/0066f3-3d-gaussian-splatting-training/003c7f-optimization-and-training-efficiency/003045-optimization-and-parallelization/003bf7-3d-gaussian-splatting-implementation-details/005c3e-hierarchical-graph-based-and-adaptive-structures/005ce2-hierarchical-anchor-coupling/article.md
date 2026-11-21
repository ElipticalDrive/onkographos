# Hierarchical Anchor Coupling

Also called: Per-level hierarchical anchor coupling

## Overview
Hierarchical Anchor Coupling (HAC) is a module that establishes cross-level gradient pathways in structured 3D Gaussian splatting by aggregating context from adjacent octree levels and broadcasting a residual feature adjustment to all visible anchors at a specific level [S1, unofficial]. It solves the problem of feature isolation, where backpropagation graphs in anchor-based octree pipelines lack connections between features at level $l_i$ and adjacent levels $l_i \pm 1$ [S1, unofficial]. Unlike sibling methods that may modify primitive geometry or rendering equations, HAC operates exclusively on the feature aggregation stage, introducing a shared residual component that preserves intra-level variation while enabling cross-level information flow [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** The set of visible anchors $V$ for a training camera, where each anchor $i$ has a base feature vector $f_i$ and an octree level $l_i$.
*   **Output:** Enhanced feature vectors $\hat{f}_i$ for each visible anchor, which replace $f_i$ as input to the downstream Neural Gaussian decoding MLPs.

**Data Structures:**
*   **Visible Anchor Set ($V$):** The collection of anchors overlapping the current view frustum.
*   **Level Set ($L_V$):** The set of unique octree levels present in $V$, defined as $L_V = \{l_i : i \in V\}$.
*   **Level-Specific Visible Anchors ($V_l$):** The subset of visible anchors at level $l$, defined as $V_l = \{i \in V : l_i = l\}$.
*   **Context Vector ($t_l$):** A concatenated vector of shape $\mathbb{R}^{3d}$, where $d$ is the feature dimension.
*   **Adjustment Vector ($\delta_l$):** A residual vector of shape $\mathbb{R}^d$ computed for each level.
*   **MLP ($g_\theta$):** A two-layer Multi-Layer Perceptron with ReLU activation, mapping $\mathbb{R}^{3d} \to \mathbb{R}^d \to \mathbb{R}^d$. The hidden dimension is $d=32$ [S1, unofficial].

**Parameters:**
*   **Granularity:** The method defaults to **per-level** granularity, computing a single shared adjustment $\delta_l$ for all anchors at level $l$. A **per-anchor** alternative exists but is not the default [S1, unofficial].
*   **Aggregation Operators:** $A_{par}$ (coarser-level) and $A_{chi}$ (finer-level) incorporate octree topology. These are specified by the Structure-Aware Containment Aggregation (SACA) module, which is external to HAC but required for context computation [S1, unofficial].

**Algorithm Steps:**
1.  Identify the set of visible levels $L_V$ and the visible anchors $V_l$ for each level $l \in L_V$.
2.  For each level $l \in L_V$, compute three context vectors:
    *   **Self-context ($c_{self}^l$):** The uniform mean of features at level $l$:
        $$ c_{self}^l = \frac{1}{|V_l|} \sum_{i \in V_l} f_i $$
    *   **Coarser-level context ($c_{par}^l$):** Aggregated from the adjacent coarser level ($l-1$) using operator $A_{par}$:
        $$ c_{par}^l = A_{par}(\{f_i\}_{i \in V_{l-1}}) $$
    *   **Finer-level context ($c_{chi}^l$):** Aggregated from the adjacent finer level ($l+1$) using operator $A_{chi}$:
        $$ c_{chi}^l = A_{chi}(\{f_i\}_{i \in V_{l+1}}) $$
3.  **Boundary Handling:** If level $l$ lacks an adjacent coarser level (e.g., root level) or finer level (e.g., leaf level), the corresponding context vector defaults to the self-context $c_{self}^l$ [S1, unofficial].
4.  Concatenate the three context vectors into a per-level triplet $t_l$:
    $$ t_l = [c_{par}^l; c_{self}^l; c_{chi}^l] \in \mathbb{R}^{3d} $$
5.  Pass the triplet through the MLP to compute the level-wise adjustment:
    $$ \delta_l = g_\theta(t_l) $$
6.  Broadcast the adjustment $\delta_l$ as a residual to all visible anchors at level $l$ to produce enhanced features:
    $$ \hat{f}_i = f_i + \delta_l, \quad \forall i \in V_l $$
7.  Replace original features $f_i$ with enhanced features $\hat{f}_i$ for downstream rendering.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def hierarchical_anchor_coupling(visible_anchors, level_set, aggregation_ops, mlp):
    """
    visible_anchors: List of objects with attributes .id, .level, .feature
    level_set: Set of unique levels present in visible_anchors
    aggregation_ops: Dictionary containing 'par' (coarser) and 'chi' (finer) functions
    mlp: Two-layer MLP function g_theta
    
    Returns:
    enhanced_features: Dictionary mapping anchor_id to enhanced feature vector
    """
    # Group anchors by level
    anchors_by_level = {}
    for anchor in visible_anchors:
        l = anchor.level
        if l not in anchors_by_level:
            anchors_by_level[l] = []
        anchors_by_level[l].append(anchor)

    enhanced_features = {}

    # Process each visible level
    for l in level_set:
        current_anchors = anchors_by_level[l]
        if not current_anchors:
            continue

        # 1. Compute Self-Context
        # Uniform mean of features at current level
        features_current = [a.feature for a in current_anchors]
        c_self = sum(features_current) / len(features_current)

        # 2. Compute Coarser-Level Context (l-1)
        level_coarser = l - 1
        if level_coarser in anchors_by_level and anchors_by_level[level_coarser]:
            features_coarser = [a.feature for a in anchors_by_level[level_coarser]]
            c_par = aggregation_ops['par'](features_coarser)
        else:
            # Boundary case: default to self-context
            c_par = c_self

        # 3. Compute Finer-Level Context (l+1)
        level_finer = l + 1
        if level_finer in anchors_by_level and anchors_by_level[level_finer]:
            features_finer = [a.feature for a in anchors_by_level[level_finer]]
            c_chi = aggregation_ops['chi'](features_finer)
        else:
            # Boundary case: default to self-context
            c_chi = c_self

        # 4. Concatenate into Triplet
        # Assuming feature dimension is d, triplet is 3d
        triplet = concatenate([c_par, c_self, c_chi])

        # 5. Compute Adjustment via MLP
        delta_l = mlp(triplet)

        # 6. Broadcast Residual to all anchors at this level
        for anchor in current_anchors:
            enhanced_features[anchor.id] = anchor.feature + delta_l

    return enhanced_features
```

**Edge Cases and Pitfalls:**
*   **Boundary Levels:** Levels at the top (root) or bottom (leaf) of the octree may lack adjacent coarser or finer levels. The algorithm must detect this and substitute the missing context with the self-context $c_{self}^l$ [S1, unofficial].
*   **Empty Levels:** If a level has no visible anchors, it is skipped in the loop, but its context may still be required by adjacent levels. The aggregation operators $A_{par}$ and $A_{chi}$ must handle empty input sets gracefully, likely returning zero vectors or defaulting to self-context if the level is absent from $L_V$ [S1, unofficial].
*   **Feature Dimension Consistency:** The MLP expects an input of size $3d$. If the base feature dimension $d$ varies, the concatenation and MLP weights must be adjusted accordingly [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** The MLP $g_\theta$ is invoked once per visible level, not once per visible anchor. The cost of MLP evaluation scales with the number of visible levels $|L_V|$. Computing level-wise means and broadcasting residuals scales linearly with the number of visible anchors $|V|$ [S1, unofficial].
*   **Memory Complexity:** HAC adds 4,160 parameters via the shared two-layer MLP (hidden dimension $d=32$). The total parameter increment is below 0.1% of the model size [S1, unofficial].
*   **Performance:** On the Mill19-Building scene, HAC yields a PSNR improvement of +1.91 dB over the baseline [S1, unofficial]. Training time increases slightly (e.g., from 2h15m to 2h22m in ablation studies) due to per-view aggregation, but the net rendering speed improves because HAC reduces the number of decoded and rasterized Neural Gaussians by roughly 35% [S1, unofficial].
*   **Granularity Trade-offs:**
    *   **Per-Level (Default):** Faster (~20% faster in training) and better for dense repetitive structures (e.g., +1.06 dB on Building). Acts as an implicit regularizer against over-parameterization [S1, unofficial].
    *   **Per-Anchor:** Computes a distinct $\delta_i$ for every anchor via spatial lookup. Cost grows linearly with $|V|$. Slightly better for simpler geometry (e.g., +0.21 dB on Rubble) but slower [S1, unofficial].
*   **Strengths:** Alleviates feature isolation; low parameter overhead; compatible with various backbones (CityGS-X, Octree-GS, Proxy-GS) [S1, unofficial].
*   **Limitations:** Relies on the underlying octree structure; performance gain depends on scene topology (less effective on simple geometry compared to complex structures) [S1, unofficial].

## Background
Hierarchical Anchor Coupling was introduced in the paper "TopoGS: Topology-Aware Anchor Feature Aggregation for Large-Scale 3D Gaussian Splatting" [S1, unofficial]. It was developed to address the lack of cross-level gradient pathways in existing anchor-based octree pipelines. The method is integrated into the TopoGS framework and has been shown to transfer effectively to other structured 3DGS backbones like Octree-GS and Proxy-GS [S1, unofficial].

## Sources
[S1] TopoGS: Topology-Aware Anchor Feature Aggregation for Large-Scale 3D Gaussian Splatting, https://arxiv.org/abs/2609.27868, fetched 2026-10-30 (unofficial)
