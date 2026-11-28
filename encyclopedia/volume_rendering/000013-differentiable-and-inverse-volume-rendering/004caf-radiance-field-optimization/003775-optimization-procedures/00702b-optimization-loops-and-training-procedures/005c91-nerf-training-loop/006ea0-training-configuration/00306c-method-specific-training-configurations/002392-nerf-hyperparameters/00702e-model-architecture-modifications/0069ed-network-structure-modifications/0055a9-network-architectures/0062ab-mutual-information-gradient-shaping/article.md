# Mutual Information Gradient Shaping

Also called: J-NeRF, MI-shaping NeRF, NeRF shaping with mutual information gradients, mutual information shaped NeRF

## Overview
Mutual Information Gradient Shaping (MIGS), implemented as JacobiNeRF, is a training regularization technique that shapes the tangent space of a Neural Radiance Field (NeRF) to encode semantic correlations between scene entities. Unlike first-order approaches that distill 2D features into 3D radiance channels or treat labels as separate output channels, MIGS aligns the Jacobians of correlated entities to maximize their mutual information under random weight perturbations, enabling efficient sparse label propagation and semantic consensus.

## How it works
**Inputs and Outputs**
*   **Inputs:** A standard NeRF model $\Phi$ parameterized by weights $\theta \in \mathbb{R}^D$; a set of labeled source pixels/points $S = \{s_1, \dots, s_K\}$ with semantic labels; unlabeled target pixels/points $T = \{t_1, \dots, t_M\}$; camera parameters for rendering.
*   **Outputs:** A trained NeRF $\Phi$ where the Jacobian of rendered values with respect to weights encodes semantic similarity; propagated semantic labels for target entities.

**Data Structures**
*   **NeRF Weights ($\theta$):** Vector of size $D$.
*   **Jacobian ($\partial \Phi$):** Gradient vector of the same size $D$, representing $\frac{\partial I(p)}{\partial \theta}$ for a pixel/point $p$.
*   **Perturbation ($n$):** Vector of size $D$, sampled from a uniform distribution on the unit sphere $S^{D-1}$.

**Method Steps**

1.  **Compute Jacobians:** For each labeled source pixel $s_k$ and target pixel $t_i$, compute the Jacobian of the rendered value with respect to the NeRF weights $\theta$ at the current training state. Let $\partial \Phi^{J,s_k}$ and $\partial \Phi^{J,t_i}$ denote these gradients [S1].

2.  **MI-Shaping Loss Calculation:** Align the Jacobians of correlated entities using contrastive learning. The Mutual Information (MI) between two entities is approximated by the cosine similarity of their Jacobians [S1].
    *   Define the MI approximation:
        $$ I(I(p_i), I(p_j)) \propto \cos(\partial \Phi_i, \partial \Phi_j) = \frac{\partial \Phi_i \cdot \partial \Phi_j}{\|\partial \Phi_i\| \|\partial \Phi_j\|} $$
        where $\alpha$ is the angle between Jacobians, and $I(I(p_i), I(p_j)) = \log(1/\sqrt{1 - \cos^2 \alpha}) + \text{const}$ [S1].
    *   Construct positive pairs (entities known to be correlated, e.g., same semantic class) and negative pairs (different classes).
    *   Compute the MI-Shaping Loss ($L_{MIG}$) using InfoNCE [S1]:
        $$ L_{MIG} = -\log \frac{\exp(\|\cos(\partial \Phi_i, \partial \Phi_{i+})\|/\tau)}{\sum_{j \in \{i^+\} \cup \{i^-\}} \exp(\|\cos(\partial \Phi_i, \partial \Phi_j)\|/\tau)} $$
        where $\tau$ is a temperature parameter, $i+$ denotes positive pairs, and $i-$ denotes negative pairs. The notation $\|\cos(\cdot)\|$ is used to handle the absolute cosine similarity as per the source's convention for magnitude [S1].

3.  **Gradient Normalization:** Apply a regularization term to compact gradients onto a unit sphere, improving training efficiency and facilitating label propagation [S1]:
    $$ L_{norm} = \gamma (1.0 - \|\partial \Phi_i\|)^2 $$
    where $\gamma$ is a weighting factor.

4.  **Total Loss:** Combine the standard photometric reconstruction loss ($L_{NeRF}$) with the shaping terms [S1]:
    $$ L_{TM} = L_{NeRF} + \lambda L_{MIG} + L_{norm} $$
    where $\lambda$ controls the strength of the mutual information shaping.

5.  **Label Propagation (Inference/Post-Training):**
    *   For a target pixel $t_i$, select the best source pixel $k^*$ based on Jacobian alignment:
        $$ k^* = \arg\max_k \frac{\|\partial \Phi^{J,s_k} \cdot \partial \Phi^{J,t_i}\|}{\|\partial \Phi^{J,s_k}\| \|\partial \Phi^{J,t_i}\|} $$
        [S1].
    *   Alternatively, propagate labels via perturbation response:
        *   Perturb weights: $\theta_D' = \theta + \sigma n$, where $n \sim \text{Uniform}(S^{D-1})$ [S1].
        *   Render perturbed image $I_k$ using weights perturbed along the gradient of source pixel $k$ (i.e., $\theta + \sigma \partial \Phi^{J,s_k}$) [S1].
        *   Compute response: $R_k(t_i) = |I_k(t_i) - I(t_i)|$ [S1].
        *   Assign label: $l_{t_i} = \arg\max_k R_k(t_i)$, treating $R_k$ as logits for $K$-way classification [S1].

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def compute_jacobian(nerf_model, pixel_coords, camera_params, weights):
    """
    Computes the gradient of the rendered pixel value with respect to network weights.
    Returns a vector of size D (number of weights).
    """
    # Use automatic differentiation to compute d(rendered_value) / d(weights)
    # rendered_value = nerf_model(pixel_coords, camera_params, weights)
    return grad(rendered_value, weights)

def cosine_similarity(vec1, vec2):
    """Computes cosine similarity between two vectors."""
    dot_product = np.dot(vec1, vec2)
    norm1 = np.linalg.norm(vec1)
    norm2 = np.linalg.norm(vec2)
    if norm1 == 0 or norm2 == 0:
        return 0.0
    return dot_product / (norm1 * norm2)

def train_jacobi_nerf(nerf_model, data_loader, labeled_sources, unlabeled_targets, 
                      lambda_mi, gamma_norm, tau, num_epochs):
    """
    Trains NeRF with Mutual Information Gradient Shaping.
    """
    for epoch in range(num_epochs):
        for batch in data_loader:
            # 1. Standard NeRF forward pass and photometric loss
            rendered_images = nerf_model(batch.rays)
            l_nerf = photometric_loss(rendered_images, batch.gt_images)
            
            # 2. Compute Jacobians for labeled sources and targets
            jacobians_s = {}
            for s in labeled_sources:
                jacobians_s[s] = compute_jacobian(nerf_model, s.coords, s.camera, nerf_model.weights)
            
            jacobians_t = {}
            for t in unlabeled_targets:
                jacobians_t[t] = compute_jacobian(nerf_model, t.coords, t.camera, nerf_model.weights)
            
            # 3. Compute MI-Shaping Loss (L_MIG)
            # Assume we have defined positive pairs (s, t) based on ground truth or self-supervised features
            l_mig = 0.0
            count = 0
            for t in unlabeled_targets:
                for s_pos in labeled_sources: # Positive pair
                    cos_sim_pos = cosine_similarity(jacobians_s[s_pos], jacobians_t[t])
                    exp_pos = np.exp(np.abs(cos_sim_pos) / tau)
                    
                    sum_exp = exp_pos
                    for s_neg in labeled_sources: # Negative pairs
                        if s_neg != s_pos:
                            cos_sim_neg = cosine_similarity(jacobians_s[s_neg], jacobians_t[t])
                            sum_exp += np.exp(np.abs(cos_sim_neg) / tau)
                    
                    l_mig -= np.log(exp_pos / sum_exp)
                    count += 1
            
            if count > 0:
                l_mig /= count
            
            # 4. Compute Normalization Loss (L_norm)
            # Regularize the norm of target Jacobians to be close to 1.0
            l_norm = 0.0
            for t in unlabeled_targets:
                norm_t = np.linalg.norm(jacobians_t[t])
                l_norm += (1.0 - norm_t)**2
            l_norm /= len(unlabeled_targets)
            
            # 5. Total Loss
            total_loss = l_nerf + lambda_mi * l_mig + gamma_norm * l_norm
            
            # 6. Backpropagation and update
            nerf_model.backward_and_update(total_loss)

def propagate_labels_jacobi_nerf(nerf_model, labeled_sources, target_pixels, sigma=0.01):
    """
    Propagates semantic labels from labeled sources to target pixels using perturbation response.
    """
    propagated_labels = {}
    
    # Pre-compute perturbed models for each source
    # Perturb weights along the gradient of the source pixel
    perturbed_models = {}
    for s in labeled_sources:
        grad_s = compute_jacobian(nerf_model, s.coords, s.camera, nerf_model.weights)
        # Create a temporary model with perturbed weights
        # theta_perturbed = theta + sigma * grad_s
        perturbed_models[s] = nerf_model.clone()
        perturbed_models[s].weights += sigma * grad_s
        
    for t in target_pixels:
        responses = []
        for s in labeled_sources:
            # Render target pixel with model perturbed along source s's gradient
            # I_k(t) = Phi(o_t, v_t; theta + sigma * grad_s)
            rendered_val = perturbed_models[s].render(t.coords, t.camera)
            original_val = nerf_model.render(t.coords, t.camera)
            
            # Response is absolute difference
            r_k = np.abs(rendered_val - original_val)
            responses.append(r_k)
            
        # Label is the argmax of responses (treating responses as logits)
        best_source_idx = np.argmax(responses)
        best_source = labeled_sources[best_source_idx]
        propagated_labels[t] = best_source.label
        
    return propagated_labels
```

**Parameters**
*   $\lambda$ (lambda): Weight for $L_{MIG}$. Not given in the sources.
*   $\gamma$ (gamma): Weight for gradient normalization. Not given in the sources.
*   $\tau$ (tau): Temperature parameter for InfoNCE loss. Not given in the sources.
*   $\sigma$: Scale for weight perturbation. Not given in the sources.

**Edge Cases and Pitfalls**
*   **Precision:** The Jacobian computation requires accurate gradients with respect to all network weights. Numerical instability may occur if weights are large; the normalization term helps mitigate this [S1].
*   **Boundaries:** The method assumes the NeRF is differentiable with respect to its weights. Discontinuities in the scene representation (e.g., hard cutoffs) may break gradient flow [S1].
*   **Degenerate Input:** If labeled sources are not semantically distinct or if the scene has no correlated entities, the contrastive loss may not provide meaningful gradients [S1].

## Subtopics

### J-NeRF 2D
Propagates labels in the 2D image domain by computing perturbation differences after volume rendering. Suitable for pixel-wise semantic segmentation [S1].

### J-NeRF 3D
Propagates labels in 3D space by estimating perturbation differences for 3D points along rays before volume rendering. Suitable for instance segmentation and 3D entity selection [S1].

## Operating characteristics
*   **Time Complexity:** Training requires computing Jacobians for all labeled and target pixels, which is $O(N \cdot D)$ where $N$ is the number of pixels/points and $D$ is the number of weights. Label propagation requires rendering with $K$ perturbed models, costing $O(K \cdot M \cdot R)$ where $M$ is the number of targets and $R$ is the number of samples per ray [S1].
*   **Memory Complexity:** Requires storing Jacobians for all entities, $O(N \cdot D)$, and potentially $K$ copies of the NeRF weights for perturbation-based propagation, $O(K \cdot D)$ [S1].
*   **Strengths:** Enables efficient label propagation from extremely sparse annotations; encodes semantic consensus in the tangent space rather than output channels; supports view-invariant propagation [S1].
*   **Limitations:** Requires computing second-order information (Jacobians), which is more expensive than first-order methods; performance depends on the quality of self-supervised features used for contrastive pairs [S1].
*   **When to use:** Sparse label propagation for semantic/instance segmentation; entity selection by clicking a single point; appearance edit propagation [S1].
*   **When not to use:** When computational resources are extremely limited and only photometric reconstruction is needed; when dense annotations are available and first-order methods suffice [S1].

## Background
Mutual Information Gradient Shaping was introduced by Xiaomeng Xu, Yanchao Yang, Kaichun Mo, Boxiao Pan, Li Yi, and Leonidas Guibas in "JacobiNeRF: NeRF Shaping with Mutual Information Gradients" (arXiv:2304.00341v1, April 2023) [S1]. The method addresses the limitation of traditional NeRFs that only capture appearance, by explicitly regularizing learning dynamics to align Jacobians of correlated entities, thereby encoding mutual information in the NeRF tangent space [S1]. It is used for sparse label propagation, entity selection, and scene modifications [S1].

## Sources
[S1] JacobiNeRF: NeRF Shaping with Mutual Information Gradients, https://doi.org/10.1109/cvpr52729.2023.01583, fetched 2026-10-29
