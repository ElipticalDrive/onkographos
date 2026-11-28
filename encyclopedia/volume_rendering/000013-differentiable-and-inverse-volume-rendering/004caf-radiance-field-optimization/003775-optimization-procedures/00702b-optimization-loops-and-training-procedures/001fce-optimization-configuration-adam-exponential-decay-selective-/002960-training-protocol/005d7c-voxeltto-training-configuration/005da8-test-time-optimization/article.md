# Test-Time Optimization

Also called: TTO for VFMs, Test-Time Optimization (TTO) for VFMs

## Overview
Test-Time Optimization (TTO) is a technique that exploits known camera parameters while keeping the pretrained visual foundation model (VFM) parameters frozen, adapting lightweight Low-Rank Adaptation (LoRA) modules using pose supervision [S1, unofficial]. It addresses misalignment in novel view synthesis by aligning predicted camera parameters with ground truth during inference [S1, unofficial]. Unlike standard inference which uses fixed weights, TTO performs a brief optimization loop on test data to refine the model's internal representation of camera geometry without retraining the backbone [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Inputs:** Pretrained VFM backbone (frozen), LoRA modules (trainable), input image(s), initial predicted camera poses ($R_{pi}, t_i, k$), and ground-truth camera poses ($R_{gi}, t_{gi}, k_{gi}$) if available for supervision [S1, unofficial].
*   **Outputs:** Optimized LoRA parameters and refined camera parameters [S1, unofficial].

**Data Structures**
The network architecture follows DA3 with $L=40$ attention layers [S1, unofficial].
*   **Layers 1–12:** Single-view attention [S1, unofficial].
*   **Layers 13–40:** Alternate between single-view and global attention [S1, unofficial].
*   **LoRA Layout:** LoRA is added to the Query (Q), Key (K), and Value (V) projections in layers 13–40 [S1, unofficial]. Each layer shares a down-projection matrix $A$, while up-projection matrices $B$ are independent per layer [S1, unofficial].

**Steps**
1.  **Initialization:** Insert LoRA modules into the VFM backbone at layers 13–40 [S1, unofficial].
2.  **Pose Alignment (Pre-computation):**
    *   Fix the first ground-truth camera pose to the identity matrix to establish a common coordinate frame [S1, unofficial].
    *   Address scale ambiguity by normalizing translation vectors [S1, unofficial]. Compute the mean scale $s$ for predicted translations $t_i$ and ground-truth translations $t_{gi}$ separately:
        $$ s = \frac{1}{M-1} \sum_{i=1}^M \|t_i\|_2 $$
        where $M$ is the number of frames in the sequence [S1, unofficial].
3.  **Loss Calculation:**
    *   **Rotation Loss ($L_{rot}$):** Constructed from the relative rotation between predicted $R_{pi}$ and ground-truth $R_{gi}$ rotations [S1, unofficial]:
        $$ L_{rot} = \frac{1}{2M} \sum_{i=1}^M (3 - \text{tr}(R_{pi}^\top R_{gi})) $$
    *   **Translation Loss ($L_{trans}$):** Calculate the scale-normalized translation loss using SmoothL1 between predicted and ground-truth translations [S1, unofficial].
    *   **Intrinsic Loss ($L_{intrinsics}$):** Supervise camera intrinsics $k = (f_x, f_y, c_x, c_y)$ by directly applying SmoothL1 to predicted and ground-truth values [S1, unofficial].
4.  **Optimization Loop:**
    *   Compute the total TTO loss [S1, unofficial]:
        $$ L_{TTO} = \lambda_{pose} (L_{rot} + w_t L_{trans}) + \lambda_K L_{intrinsics} $$
    *   Update LoRA parameters by backpropagation for a fixed number of iterations [S1, unofficial].

**Parameters**
*   $\lambda_{pose}$: Weight for pose loss. Not given in the sources.
*   $w_t$: Weight for translation loss. Not given in the sources.
*   $\lambda_K$: Weight for intrinsic loss. Not given in the sources.
*   Number of iterations: Fixed, but exact value not given in the sources [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def test_time_optimization(vfm_backbone, lora_modules, images, 
                           predicted_poses, ground_truth_poses, 
                           lambda_pose, w_t, lambda_K, num_iterations):
    # Step 1: Align coordinate frames
    # Fix first ground-truth pose to identity matrix
    ground_truth_poses.R_gi[0] = [[1.0 if i == j else 0.0 for j in range(3)] for i in range(3)]
    
    # Step 2: Normalize scales to address ambiguity
    M = len(predicted_poses.t_i)
    # Compute mean scale for predicted translations
    scale_pred = sum(norm(t) for t in predicted_poses.t_i) / (M - 1)
    # Compute mean scale for ground-truth translations
    scale_gt = sum(norm(t) for t in ground_truth_poses.t_gi) / (M - 1)
    # Normalize translations
    t_i_norm = [t / scale_pred for t in predicted_poses.t_i]
    t_gi_norm = [t / scale_gt for t in ground_truth_poses.t_gi]
    
    optimizer_state = {}
    
    for step in range(1, num_iterations + 1):
        # Zero gradients
        for p in lora_modules.parameters(): p.grad = None
        
        # Forward pass through VFM with LoRA
        # Note: The exact forward pass mechanism for camera decoding is not detailed in the sources [S1, unofficial].
        predicted_poses = vfm_forward(vfm_backbone, lora_modules, images)
        
        # Calculate Rotation Loss: L_rot = 1/(2M) * sum(3 - tr(R_pi^T @ R_gi))
        rot_loss = 0.0
        for i in range(M):
            R_pi = predicted_poses.R_pi[i]
            R_gi = ground_truth_poses.R_gi[i]
            R_pi_T_R_gi = matmul(transpose(R_pi), R_gi)
            rot_loss += 3.0 - trace(R_pi_T_R_gi)
        rot_loss /= (2.0 * M)
        
        # Calculate Translation Loss: SmoothL1 on normalized translations
        trans_loss = 0.0
        for i in range(M):
            trans_loss += smooth_l1_loss(t_i_norm[i], t_gi_norm[i])
        trans_loss /= M
        
        # Calculate Intrinsic Loss: SmoothL1 on intrinsics
        intrinsic_loss = smooth_l1_loss(predicted_poses.k, ground_truth_poses.k)
        
        # Total TTO Loss: L_TTO = lambda_pose * (L_rot + w_t * L_trans) + lambda_K * L_intrinsics
        loss = lambda_pose * (rot_loss + w_t * trans_loss) + lambda_K * intrinsic_loss
        
        # Backpropagation and parameter update
        loss.backward()
        adam_step(lora_modules.parameters(), [p.grad for p in lora_modules.parameters()], optimizer_state, step)
        
    return lora_modules

def norm(v): return sum(x**2 for x in v)**0.5
def transpose(M): return [[M[j][i] for j in range(len(M))] for i in range(len(M[0]))]
def matmul(A, B): return [[sum(A[i][k]*B[k][j] for k in range(len(B))) for j in range(len(B[0]))] for i in range(len(A))]
def trace(M): return sum(M[i][i] for i in range(len(M)))
def smooth_l1_loss(pred, target):
    diff = [p - t for p, t in zip(pred, target)]
    return sum(0.5 * x**2 if abs(x) < 1.0 else abs(x) - 0.5 for x in diff) / len(diff)
def adam_step(params, grads, state, t, lr=1e-3, beta1=0.9, beta2=0.999, eps=1e-8):
    for p, g in zip(params, grads):
        if p not in state: state[p] = {'m': [0.0]*len(p), 'v': [0.0]*len(p)}
        m, v = state[p]['m'], state[p]['v']
        for i in range(len(p)):
            m[i] = beta1 * m[i] + (1 - beta1) * g[i]
            v[i] = beta2 * v[i] + (1 - beta2) * g[i]**2
            m_hat = m[i] / (1 - beta1**t)
            v_hat = v[i] / (1 - beta2**t)
            p[i] -= lr * m_hat / (v_hat**0.5 + eps)
```

**Edge Cases and Pitfalls**
*   **Scale Ambiguity:** Image-based reconstruction has inherent scale ambiguity [S1, unofficial]. Failure to normalize translations by their mean scale $s$ will result in incorrect loss values [S1, unofficial].
*   **Coordinate Frame:** The first ground-truth pose must be fixed to the identity matrix to align the predicted and ground-truth coordinate systems before computing relative rotation [S1, unofficial].
*   **LoRA Placement:** LoRA is strictly applied only to layers 13–40 [S1, unofficial].

## Subtopics

## Operating characteristics
*   **Time Complexity:** Depends on the number of optimization iterations, which is fixed but unspecified [S1, unofficial]. The full model training requires 80 GPU hours, while test-time optimization is described as lightweight and brief [S1, unofficial].
*   **Memory Complexity:** Adds the memory overhead of LoRA parameters, with the full VoxelTTO framework introducing 75.3M trainable parameters [S1, unofficial].
*   **Strengths:** Improves geometric fidelity and camera-pose estimation for novel view synthesis (NVS) [S1, unofficial]. Compatible with any VGGT-style VFMs [S1, unofficial].
*   **Limitations:** Requires ground-truth camera poses for supervision during the optimization loop [S1, unofficial]. Performance depends on the quality of the initial predicted poses [G].
*   **When to use:** When known camera parameters are available and alignment between predicted and ground-truth poses is critical for high-quality NVS [S1, unofficial].
*   **When not to use:** When ground-truth poses are unavailable or when computational resources for iterative optimization are limited [G].

## Background
Test-Time Optimization (TTO) was introduced in the VoxelTTO paper [S1, unofficial]. It builds upon the DA3 architecture and VGGT-style visual foundation models [S1, unofficial]. The method was developed to address misalignment issues in novel view synthesis, particularly when injecting camera parameters into the model [S1, unofficial]. It replaces vanilla 3DGS rasterization with stochastic solid volume rendering to improve geometric supervision [S1, unofficial].

## Sources
[S1] VoxelTTO: Voxel-Aligned Feed-Forward 3D Gaussian Splatting with Test-Time Optimization, https://doi.org/10.48550/arxiv.2609.21498, fetched 2026-10-26 (unofficial)
