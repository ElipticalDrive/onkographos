# Mask and Weight Filtering for Patch Points

Also called: $w_j$ and $\eta_j$ filtering, $w_j$ and $η_j$ filtering

## Overview
This method refines the geometric constraints applied to surface patch points during the optimization of a Signed Distance Field (SDF) by filtering outliers and prioritizing critical surface locations. It addresses the problem of noisy or degenerate sampled points on a local surface patch, which can degrade the accuracy of the reconstructed geometry. Unlike the parent method, which samples points around a 3D anchor, this technique specifically modifies the loss calculation by applying a binary mask $w_j$ to exclude outliers and a scalar weight $\eta_j$ to emphasize points with strong gradient alignment to the ground truth normal [S1].

## How it works
**Inputs and Outputs**
*   **Inputs:**
    *   $q$: The 3D anchor point $[x, y, z]$ [S1].
    *   $J$: The number of sampled points on the patch [S1].
    *   $p'_j$: The $j$-th pulled point on the zero level set, derived from $q$ [S1].
    *   $f_\theta$: The SDF function parameterized by $\theta$ [S1].
    *   $e(q)$: The eye position or camera pose associated with point $q$ [S1].
    *   $\nabla f_\theta(p'_j, e(p'_j))$: The gradient of the SDF at the pulled point [S1].
    *   $N^k_{GT}(r)$: The ground truth normal at pixel $r$ corresponding to the view [S1].
*   **Outputs:**
    *   $w_j$: A binary mask value for each point $j \in [1, J]$ [S1].
    *   $\eta_j$: A scalar weight value for each point $j \in [1, J]$ [S1].

**Data Structures**
*   The mask $w$ is a vector of size $J$ with binary values $\{0, 1\}$.
*   The weight $\eta$ is a vector of size $J$ with scalar values in $[0, 1]$ (derived from cosine similarity).

**Steps**
1.  **Pull Points to Zero Level Set:** For each anchor point $q$, compute the pulled point $p'_j$ on the zero level set of the SDF using the signed distance and gradient [S1]:
    $$ p'_j = q - f_\theta(q, e(q)) \frac{\nabla f_\theta(q, e(q))}{\|\nabla f_\theta(q, e(q))\|} $$
    Note: The calculation of $\nabla f_\theta(p'_j, e(p'_j))$ involves second-order derivatives due to the dependency of $p'_j$ on the previous step, which helps the network find better solutions [S1].

2.  **Compute Weight $\eta_j$:** Calculate the cosine similarity between the SDF gradient at the pulled point and the ground truth normal to determine the importance of the point [S1]:
    $$ \eta_j = \cos(\nabla f_\theta(p'_j, e(p'_j)), N^k_{GT}(r)) $$
    This weight focuses the network on points where the gradient is well-aligned with the surface normal [S1].

3.  **Compute Mask $w_j$:** Apply a binary mask to rule out outlier points during the pulling step. The specific thresholding logic for $w_j$ is not explicitly defined in the sources, but it is used to filter out points that do not meet geometric consistency criteria [S1].

4.  **Apply to Loss Function:** Use $w_j$ and $\eta_j$ to modulate the surface fitting loss or other constraints. Points with $w_j=0$ are excluded, and points with higher $\eta_j$ contribute more significantly to the optimization [S1].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_mask_and_weight(q, f_theta, e_q, N_GT, J):
    """
    Computes the mask w_j and weight eta_j for J points sampled around anchor q.
    
    Args:
        q: 3D anchor point [x, y, z].
        f_theta: SDF function.
        e_q: Eye position/camera pose.
        N_GT: Ground truth normal at the corresponding pixel.
        J: Number of points in the patch.
        
    Returns:
        w: Binary mask vector of size J.
        eta: Weight vector of size J.
    """
    # Step 1: Pull points to zero level set
    # Note: In practice, J points are sampled around q. 
    # For simplicity, we assume p'_j is computed for each j.
    # The source formula uses a single q, but the context implies 
    # J points are sampled and pulled. We compute for each j.
    
    w = []
    eta = []
    
    for j in range(J):
        # Sample point p_j around q (logic from parent method)
        p_j = sample_point_around_anchor(q) 
        
        # Compute signed distance and gradient at q
        dist_q = f_theta(q, e_q)
        grad_q = compute_gradient(f_theta, q, e_q)
        
        # Pull to zero level set
        # p'_j = q - f_theta(q, e(q)) * grad_q / ||grad_q||
        # Note: The source formula uses grad at q, but the weight uses grad at p'_j
        norm_grad_q = vector_norm(grad_q)
        if norm_grad_q == 0:
            p_prime_j = q
        else:
            p_prime_j = q - dist_q * (grad_q / norm_grad_q)
            
        # Compute gradient at pulled point (involves second-order derivatives)
        grad_p_prime = compute_gradient(f_theta, p_prime_j, e_q)
        
        # Step 2: Compute weight eta_j
        # eta_j = cos(grad_p_prime, N_GT)
        cos_sim = dot_product(grad_p_prime, N_GT) / (vector_norm(grad_p_prime) * vector_norm(N_GT))
        eta_j = cos_sim
        eta.append(eta_j)
        
        # Step 3: Compute mask w_j
        # The source states w_j rules out outliers but does not give the exact threshold.
        # We mark this as a placeholder for the specific outlier rejection logic.
        # [G] Typically, this might involve checking if the point is within a certain 
        # distance of the zero level set or if the gradient alignment is above a threshold.
        # However, to remain faithful to the source which only says "rule out outliers",
        # we cannot invent the threshold. We assume a binary decision is made.
        # For implementation, one might use a threshold on eta_j or distance.
        # Since the source does not specify, we leave the logic abstract but functional.
        # A common heuristic for "outlier" in this context is low alignment or large distance.
        # We will use a placeholder comment for the exact thresholding rule.
        
        # [G] Implementation note: The source does not specify the threshold for w_j.
        # A reasonable implementation might be w_j = 1 if eta_j > threshold else 0.
        # But since we must not guess, we will assume the mask is provided or 
        # computed via a separate outlier detection step not detailed here.
        # To make the code valid, we will assume a simple threshold for eta_j as a proxy 
        # for "importance" which correlates with outlier status, but this is a gap fill.
        # Actually, the source says w_j rules out outliers AND eta_j focuses on important points.
        # They are distinct. We cannot implement w_j without the rule.
        # We will set w_j to 1 for all points and add a comment that the specific 
        # outlier filtering logic is not provided in the source text.
        w_j = 1.0 # Placeholder: Specific outlier filtering logic not given in source.
        w.append(w_j)
        
    return w, eta
```

**Parameters**
*   $J$: Number of sampled points. The source indicates performance varies with $J$ (Table 5 in [S1]), but the optimal value is not explicitly stated in the notes, only that it affects performance.
*   $\lambda_1 \dots \lambda_6$: Weights for the total loss function $L = L_{RGB} + \lambda_1 L_{DR} + \lambda_2 L_N + \lambda_3 L_{Eik} + \lambda_4 L_{DC} + \lambda_5 L_{NCC} + \lambda_6 L_{Fit}$ [S1]. The mask and weight specifically modulate the surface fitting or normal-related terms.

**Edge Cases and Pitfalls**
*   Removing $w_j$ degrades performance due to unfiltered outliers [S1].
*   Removing $\eta_j$ degrades performance due to lack of focus on critical surface points [S1].
*   The calculation of $\nabla f_\theta(p'_j, e(p'_j))$ involves second-order derivatives, which can be computationally expensive or numerically unstable if not handled carefully [S1].

## Subtopics

### Pulling Mechanism for Zero Level Set Alignment
This variant refers to the specific geometric operation of moving points from a $d$-level set to the zero level set of the SDF. It sets itself apart by explicitly using the SDF gradient and signed distance to enforce surface constraints on the pulled points $p'_j$ rather than the original anchor $q$. It should be chosen when explicit surface alignment is required to improve geometry accuracy. The formula is $p'_j = q - f_\theta(q, e(q)) \frac{\nabla f_\theta(q, e(q))}{\|\nabla f_\theta(q, e(q))\|}$ [S1]. Disabling this mechanism (imposing constraints directly on $q$) results in lower accuracy (F-score 0.753 vs 0.794 on Replica) [S1].

## Operating characteristics
**Time and Memory Complexity**
*   The method adds computational overhead due to the calculation of gradients at pulled points, which involves second-order derivatives [S1].
*   Memory usage increases slightly to store the mask $w$ and weight $\eta$ vectors for each patch.

**Measured Performance**
*   On the Replica dataset, the full method yields Acc=0.037, Comp=0.038, CD-L1=0.037, Prec=0.797, Recall=0.777, F-score=0.794 [S1].
*   Removing $w_j$ ('No $w_j$') yields Acc=0.039, Comp=0.041, CD-L1=0.038, Prec=0.764, Recall=0.790, F-score=0.772 [S1].
*   Removing $\eta_j$ ('No $\eta_j$') yields Acc=0.039, Comp=0.042, CD-L1=0.038, Prec=0.733, Recall=0.772, F-score=0.753 [S1].
*   Disabling pulling ('No pulling') yields Acc=0.039, Comp=0.042, CD-L1=0.038, Prec=0.733, Recall=0.772, F-score=0.753 [S1].

**Strengths and Limitations**
*   **Strengths:** Improves accuracy by filtering outliers and focusing on geometrically consistent points. Enhances the capture of finer geometry details [S1].
*   **Limitations:** Requires the computation of second-order derivatives, which may be costly. The specific thresholding logic for the mask $w_j$ is not provided, making implementation dependent on heuristic choices [S1].

**When to Use**
*   Use when high-fidelity surface reconstruction is required and the SDF is being optimized with multi-view constraints.
*   Use when the sampled points on the patch are prone to outliers or noise.

**When Not to Use**
*   Use with caution if computational resources are limited due to the second-order derivative calculation.
*   Do not use if the ground truth normals are unreliable, as $\eta_j$ depends on $N^k_{GT}(r)$ [S1].

## Background
This method was proposed in the paper "Sensing Surface Patches in Volume Rendering for Inferring Signed Distance Functions" by Ma et al. (2025) [S1]. It builds upon the concept of using SDFs to define local patches around ray-surface intersections, enabling explicit surface constraints in volume rendering. The method leverages the gradient and signed distance characteristics of SDFs to pull points onto the zero level set, enhancing accuracy and capturing finer geometry details [S1]. It is part of a broader effort to infer SDFs from multi-view images via volume rendering.

## Sources
[S1] Sensing Surface Patches in Volume Rendering for Inferring Signed Distance Functions, https://doi.org/10.1609/aaai.v39i4.32426, fetched 2026-10-26
