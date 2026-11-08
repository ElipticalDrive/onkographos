# Warm-start Encoding

Also called: Temporal Coherency Encoding, Temporal warm-start, Temporally-Coherent Warm-Start Encoder, Warm-start Encoder

## Overview
Warm-start encoding accelerates the training of neural volume representations for animated sparse volumes by initializing the network weights for each frame with the converged weights from the previous frame [S1][S2, unofficial]. It solves the problem of flickering artifacts caused by weight discontinuities in high-frequency details (e.g., thin smoke layers) when using naive cold-start training, while simultaneously reducing encoding time by a factor of 1.5–2.5 compared to independent frame training [S1][S2, unofficial]. Unlike standard cold-start methods, it leverages temporal coherency to preserve smooth transitions across frames [S1][S2, unofficial].

## How it works
**Inputs:** A sequence of animated sparse volumes $V_1, V_2, \dots, V_N$; a target learning rate $\eta_{target}$; a neural network architecture (e.g., the hierarchical regressor in NeuralVDB).
**Outputs:** A sequence of trained neural networks $W_1, W_2, \dots, W_N$, where $W_i$ encodes volume $V_i$.

**Data Structures:**
*   $W_{prev}$: The converged network weights from the previous frame. For the first frame, $W_{prev}$ is initialized randomly or from a pre-trained model [G].
*   $\eta_{current}$: The active learning rate for the current training phase.

**Numbered Steps:**

1.  **Initialize:** Set $W_{prev} \leftarrow \text{RandomInit}()$. Set $\eta_{current} \leftarrow \eta_{target}$.
2.  **Iterate Frames:** For each frame $i$ from 1 to $N$:
    a.  **Warm-Start Initialization:** Initialize the current network weights $W_{curr}$ with $W_{prev}$.
    b.  **First Frame Special Handling:** If $i == 1$:
        i.   Train $W_{curr}$ on $V_1$ using learning rate $\eta_{target}$ until convergence (loss $L_1$ stabilizes).
        ii.  Save $W_1 \leftarrow W_{curr}$.
        iii. Re-process frame 1: Set $\eta_{fine} \leftarrow \eta_{target} / 100$ (or smaller) [S1]. Train $W_1$ on $V_1$ using $\eta_{fine}$ until convergence.
        iv.  Update $\eta_{current} \leftarrow \eta_{fine}$.
    c.  **Subsequent Frames:** If $i > 1$:
        i.   Train $W_{curr}$ on $V_i$ using $\eta_{current}$ for a single pass or until loss $L_i < L_1$ (the final loss of the first frame) [S1].
        ii.  Save $W_i \leftarrow W_{curr}$.
    d.  **Update Previous:** Set $W_{prev} \leftarrow W_i$.
3.  **Domain Decomposition Exception:** If a new domain is added mid-sequence (e.g., via domain decomposition), repeat the "First Frame Special Handling" process for that new domain using $\eta_{target}$ followed by $\eta_{fine}$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def warm_start_encode(volumes, target_lr, network_class):
    """
    volumes: List of sparse volume data structures [V1, V2, ..., VN]
    target_lr: Float, the initial learning rate
    network_class: Class constructor for the neural network
    """
    prev_weights = None
    current_lr = target_lr
    
    # Store final losses for comparison
    first_frame_final_loss = None
    
    for i, volume in enumerate(volumes):
        # 1. Initialize network with previous frame's weights
        if prev_weights is None:
            # [G] Standard random initialization for the very first frame
            net = network_class()
            net.random_init()
        else:
            net = network_class()
            net.set_weights(prev_weights)
            
        if i == 0:
            # 2. First frame: Train with target LR
            net.train(volume, lr=target_lr, epochs=100) # [G] Placeholder for convergence criterion
            loss_1 = net.get_loss()
            
            # 3. First frame: Re-process with smaller LR to fix flickering
            fine_lr = target_lr / 100.0
            net.train(volume, lr=fine_lr, epochs=100) # [G] Placeholder for convergence criterion
            first_frame_final_loss = net.get_loss()
            
            current_lr = fine_lr
            saved_weights = net.get_weights()
        else:
            # 4. Subsequent frames: Train once with reduced LR
            # Stop when loss is lower than first frame's final loss
            net.train(volume, lr=current_lr, stop_loss=first_frame_final_loss)
            saved_weights = net.get_weights()
            
        prev_weights = saved_weights
        
    return [net.get_weights() for net in dummy_list] # [G] Simplified return for illustration
```

**Parameters:**
*   $\eta_{target}$: Target learning rate for initial convergence. Typical value: Not given in the sources.
*   $\eta_{fine}$: Reduced learning rate for re-processing. Value: Up to 100 times smaller than $\eta_{target}$ [S1][S2, unofficial].
*   Stop Condition: Training iterations stop when loss becomes lower than the first frame's final loss [S1].

**Edge Cases and Pitfalls:**
*   **Flickering:** Naive cold-start encoding with a fixed learning rate causes discontinuities in network weights across frames, leading to flickering, especially in high-frequency details like thin smoke layers [S1].
*   **Domain Decomposition:** If new domains are added mid-sequence, the warm-start logic must be reset for that domain, re-running the target LR then fine LR process [S1].

## Subtopics

### [Training Loop Setup](1522-training-loop-setup.md)
Defines the basic iteration structure. Warm-start encoding builds on this by injecting specific weight initialization logic.

### [Training Loop with Adaptive Regularization & LR Decay](2408-training-loop-with-adaptive-regularization-lr-decay.md)
Provides mechanisms for dynamic LR adjustment. Warm-start uses a fixed reduction factor (1/100) rather than adaptive decay.

### [Learning Rate Scheduling](2701-learning-rate-scheduling.md)
General LR strategies. Warm-start is a specific temporal strategy, not a general schedule.

### [Mixed Precision Training](2766-mixed-precision-training.md)
Precision optimization. Independent of warm-start logic.

### [Adam Optimisation with Adaptive Scaling](3093-adam-optimisation-with-adaptive-scaling.md)
Optimizer choice. Warm-start is compatible with Adam but focuses on initialization.

### [Training Schedule](3154-training-schedule.md)
Overall training timeline. Warm-start is a component of the schedule.

### [Training Objective for Fine Representation](4686-training-objective-for-fine-representation.md)
Loss functions. Warm-start helps preserve fine details but doesn't define the loss.

### [Method-Specific Training Configurations](6267-method-specific-training-configurations.md)
General category. Warm-start is a specific configuration for NeuralVDB.

### [Regularization and Priors](6281-regularization-and-priors.md)
Regularization techniques. Warm-start acts as a temporal prior.

### [PixelNeRF](7471-pixelnerf.md)
Alternative view-based method. Warm-start is specific to volume-based neural representations like NeuralVDB.

## Operating characteristics
*   **Time Complexity:** Accelerates encoding by a factor of 1.5–2.5 compared to cold-start training [S1][S2, unofficial].
*   **Memory Complexity:** Requires storing the network weights for each frame if random access is needed, or just the previous frame's weights if sequential processing is sufficient [S1].
*   **Measured Performance:** Tested on NVIDIA RTX A40 GPUs and AMD EPYC 7502 CPU [S1].
*   **Strengths:** Ensures temporal coherence; reduces flickering in high-frequency details; speeds up training.
*   **Limitations:** Sequential dependency (frame $i$ depends on $i-1$); requires careful LR tuning for the fine step.
*   **When to Use:** Animated sparse volumes with high-frequency details (e.g., smoke, fire) where temporal coherence is critical.
*   **When Not to Use:** Static volumes or sequences with no temporal coherence.

## Background
Originated in the NeuralVDB project for encoding time-sequences of animated sparse volumes [S1][S2, unofficial]. It is similar to fine-tuning in transfer learning [Zhou et al. 2017], adapting to a new target without drifting too much from the old one [S2, unofficial]. The method leverages the temporal coherency of input volumes to preserve coherency in output volumes [S1][S2, unofficial].

## Sources
[S1] NeuralVDB: High-resolution Sparse Volume Representation using Hierarchical Neural Networks, https://www.emergentmind.com/papers/2208.04448, fetched 2026-10-27
[S2] NeuralVDB: High-resolution Sparse Volume Representation using Hierarchical Neural Networks | NVIDIA, https://www.youtube.com/watch?v=IfgeduMv_ns, fetched 2026-11-04 (unofficial)
