# Probabilistic Directed Distance Field (PDDF) Training

Also called: DDF Learning and Loss Functions, PDDF Optimization, Probabilistic Directed Distance Field (PDDF), View Consistency Inequality

## Overview
Probabilistic Directed Distance Field (PDDF) Training is a method for learning implicit surface representations from oriented point clouds by supervising a neural network to output a probability distribution over depths rather than a single scalar value. It solves the problem of representing discontinuous directed distance fields (DDFs) with a continuous ($C^1$) network, enabling accurate modeling of surface occlusions and visibility transitions. It distinguishes itself from standard SDF training by explicitly modeling depth as a discrete mixture of delta functions and enforcing view consistency inequalities to ensure geometric coherence across different viewing directions.

## How it works
**Inputs and Outputs:**
*   **Input:** A mesh defining the target surface $S$ and bounding volume $B$. The network takes an oriented point $\tau = (p, v)$ as input, where $p \in B$ is a 3D position and $v \in S^2$ is a viewing direction.
*   **Output:** The network predicts a visibility scalar $\xi_b(p, v) \in [0, 1]$ and a probabilistic depth distribution $P_{p,v}(d)$. The distribution is a mixture of $K$ Dirac delta functions:
    $$ P_{p,v}(d) = \sum_{i=1}^K w_i(p, v) \delta(d - d_i(p, v)) $$
    where $w_i$ are mixture weights ($\sum w_i = 1$) and $d_i$ are delta locations. The final predicted depth is $d_{i^*}$ where $i^* = \arg\max_i w_i$.

**Data Structures:**
The network outputs $K+1$ values per oriented point:
1.  $\xi_b$: Visibility prediction (scalar).
2.  $d_i$: Depth component $i$ (scalar), for $i=1 \dots K$.
3.  $w_i$: Weight component $i$ (scalar), for $i=1 \dots K$. Note that $w_K$ is often derived or constrained such that $\sum w_i = 1$.

**Numbered Steps:**

1.  **Data Extraction:** Generate six types of samples $(p, v)$ from the mesh to train the network [S1]:
    *   **(U) Uniform:** Random $(p, v)$ within the bounding volume and sphere.
    *   **(A) At-surface:** Samples where ground truth visibility $\xi(p, v) = 1$.
    *   **(B) Bounding:** Samples where $p \in \partial B$ (boundary of bounding volume) and $v$ points interior.
    *   **(S) Surface:** Samples where $p \in S$ (on the surface) and $v$ is uniformly distributed on the sphere $U[S^2]$.
    *   **(T) Tangent:** Samples where $v$ lies in the tangent space of the surface point $q(p, v) \in S$.
    *   **(O) Offset:** Samples where $p$ is offset from (T) samples along the surface normal $n(p, v)$ by a small value.

2.  **View Consistency Inequality Enforcement:** Ensure that if a surface point $q_1$ is visible from $\tau_1 = (p_1, v_1)$ (i.e., $q_1 = p_1 + d(\tau_1)v_1 \in S$) and $q_1$ also lies on the ray from $\tau_2 = (p_2, v_2)$ (i.e., $\exists t > 0$ such that $p_2 + t v_2 = q_1$), then:
    *   (i) $\xi(\tau_2) = 1$ (the point is visible from $\tau_2$).
    *   (ii) $d(\tau_2) \le ||p_2 - q_1||_2 = t$ (the depth from $\tau_2$ is no greater than the distance to the known surface point) [S1].

3.  **Loss Computation:** Compute the following loss terms per oriented point [S1]:
    *   **Minimum Distance Loss:** Trains the highest probability depth component $i^* = \arg\max_j w_j$:
        $$ \mathcal{L}_d = \xi |d_{b i^*} - d| $$
        where $d$ is the ground truth depth.
    *   **Visibility Loss:** Binary cross-entropy between ground truth $\xi$ and prediction $\xi_b$:
        $$ \mathcal{L}_\xi = \text{BCE}(\xi, \xi_b) $$
    *   **First-Order Normals Loss:** Matches surface normals derived from $\nabla_p d_{b i^*}$ to ground truth normals $n$:
        $$ \mathcal{L}_n = -\xi |n^T n_{b i^*}(p, v)| $$
    *   **Directed Eikonal Regularization:** Applied to visibility and each delta component $d_i$:
        $$ \mathcal{L}_{\text{DE}} = \gamma_{\text{E},d} \sum_{i=1}^K \xi [\nabla_i \widehat{d}_i v + 1]^2 + \gamma_{\text{E},\xi} [\nabla_\xi \widehat{\xi} v]^2 $$
        where $\nabla_i$ denotes the gradient with respect to $p$ for component $i$.
    *   **Weight Field Regularization:** Encourages low entropy (preventing unnecessary switching of $i^*$) and transition at discontinuities:
        $$ \mathcal{L}_W = \gamma_V \mathcal{L}_V + \gamma_T \mathcal{L}_T $$
        (Specific forms of $\mathcal{L}_V$ and $\mathcal{L}_T$ are not detailed in the provided text, but they regularize the weight field $w$) [S1].

4.  **Optimization:** Minimize the total loss $\mathcal{L} = \mathcal{L}_d + \mathcal{L}_\xi + \mathcal{L}_n + \mathcal{L}_{\text{DE}} + \mathcal{L}_W$ using gradient descent.

**Parameters:**
*   **K:** Number of Dirac delta components. Typical value is $K=2$ to represent discontinuities without sacrificing efficiency [S1].
*   **$\gamma_{\text{E},d}, \gamma_{\text{E},\xi}$:** Eikonal regularization weights.
*   **$\gamma_V, \gamma_T$:** Weight field regularization weights.

**Edge Cases and Pitfalls:**
*   **Discontinuities:** Standard DDFs are discontinuous at surface collisions and occlusions. PDDF handles this by allowing the weight field $w(p, v)$ to smoothly transition from one index $i^*$ to another $j$ at discontinuities, causing the selected delta component $d_{i^*}$ to jump to $d_j$, creating the desired discontinuity in $d$ while keeping the network $C^1$ [S1].
*   **Precision:** The use of $\arg\max$ for $i^*$ introduces a non-differentiable step. The smooth transition of weights ensures that the *distribution* is differentiable, but the final depth selection is discrete. The loss functions are designed to train the components such that the correct component has the highest weight.

## Subtopics

### [PDDF Shape-Fitting Loss](9353-pddf-shape-fitting-loss.md)
The PDDF Shape-Fitting Loss ($\mathcal{L}_S$) is the primary training objective for Probabilistic Directed Distance Fields, designed to supervise the network's prediction of visibility, surface normals, and a mixture of depth components. It solves the problem of learning accurate implicit surface representations from oriented point clouds by combining geometric consistency, visibility prediction, and regularization of the probabilistic weight fields. It distinguishes itself from standard SDF training by explicitly modeling depth as a discrete mixture of delta functions (PDDF) and enforcing transitions between these components at surface discontinuities via a dedicated weight transition loss.

## Operating characteristics
*   **Time Complexity:** Rendering is a single forward pass per pixel, independent of mesh complexity [S1]. Training requires generating samples and computing gradients for all loss terms.
*   **Memory Complexity:** The network stores weights for $K$ depth components and visibility. For $K=2$, this is minimal overhead over a standard DDF.
*   **Strengths:** Handles discontinuities (occlusions, self-occlusions) naturally via the probabilistic mixture. View consistency is enforced. Efficient rendering.
*   **Limitations:** Requires careful tuning of weight field regularization to prevent unnecessary switching of $i^*$. The discrete selection of $i^*$ is non-differentiable, though the underlying distribution is smooth.
*   **When to use:** When learning implicit surfaces from oriented point clouds or meshes where occlusions and visibility changes are significant.
*   **When not to use:** For simple, convex, or non-occluding shapes where a standard SDF or DDF suffices.

## Background
Probabilistic Directed Distance Fields were introduced to address the limitations of standard Directed Distance Fields (DDFs) in representing discontinuous fields with continuous neural networks. The method builds on the properties of DDFs, specifically the View Consistency Inequality, which ensures geometric coherence across viewing directions. The key innovation is the modification of the depth field $d$ to output a probability distribution over depths, allowing a $C^1$ network to represent discontinuous fields [S1]. This approach is particularly useful for differentiable rendering and shape reconstruction from oriented point clouds.

## Sources
[S1] Probabilistic Directed Distance Fields for Ray-Based Shape Representations, https://doi.org/10.1109/tpami.2025.3594225, fetched 2026-10-29
