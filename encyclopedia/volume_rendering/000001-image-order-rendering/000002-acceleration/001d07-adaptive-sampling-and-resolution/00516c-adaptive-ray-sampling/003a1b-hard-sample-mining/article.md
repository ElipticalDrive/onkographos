# Hard Sample Mining

Also called: HSM

## Overview
Hard Sample Mining (HSM) is a training acceleration technique for neural radiance fields that reduces iteration time and memory usage by subsampling a "hard minibatch" of point samples [S1]. The algorithm employs propagated pixel loss to rank point samples and subsequently subsamples a hard minibatch, estimated to retain most of the update information [S1]. Unlike approaches that simply prune low-weight samples, HSM uses the magnitude of the reconstruction error to prioritize challenging regions and skip well-reconstructed ones [S1].

## How it works
**Inputs:**
*   A batch of point samples $\mathcal{S}$ associated with training images [S1].
*   Propagated pixel loss values $L_i$ for each sample $i \in \mathcal{S}$ [S1].
*   Total batch size $B$ [S1].

**Outputs:**
*   A subsampled set of hard samples $\mathcal{S}_{\text{hard}} \subset \mathcal{S}$ used for the backward pass and parameter update [S1].

**Data Structures:**
*   **Sample List:** A list of tuples containing the sample index, its associated pixel coordinates, and its computed loss value $L_i$ [G].
*   **Ranking Array:** An array of indices sorted by $L_i$ in descending order [G].

**Steps:**
1.  **Loss Computation:** Compute the pixel-wise loss $\ell_i$ for each sample $i$ against the ground truth [S1].
2.  **Ranking:** Sort all samples in the batch $\mathcal{S}$ based on their loss values $\ell_i$ in descending order [S1].
3.  **Subsampling:** Select the top samples from the sorted list to form the hard minibatch $\mathcal{S}_{\text{hard}}$, constrained by the target batch size $B$ [S1].
4.  **Backward Pass:** Compute gradients and update network parameters using only the samples in $\mathcal{S}_{\text{hard}}$ [S1].

**Parameters:**
*   $B$: Total batch size. Typical values: 220 for Instant-NGP, 218 for Nerfacto [S1].

**Edge Cases and Pitfalls:**
*   **Uniformly Challenging Content:** Scenes with unbounded backgrounds or multiple outward-facing objects result in a more uniform pixel loss distribution, limiting computational efficiency gains [S1].
*   **Pose Errors:** Severe pose mistakes in the dataset can produce "impossible to learn" point samples that consistently have high loss and crowd the hard minibatch, potentially degrading training stability [S1]. Mitigation strategies include uncertainty or pose estimation techniques [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def hard_sample_mining_step(network, samples, ground_truth, batch_size):
    # 1. Loss Computation
    # Compute per-sample loss (e.g., Mean Squared Error)
    loss_values = [sum((p - g) ** 2 for p, g in zip(sample, gt)) / len(sample) 
                   for sample, gt in zip(samples, ground_truth)]

    # 2. Ranking
    # Sort indices by loss value in descending order
    sorted_indices = sorted(range(len(loss_values)), key=lambda i: loss_values[i], reverse=True)

    # 3. Subsampling
    # Select top indices up to batch_size
    hard_indices = sorted_indices[:batch_size]

    # 4. Backward Pass
    # Gather predictions and targets for hard samples
    hard_preds = [samples[i] for i in hard_indices]
    hard_gts = [ground_truth[i] for i in hard_indices]

    # Compute loss on the hard minibatch
    hard_loss = sum(sum((p - g) ** 2 for p, g in zip(pred, gt)) / len(pred) 
                    for pred, gt in zip(hard_preds, hard_gts)) / len(hard_preds)

    # Perform gradient update using only the hard samples
    hard_loss.backward()

    return hard_loss
```

## Operating characteristics
*   **Time Complexity:** Reduces iteration time by focusing updates on a subsampled hard minibatch rather than the full batch [S1].
*   **Memory Complexity:** Reduces dynamic memory footprint during training; for example, usage dropped from 300 MB to 160 MB for normal iterations in Nerfacto [S1].
*   **Performance:** On Mip-NeRF-360 scenes, converges ~2x faster to a given PSNR level and achieves ~1 dB gain per same training time compared to the baseline [S1].
*   **Strengths:** Effective for scenes with centrally placed objects against easy backgrounds where loss distribution is skewed [S1].
*   **Limitations:** Less effective in uniformly challenging scenes (e.g., unbounded scenes) where loss distribution is uniform [S1]. Sensitive to severe pose errors in the dataset [S1].
*   **When to use:** When training time and memory are bottlenecks, and the scene has distinct hard/easy regions [S1].
*   **When not to use:** When the dataset has significant pose errors or the scene is uniformly complex [S1].

## Background
Hard Sample Mining was introduced by J. Korhonen et al. in the paper "Efficient NeRF Optimization -- Not All Samples Remain Equally Hard" [S1]. The method was applied to PyTorch reimplimentations of Instant-NGP and Nerfacto on Mip-NeRF-360 and NeRF synthetic datasets [S1]. It differs from Nerfacc's approach of pruning low-weight samples by using propagated pixel loss to rank and subsample a hard minibatch [S1].

## Sources
[S1] Efficient NeRF Optimization -- Not All Samples Remain Equally Hard, https://arxiv.org/abs/2408.03193v1, fetched 2026-10-27
