# Hybrid 2D/3D Registration Optimization Pipeline

Also called: Net+GradNCC, Two-Stage Similarity Optimization

## Overview
The Hybrid 2D/3D Registration Optimization Pipeline is a two-stage method for estimating the 6 Degrees of Freedom (DoF) pose of a 2D X-ray image relative to a 3D CT volume [S1]. It combines a differentiable network similarity loss for global convergence with a traditional Gradient-based Normalized Cross-Correlation (Grad-NCC) loss for local refinement [S1].

## How it works
The pipeline optimizes pose parameters $\theta$ (rotation and translation) in two sequential stages.

**Inputs:**
1.  Fixed 3D volume $V_{CT}$ (CT scan) [S1].
2.  Moving 2D target image $I_{target}$ (X-ray) [S1].
3.  Initial pose estimate $\theta_0$ (randomly sampled from a distribution, e.g., rotation $N(0, 15^\circ)$, translation $N(0, 30 \text{ mm})$) [S1].
4.  Segmentation mask $V_{Seg}$ used as input to the network forward pass [S1].

**Outputs:**
1.  Optimized pose parameters $\theta^*$.

**Data Structures:**
*   Pose parameters $\theta$: Represented as rotation and translation components [S1].
*   Loss values: Scalar values $L_N$ and $L_{GNNC}$ tracked over iterations to monitor convergence [S1].

**Step 1: Global Optimization (Network Loss)**
1.  Initialize pose parameters $\theta$ [S1].
2.  Generate a Digitally Reconstructed Radiograph (DRR) $I_{DRR}$ from $V_{CT}$ and current $\theta$ using a differentiable renderer (e.g., ProST) [S1].
3.  Compute the network similarity loss $L_N$ between $I_{target}$ and $I_{DRR}$ using $V_{Seg}$ as network input [S1].
4.  Update $\theta$ using Stochastic Gradient Descent (SGD) with momentum 0.9 [S1].
5.  Use a cyclic learning rate schedule oscillating between $1 \times 10^{-6}$ and $1 \times 10^{-4}$ every 100 steps [S1].
6.  Monitor convergence by calculating the standard deviation of $L_N$ over the last 10 iterations, denoted as $std_{L_N}$ [S1].
7.  If $std_{L_N} < 3 \times 10^{-3}$, switch to Step 2. Otherwise, repeat Step 4 [S1].

**Step 2: Local Refinement (Grad-NCC Loss)**
1.  Compute the Gradient-based Normalized Cross-Correlation loss $L_{GNNC}$ between $I_{target}$ and $I_{DRR}$ [S1].
2.  Update $\theta$ using SGD with momentum 0.9 [S1].
3.  Use a cyclic learning rate schedule oscillating between $1 \times 10^{-3}$ and $3 \times 10^{-3}$ [S1].
4.  Monitor convergence by calculating the standard deviation of $L_{GNNC}$ over the last 10 iterations, denoted as $std_{L_{GNNC}}$ [S1].
5.  If $std_{L_{GNNC}} < 1 \times 10^{-5}$, terminate [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def hybrid_2d_3d_registration(V_CT, I_target, theta_init, V_Seg=None):
    theta = theta_init
    L_N_history = []
    L_GNNC_history = []
    velocity = 0.0

    def get_cyclic_lr(iteration, min_lr, max_lr, period):
        return min_lr + (max_lr - min_lr) * ((iteration % period) / period)

    def sgd_step(theta, grad, lr, momentum):
        nonlocal velocity
        velocity = momentum * velocity - lr * grad
        return theta + velocity

    def standard_deviation(history):
        if len(history) < 2: return float('inf')
        mean = sum(history) / len(history)
        return (sum((x - mean) ** 2 for x in history) / len(history)) ** 0.5

    # Stage 1: Global Optimization
    for iteration in range(100000):  # Not given in sources
        # Generate DRR from V_CT and theta using differentiable renderer [S1]
        I_DRR = ... # Not given in sources
        # Compute network similarity loss L_N [S1]
        L_N = ... # Not given in sources
        L_N_history.append(L_N)

        # Compute gradient and update theta with SGD, momentum 0.9 [S1]
        grad_LN = ... # Not given in sources
        lr = get_cyclic_lr(iteration, 1e-6, 1e-4, 100)
        theta = sgd_step(theta, grad_LN, lr, 0.9)

        if len(L_N_history) >= 10:
            if standard_deviation(L_N_history[-10:]) < 3e-3:
                break

    # Stage 2: Local Refinement
    for iteration in range(100000):
        I_DRR = ... # Not given in sources
        # Compute Grad-NCC loss L_GNNC [S1]
        L_GNNC = ... # Not given in sources
        L_GNNC_history.append(L_GNNC)

        grad_LGNNC = ... # Not given in sources
        lr = get_cyclic_lr(iteration, 1e-3, 3e-3, 100)
        theta = sgd_step(theta, grad_LGNNC, lr, 0.9)

        if len(L_GNNC_history) >= 10:
            if standard_deviation(L_GNNC_history[-10:]) < 1e-5:
                break

    return theta
```

**Parameters:**
*   Momentum: 0.9 for both stages [S1].
*   Stage 1 Learning Rate: Cyclic between $1 \times 10^{-6}$ and $1 \times 10^{-4}$, period 100 steps [S1].
*   Stage 2 Learning Rate: Cyclic between $1 \times 10^{-3}$ and $3 \times 10^{-3}$ [S1].
*   Stage 1 Convergence Threshold: $std_{L_N} < 3 \times 10^{-3}$ [S1].
*   Stage 2 Convergence Threshold: $std_{L_{GNNC}} < 1 \times 10^{-5}$ [S1].
*   Batch Size: 2 (for training the network component, if applicable) [S1].

**Edge Cases and Pitfalls:**
*   The switch criterion relies on the standard deviation of the loss over a small window (10 iterations). If the loss oscillates significantly due to learning rate cycling, premature switching may occur [S1].
*   The method assumes the differentiable renderer (ProST) is available and differentiable with respect to pose parameters [S1].

## Subtopics

### [Gradient Descent Registration via ZNCC](9913-gradient-descent-registration-via-zncc.md)
This method performs 6 Degrees of Freedom (DoF) slice-to-volume registration by minimizing the negative Zero-Normalized Cross-Correlation (ZNCC) between a fixed Digitally Reconstructed Radiograph (DRR) and a moving DRR using gradient descent. It serves as the local refinement stage of the Hybrid 2D/3D Registration Optimization Pipeline, leveraging the locally convex loss landscape of ZNCC to converge quickly from random initializations. Unlike the global network-based loss ($L_N$) in its parent pipeline, this method relies on traditional gradient descent with momentum, requiring differentiable DRR generation to compute gradients with respect to pose parameters.

## Operating characteristics
*   **Time Complexity:** The method requires two sequential optimization phases. The first phase uses a differentiable network similarity loss, while the second uses Grad-NCC [S1].
*   **Memory Complexity:** Depends on the memory footprint of the differentiable renderer and the network. The source uses a batch size of 2 [S1].
*   **Measured Performance:** Tested on 150 simulation studies and 100 real X-ray registrations (cadaver specimens) [S1]. The method is less sensitive to local textures compared to traditional image-based methods like Grad-NCC alone [S1].
*   **Strengths:** Combines global convergence of network-based features with local precision of gradient-based correlation. Robust to initial pose errors (rotation $N(0, 15^\circ)$, translation $N(0, 30 \text{ mm})$) [S1].
*   **Limitations:** Requires a trained network for the first stage. The switch criterion is heuristic [S1].
*   **When to use:** When robust global alignment is needed before fine-tuning, especially in the presence of local texture ambiguities.
*   **When not to use:** When computational resources are extremely limited and a single-stage method is preferred, or when the network is not available/trained.

## Background
The method was introduced in the paper "Generalizing Spatial Transformers to Projective Geometry with Applications to 2D/3D Registration" [S1]. It addresses the limitations of traditional image-based methods like Grad-NCC, which are sensitive to local textures, and pure differentiable methods that may get stuck in local minima. The pipeline was tested on simulated data from NIH Cancer CT scans and real X-ray images from cadaver specimens [S1].

## Sources
[S1] Generalizing Spatial Transformers to Projective Geometry with Applications to 2D/3D Registration, https://arxiv.org/abs/2003.10987v1, fetched 2026-10-26
