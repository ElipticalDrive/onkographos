# Decomposed Spatiotemporal Representation

Also called: NeRFPlayer Decomposition

## Overview
Decomposed Spatiotemporal Representation is a method for modeling dynamic volumetric scenes by categorizing spatial regions into static, deforming, and new content, assigning each category a specialized neural field to optimize temporal consistency and computational efficiency [S1]. It sets itself apart from the parent DyNeRF method by replacing a single monolithic MLP with a mixture of specialized fields (stationary, deformation, and newness) weighted by a learned decomposition probability, rather than relying on a single temporal embedding [S1, 7274-dynerf.md].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D spatial coordinate $p \in \mathbb{R}^3$ and a scalar time $t \in \mathbb{R}$.
*   **Output:** A density $\sigma \in \mathbb{R}$ and a view-dependent color $c \in \mathbb{R}^3$.

**Data Structures:**
1.  **Decomposition Field ($f$):** Maps $(p, t)$ to a probability distribution over three categories. Implemented as an explicitly cached feature volume $V_f$ and a small MLP decoder $D_f$.
    *   Output: Probabilities $P_{static}, P_{deform}, P_{new}$ such that $\sum P_* = 1$.
2.  **Stationary Field ($s$):** Maps $(p)$ to a feature vector $v_{static}$. Used for static geometry and low-frequency appearance.
    *   Implementation: Explicitly cached features $V_s$ and a tiny MLP.
3.  **Deformation Field ($d$):** Maps $(p, t)$ to a displacement vector $\Delta p$.
    *   Implementation: A small MLP.
4.  **Newness Field ($n$):** Maps $(p, t)$ to a feature vector $v_{new}$.
    *   Implementation: Explicitly cached features $V_n$ and a tiny MLP.
5.  **View-Conditioned Network (Decoder):** Maps the aggregated feature vector $v$ and view direction $d_v$ to density and color.

**Numbered Steps:**
1.  **Query Decomposition:** For a query point $p$ at time $t$, compute the decomposition probabilities $P_{static}(p,t)$, $P_{deform}(p,t)$, and $P_{new}(p,t)$ using the decomposition field $f(p,t)$.
2.  **Compute Expected Feature Vector:**
    *   Retrieve $v_{static}$ from the stationary field $s(p)$.
    *   Retrieve $\Delta p$ from the deformation field $d(p,t)$. Compute the deformed position $p' = p + \Delta p$. Query $v_{deform}$ from the stationary field $s(p')$ (canonical space) [S1].
    *   Retrieve $v_{new}$ from the newness field $n(p,t)$.
    *   Compute the aggregated feature vector:
        $$ v = P_{static} v_{static} + P_{deform} v_{deform} + P_{new} v_{new} $$
3.  **Thresholding (Optimization):** If any probability $P_* < \tau$ (default $\tau = 0.001$), set the corresponding feature vector $v_* = \mathbf{0}$ and skip the field query to save computation [S1].
4.  **Decode Radiance:** Pass the aggregated feature vector $v$ and the view direction $d_v$ to the lightweight view-conditioned network to predict density $\sigma$ and color $c$.

**Formulas:**
*   **Aggregation:** $v = \sum_{* \in \{static, deform, new\}} P_* v_*$ [S1].
*   **Regularization Loss:** To encourage static background and penalize new content (assuming most points are static or deforming):
    $$ L_{reg} = \alpha P_{deform} + P_{new} $$
    where $P_* = \frac{1}{|R_p|} \sum_{p \in R_p} P_*(p)$ is the average probability over the batch $R_p$, and $\alpha$ weights the ratio of deforming to new [S1].
*   **Total Loss:**
    $$ L = L_{rec} + \lambda L_{reg} $$
    where $L_{rec}$ is the reconstruction loss and $\lambda$ is a balancing hyper-parameter [S1].

**Parameters:**
*   $\tau$: Threshold for skipping fields. Default: 0.001 [S1].
*   $\alpha$: Weight for deformation probability in regularization. Value not given in the sources.
*   $\lambda$: Weight for regularization loss. Value not given in the sources.

**Edge Cases and Pitfalls:**
*   **Precision:** The sum of probabilities must be normalized to 1. If the decomposition field outputs unnormalized logits, a softmax is required [G].
*   **Deformation Query:** The deformation field outputs $\Delta p$, which is added to $p$ before querying the *static* field for the deforming component's feature. This implies the deforming component's appearance is anchored to the canonical static geometry [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def decomposed_spatiotemporal_render(p, t, view_dir, model):
    """
    p: (3,) spatial coordinate
    t: scalar time
    view_dir: (3,) view direction
    model: object containing fields f, s, d, n, and decoder
    """
    # 1. Decomposition probabilities
    probs = model.f(p, t) # Returns [P_static, P_deform, P_new]
    P_static, P_deform, P_new = probs
    
    # 2. Feature aggregation
    v_static = model.s(p)
    
    # Deformation: query canonical space
    delta_p = model.d(p, t)
    p_deformed = p + delta_p
    v_deform = model.s(p_deformed)
    
    # Newness
    v_new = model.n(p, t)
    
    # 3. Thresholding
    tau = 0.001
    if P_static < tau:
        v_static = zeros(3) # Assuming 3D feature vector, size depends on implementation
    if P_deform < tau:
        v_deform = zeros(3)
    if P_new < tau:
        v_new = zeros(3)
        
    # 4. Weighted sum
    v = P_static * v_static + P_deform * v_deform + P_new * v_new
    
    # 5. Decode
    sigma, color = model.decoder(v, view_dir)
    
    return sigma, color
```

## Subtopics

### [Dynamic Field Blending](9485-dynamic-field-blending.md)
A technique that explicitly linearly interpolates opacity and premultiplied radiance terms based on learned blending weights, rather than aggregating feature vectors before decoding. It is chosen when adaptive mixing of static and dynamic radiance is preferred over feature-space blending.

## Operating characteristics
*   **Time Complexity:** Rendering time depends on the number of active fields. With thresholding, fields with low probability are skipped, reducing computation for static backgrounds [S1].
*   **Memory Complexity:** Uses explicit feature volumes for static and new fields, which can increase model size compared to pure MLPs, but allows for smaller MLPs for deformation and newness due to specialized representation [S1].
*   **Strengths:** Maintains temporal consistency by separating static, deforming, and new content; saves computation by not modeling static areas with time-dependent fields; handles rigid and non-rigid motion [S1].
*   **Limitations:** Requires careful tuning of regularization parameters ($\alpha, \lambda$); assumes static background exists; new content must be modeled explicitly, which may be inefficient for highly dynamic scenes with no static parts [S1].
*   **When to use:** Dynamic scenes with significant static background and distinct deforming objects.
*   **When not to use:** Scenes with no static elements or highly chaotic dynamics where decomposition is unstable.

## Background
The method was introduced in "NeRFPlayer: A Streamable Dynamic Scene Representation with Decomposed Neural Radiance Fields" by the authors of [S1] in 2023. It builds on the DyNeRF baseline [7274-dynerf.md] by addressing its limitation of using a single MLP for all temporal variations, which can lead to temporal inconsistency and high computational cost for static regions [S1].

## Sources
[S1] NeRFPlayer: A Streamable Dynamic Scene Representation with Decomposed Neural Radiance Fields, https://doi.org/10.1109/tvcg.2023.3247082, fetched 2026-10-26
