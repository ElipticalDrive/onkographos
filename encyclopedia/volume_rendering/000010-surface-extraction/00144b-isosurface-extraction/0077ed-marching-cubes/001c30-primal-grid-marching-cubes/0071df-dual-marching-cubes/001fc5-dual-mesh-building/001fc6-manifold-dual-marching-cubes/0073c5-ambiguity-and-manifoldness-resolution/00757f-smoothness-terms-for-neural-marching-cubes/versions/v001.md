# Smoothness Terms for Neural Marching Cubes

Also called: Regularization terms for NMC surface smoothness

## Overview
Smoothness terms for Neural Marching Cubes (NMC) are a family of loss functions designed to regularize the generated surfaces, resolving ambiguities inherent in shapes represented by binary voxels and preventing geometric artifacts [S1]. Unlike sibling methods that resolve topological ambiguities via graph connectivity analysis or dual-node generation, this family operates by penalizing specific geometric properties of the predicted mesh edges during the training phase, thereby guiding the neural network toward smoother, more plausible surface configurations [S1].

## Shared principles
The family shares a common mathematical framework based on edge-level regularization. Let $\mathcal{S}$ be a batch of samples, $E_s$ be the set of edges in the predicted mesh for sample $s$, $F_s^u$ be the predicted position of vertex $u$, and $H_s^u$ be the ground truth position of vertex $u$ [S1]. The general form of the smoothness loss is:

$$ L_{smooth} = \mathbb{E}_{s \sim \mathcal{S}} \sum_{(u,v) \in E_s} \text{Penalty}(u, v) $$

where the penalty function varies by variant. The terms rely on the edge vector difference $F_s^{uv} = F_s^v - F_s^u$ and the ground truth edge vector difference $H_s^{uv} = H_s^v - H_s^u$ [S1]. The notation $[F_s^u]_x$ denotes the $x$-coordinate of $F_s^u$, and $\mathbb{1}(\cdot)$ is the indicator function [S1].

## Subtopics

### No Smoothness Term
*   **Definition:** A baseline configuration where no regularization is applied.
*   **Formula:**
    $$ L_{smooth}^{(1)} = 0 \quad (8) $$
*   **Characteristics:** Serves as the control case. It does not penalize any geometric properties, leaving the surface entirely dependent on the data fidelity terms and other constraints [S1].
*   **When to choose:** Only for ablation studies or when other regularization mechanisms are sufficient.

### Gradient Difference Minimization
*   **Definition:** A term that minimizes the difference between the edge gradients of the predicted mesh and the ground truth mesh.
*   **Formula:**
    $$ L_{smooth}^{(2)} = \mathbb{E}_{s \sim \mathcal{S}} \sum_{(u,v) \in E_s} \| F_s^{uv} - H_s^{uv} \|_2^2 \quad (9) $$
*   **Characteristics:** Directly aligns the predicted edge vectors with the ground truth edge vectors in Euclidean space [S1].
*   **When to choose:** When strict local gradient alignment is prioritized, though it was found to be less effective than other variants in the source study [S1].

### Absolute Angle Preservation (Coordinate-wise)
*   **Definition:** A term designed to preserve the absolute angles of edges by minimizing the cross-product-like components of the predicted and ground truth edge vectors.
*   **Formula:**
    $$ L_{smooth}^{(3)} = \mathbb{E}_{s \sim \mathcal{S}} \sum_{(u,v) \in E_s} (L_y^x + L_z^x + L_x^y + L_z^y + L_x^z + L_y^z) \quad (10) $$
    where
    $$ L_y^x = ( [H_s^{uv}]_x \cdot [F_s^{uv}]_y - [F_s^{uv}]_x \cdot [H_s^{uv}]_y )^2 $$
*   **Characteristics:** Based on the ideal condition $[H^{uv}]_x/|[H^{uv}]_y| = [F^{uv}]_x/|[F^{uv}]_y|$, which implies $[H^{uv}]_x \cdot |[F^{uv}]_y| - [F^{uv}]_x \cdot |[H^{uv}]_y| = 0$. It penalizes deviations from this proportionality for each coordinate pair [S1].
*   **When to choose:** When preserving the directional orientation of edges relative to the ground truth is critical, though it is computationally more complex than simple gradient matching [S1].

### Absolute Angle Preservation (Magnitude-weighted)
*   **Definition:** A variant of the angle preservation term that uses the magnitude of the edge components in the orthogonal plane.
*   **Formula:**
    $$ L_{smooth}^{(4)} = \mathbb{E}_{s \sim \mathcal{S}} \sum_{(u,v) \in E_s} (L_{yz}^x + L_{xz}^y + L_{xy}^z) \quad (11) $$
    where
    $$ L_{yz}^x = ( [H_s^{uv}]_x \cdot [F_s^{uv}]_{yz} - [F_s^{uv}]_x \cdot [H_s^{uv}]_{yz} )^2 $$
    and
    $$ [F_s^{uv}]_{yz} = \sqrt{[F_s^{uv}]_y^2 + [F_s^{uv}]_z^2} $$
*   **Characteristics:** A variant of Eq. (10) that incorporates the magnitude of the orthogonal components, offering a different geometric constraint on edge angles [S1].
*   **When to choose:** As an alternative to Eq. (10) when magnitude-weighted angle constraints are preferred [S1].

### Short Edge Suppression (Adopted Term)
*   **Definition:** The term adopted in the source work, equivalent to Eq. (7) in Section 3.5 of the source. It penalizes ground truth edge vectors for edges that are predicted to be very short.
*   **Formula:**
    $$ L_{smooth}^{(5)} = \mathbb{E}_{s \sim \mathcal{S}} \sum_{(u,v) \in E_s} \| \mathbb{1}(|F_s^{uv}| < \sigma) \cdot H_s^{uv} \|_2^2 \quad (12) $$
*   **Parameters:** $\sigma = 0.0002$ and $\gamma = 10$ (where $\gamma$ likely scales the loss, though the formula shows the indicator threshold $\sigma$) [S1].
*   **Characteristics:** This term effectively suppresses artifacts caused by ambiguous voxel configurations by ensuring that if the network predicts a near-zero length edge (indicating a potential collapse or ambiguity), the ground truth structure is not enforced in a way that creates sharp, noisy features. It was selected based on visual results and quantitative performance [S1].
*   **When to choose:** As the default choice for NMC training to resolve binary voxel ambiguities and ensure smooth surface generation [S1].

## Background
The concept of smoothness terms for Neural Marching Cubes originates from the work on "Neural Marching Cubes" by Lequan Yu, Xianzhi Li, Chi-Wing Fu, Daniel Cohen-Or, and Pheng-Ann Heng, presented at CVPR 2018 [S1]. The method was developed to address the limitations of traditional Marching Cubes when applied to neural network outputs, specifically the artifacts arising from the discrete nature of binary voxel representations. The authors experimented with multiple smoothness formulations (Eqs. 8-12) to find the most effective regularization for surface quality. The study concluded that $L_{smooth}^{(5)}$ provided the best balance of visual fidelity and quantitative performance [S1].

## Sources
[S1] Neural marching cubes, https://doi.org/10.1145/3478513.3480518, fetched 2026-10-26
