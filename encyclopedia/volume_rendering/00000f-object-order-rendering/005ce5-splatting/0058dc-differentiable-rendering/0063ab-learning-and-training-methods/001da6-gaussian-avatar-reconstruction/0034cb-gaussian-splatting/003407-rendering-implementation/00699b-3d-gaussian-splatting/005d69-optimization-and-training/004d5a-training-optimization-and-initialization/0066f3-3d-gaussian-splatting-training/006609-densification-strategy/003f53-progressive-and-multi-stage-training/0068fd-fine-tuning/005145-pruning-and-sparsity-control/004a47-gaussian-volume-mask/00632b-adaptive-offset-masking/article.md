# Adaptive Offset Masking

Also called: Learnable Binary Masking for Anchors and Offsets

## Overview
Adaptive Offset Masking is a pruning strategy that uses learnable binary masks to eliminate invalid Gaussians and their parent anchors, integrating the mask directly into the rate calculation for adaptive updates without requiring extra loss terms [S1]. It distinguishes itself from static pruning methods by allowing the mask ratio to be adaptively adjusted according to the compression rate via backpropagation, thereby balancing fidelity and bit consumption dynamically [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Learnable features $\mathbf{f}_i^m \in \mathbb{R}^K$ for each anchor $i$, where $K$ is the number of Gaussian offsets per anchor. Threshold $\epsilon_m$ for mask generation. Trade-off hyperparameter $\lambda$ [S1].
*   **Outputs:** Binary Gaussian-level masks $m_{i,k}$, binary anchor-level masks $m_i^a$, and updated entropy loss $L_{\text{entropy}}$ [S1].

**Data Structures:**
*   $\mathbf{f}_i^m$: A vector of size $K$ containing learnable features used to determine the validity of each offset within anchor $i$ [S1].
*   $m_{i,k}$: A binary scalar (0 or 1) indicating if the $k$-th offset of anchor $i$ is valid [S1].
*   $m_i^a$: A binary scalar indicating if anchor $i$ is valid [S1].

**Numbered Steps:**
1.  **Gaussian-Level Mask Generation:** For each anchor $i$, compute the Gaussian-level mask $m_i$ using the learnable feature $\mathbf{f}_i^m$ and a sigmoid function with straight-through estimation [S1].
2.  **Anchor-Level Mask Derivation:** Compute the average offset mask ratio $\overline{m}_i$ for anchor $i$. Derive the anchor-level mask $m_i^a$ from $\overline{m}_i$ [S1].
3.  **Pruning Logic:** If all offsets in an anchor are pruned ($m_i = \mathbf{0}$), the anchor is removed entirely from the rendering pipeline, including its position $\mathbf{x}_a$, covariance $\mathbf{A}$, and mask features $\mathbf{m}$ [S1].
4.  **Rendering Application:** Apply the Gaussian-level mask to the opacity $\alpha_{i,k}$ and scale $s_{i,k}$ of each Gaussian during rendering: $m_{i,k} \alpha_{i,k}$ and $m_{i,k} s_{i,k}$ [S1].
5.  **Entropy Loss Calculation:** Include both Gaussian-level and anchor-level masks in the bit consumption calculation for the entropy loss $L_{\text{entropy}}$ [S1].

**Formulas:**
The Gaussian-level mask $m_i$ is defined as [S1]:
$$m_i = \text{sg}(\mathbb{1}[\text{Sig}(\mathbf{f}_i^m) > \epsilon_m] - \text{Sig}(\mathbf{f}_i^m)) + \text{Sig}(\mathbf{f}_i^m) - \mathbf{0}$$
where:
*   $\text{sg}$ is the stop-gradient operator [S1].
*   $\text{Sig}$ is the sigmoid function [S1].
*   $\mathbf{f}_i^m \in \mathbb{R}^K$ is the learnable feature [S1].
*   $\epsilon_m$ is a threshold [S1].
*   $\mathbb{1}[\cdot]$ is the indicator function [S1].
*   A value of 1 in $m$ indicates the corresponding Gaussian offset is valid, while 0 denotes invalid [S1].

The anchor-level mask $m_i^a$ is derived from the average offset mask ratio $\overline{m}_i$ [S1]:
$$m_i^a = \text{sg}(\mathbb{1}[\overline{m}_i > 0] - \overline{m}_i) + \overline{m}_i$$
where:
$$\overline{m}_i = \frac{1}{K} \sum_{k=1}^K m_{i,k}$$
If $\overline{m}_i = 0$, the anchor is pruned [S1].

The entropy loss $L_{\text{entropy}}$ is defined as [S1]:
$$L_{\text{entropy}} = \sum_i^N b_i$$
where $b_i$ is the bit consumption of the $i$-th anchor:
$$b_i = m_i^a \sum_{\mathbf{f} \in \{\mathbf{f}^a, \mathbf{l}\}} \sum_{j=1}^{D^f} \left( -\log_2 p(\hat{\mathbf{f}}_{i,j}) \right) + m_i^a \sum_{\mathbf{f} \in \{\mathbf{o}\}} \sum_{k=1}^K m_{i,k} \sum_{j=1}^3 \left( -\log_2 p(\hat{\mathbf{f}}_{i,3k+j}) \right)$$
where:
*   $N$ is the total number of anchors [S1].
*   $\mathbf{f}^a$ and $\mathbf{l}$ represent anchor attributes and scaling factors [S1].
*   $\mathbf{o}$ represents opacity attributes [S1].
*   $D^f$ is the dimensionality of the features [S1].
*   $p(\hat{\mathbf{f}})$ is the estimated probability of the quantized feature [S1].

**Parameters:**
*   $\lambda$: Trade-off hyperparameter balancing rate and fidelity in the overall loss function [S1].
*   $\epsilon_m$: Threshold for mask generation [S1].

**Edge Cases and Pitfalls:**
*   If all offsets associated with an anchor are pruned, the anchor becomes irrelevant for rendering and must be entirely removed to save parameters [S1].
*   The weight of the mask loss $L_m$ must be manually adjusted across different RD trade-off points as $\lambda$ varies to achieve optimal mask ratios, which can be tedious and difficult to optimize effectively [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_gaussian_mask(f_m, epsilon_m):
    """
    Compute Gaussian-level mask for an anchor.
    Args:
        f_m: Learnable feature vector of size K.
        epsilon_m: Threshold for mask generation.
    Returns:
        m: Binary mask vector of size K.
    """
    sig = sigmoid(f_m)
    indicator = (sig > epsilon_m).astype(float)
    # Straight-through estimator: gradient flows through sig, but value is binary
    m = indicator - sig + sig # The formula simplifies to indicator - sig + sig = indicator if sg is applied correctly in backprop
    # Note: The formula is m = sg(indicator - sig) + sig
    # In practice, this means the forward pass uses 'indicator' (or sig if indicator is 0/1 based on threshold)
    # and the backward pass uses 'sig'.
    # However, the text says: m = sg(1[Sig > eps] - Sig) + Sig
    # Let's implement exactly as written:
    raw = indicator - sig
    m = raw + sig # sg(raw) + sig. Since sg stops gradient on raw, gradient comes from sig.
    # But wait, sg(1[Sig > eps] - Sig) means the gradient of (1[Sig > eps] - Sig) is zero.
    # So the gradient of m w.r.t f_m comes only from the last + Sig term.
    # Forward value: indicator - sig + sig = indicator.
    return indicator

def compute_anchor_mask(m_gaussian):
    """
    Compute anchor-level mask from Gaussian-level masks.
    Args:
        m_gaussian: Binary mask vector of size K.
    Returns:
        m_a: Binary anchor mask.
    """
    avg_m = mean(m_gaussian)
    indicator = (avg_m > 0).astype(float)
    # m_a = sg(1[avg_m > 0] - avg_m) + avg_m
    # Forward value: indicator
    return indicator

def apply_mask_to_rendering(alpha, scale, m_gaussian):
    """
    Apply Gaussian-level mask to opacity and scale.
    Args:
        alpha: Opacity vector of size K.
        scale: Scale vector of size K.
        m_gaussian: Binary mask vector of size K.
    Returns:
        masked_alpha, masked_scale
    """
    return m_gaussian * alpha, m_gaussian * scale

def compute_entropy_loss(anchors, masks):
    """
    Compute entropy loss including mask information.
    Args:
        anchors: List of anchor objects with attributes f^a, l, o.
        masks: Dictionary with anchor masks m_a and Gaussian masks m_g.
    Returns:
        L_entropy: Total entropy loss.
    """
    L_entropy = 0
    for i, anchor in enumerate(anchors):
        m_a = masks['anchor'][i]
        m_g = masks['gaussian'][i]
        
        # Bit consumption for anchor attributes f^a and l
        bits_a = 0
        for f in [anchor.f_a, anchor.l]:
            for j in range(len(f)):
                # p(f_hat) is estimated probability
                bits_a += -log2(anchor.prob_estimate(f[j]))
        
        # Bit consumption for opacity attributes o
        bits_o = 0
        for k in range(len(m_g)):
            if m_g[k] > 0: # Only count valid Gaussians
                for j in range(3):
                    bits_o += -log2(anchor.prob_estimate(anchor.o[3*k+j]))
        
        b_i = m_a * bits_a + m_a * m_g.sum() * bits_o # Note: The formula has m_a outside both sums, and m_g inside the second sum
        # Correction based on formula:
        # b_i = m_a * sum(f^a, l) + m_a * sum(o, k, m_g)
        # The second term is m_a * sum_{f in {o}} sum_{k=1}^K m_{i,k} sum_{j=1}^3 ...
        # So it is m_a * (sum over k of m_g[k] * sum over j of bits)
        
        b_i_corrected = m_a * bits_a + m_a * sum(m_g[k] * sum(-log2(anchor.prob_estimate(anchor.o[3*k+j])) for j in range(3)) for k in range(len(m_g)))
        
        L_entropy += b_i_corrected
    return L_entropy
```

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Not given in the sources.
*   **Strengths:** Allows adaptive mask ratio adjustment according to the rate via backpropagation, eliminating the need for manual tuning of mask loss weights across different RD trade-off points [S1]. Enhances parameter savings by encouraging anchor pruning [S1].
*   **Limitations:** The weight of the mask loss must be manually adjusted across different RD trade-off points as $\lambda$ varies to achieve optimal mask ratios, which can be tedious and difficult to optimize effectively [S1].
*   **When to use:** When compressing 3D Gaussian Splatting models and needing adaptive pruning to balance fidelity and bit consumption [S1].
*   **When not to use:** When manual tuning of hyperparameters is not feasible or when static pruning is sufficient [S1].

## Background
Adaptive Offset Masking was introduced in the paper "HAC++: Towards 100X Compression of 3D Gaussian Splatting" [S1]. It builds on the Gaussian Volume Mask technique [12710-gaussian-volume-mask.md] by integrating the mask into the rate calculation for adaptive updates [S1].

## Sources
[S1] HAC++: Towards 100X Compression of 3D Gaussian Splatting, https://doi.org/10.1109/tpami.2025.3594066, fetched 2026-10-28
