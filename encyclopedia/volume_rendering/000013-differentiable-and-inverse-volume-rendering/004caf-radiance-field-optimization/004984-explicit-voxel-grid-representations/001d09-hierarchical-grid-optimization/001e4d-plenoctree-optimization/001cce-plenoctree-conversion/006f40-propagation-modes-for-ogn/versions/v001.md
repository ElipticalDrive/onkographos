# Propagation Modes for OGN

Also called: Prop-known, Prop-pred propagation modes, Prop-pred

## Overview
Propagation modes define the training and inference regimes for Octree Generating Networks (OGN) based on whether the output octree structure is known at test time [S1]. **Prop-known** assumes the target structure is fixed and known, while **Prop-pred** simulates an unknown structure by predicting which cells are "mixed" and propagating features accordingly [S1]. The choice of mode determines whether the network learns to handle erroneous propagations, which is critical when the ground-truth structure is unavailable during inference [S1].

## How it works
**Inputs:**
- Trained OGN weights.
- Input data (e.g., images or latent codes).
- Network architecture definition (specifically convolutional filter sizes and strides).

**Outputs:**
- Refined OGN weights (if fine-tuning).
- Final octree structure and feature values.

**Data Structures:**
- **OGN Cell:** Represents a node in the octree. Contains a status flag (e.g., "empty", "mixed", "full") and feature vectors (e.g., SH coefficients for PlenOctree).
- **Propagation Mask:** A binary or categorical map indicating which cells require neighbor propagation based on the predicted structure.

**Steps:**

1. **Determine Propagation Mode:**
   - If the output octree structure is known at test time, use **Prop-known**.
   - If the output octree structure is unknown at test time, use **Prop-pred**.

2. **Training with Prop-known:**
   - Train the network until convergence using the ground-truth structure.
   - In this mode, the network assumes the structure is correct and does not account for prediction errors [S1].

3. **Fine-tuning with Prop-pred (Conditional):**
   - If the structure is unknown at test time, fine-tune the converged network using **Prop-pred** [S1].
   - During Prop-pred, the network predicts which cells are "mixed" and propagates features from neighboring cells as required by subsequent layers.
   - This step teaches the network to handle erroneous propagations that occur when the predicted structure deviates from the ground truth [S1].

4. **Inference:**
   - If trained with Prop-known and tested with Prop-known: No fine-tuning needed. Test directly [S1].
   - If trained with Prop-known and tested with Prop-pred: Use the fine-tuned weights from Step 3 [S1].
   - If trained with Prop-pred and tested with Prop-known: Performance is suboptimal compared to Prop-known training [S1].

**Parameters:**
- **Propagation Mode:** `Prop-known` or `Prop-pred`.
- **Filter Sizes:** Determine the number of neighbors to propagate (see OGNProp Feature Propagation Rule).

**Pitfalls:**
- Using Prop-pred fine-tuning when the structure is known at test time hurts performance [S1].
- Failing to propagate the required number of neighboring features causes incorrect computation in subsequent layers [S1].

## Subtopics

### OGNProp Feature Propagation Rule
A rule to determine how many neighboring features must be propagated alongside a 'mixed' cell's features during an OGNProp layer, based on subsequent convolutional filter sizes [S1]. The number of neighbors is calculated automatically before training based on the network architecture [S1]. For a $2^3$ up-convolution with stride 2, propagate 0 neighbors. For a $4^3$ up-convolution with stride 2 followed by a $3^3$ convolution with stride 1, propagate 2 neighbors [S1]. Failing to propagate the required neighboring features causes incorrect computation in subsequent convolutional or up-convolutional layers [S1].

## Operating characteristics
**Time and Memory Complexity:**
- Not given in the sources.

**Measured Performance:**
- Performance is measured using a metric (likely PSNR or similar, though not explicitly named in the provided notes) on a table with 3 filters, 4 filters, and IntConv configurations [S1].
- **Known/known:** 0.904 (3 filters), 0.907 (4 filters), 0.907 (IntConv) [S1].
- **Known/pred:** 0.862 (3 filters), 0.804 (4 filters), 0.823 (IntConv) [S1].
- **Pred/known:** 0.898 (3 filters), 0.896 (4 filters), 0.897 (IntConv) [S1].
- **Pred/pred:** Data incomplete in source [S1].

**Strengths and Limitations:**
- Prop-pred fine-tuning improves performance when the structure is unknown at test time, especially with larger filters, because the network learns to handle erroneous propagations [S1].
- Fine-tuning with Prop-pred hurts performance when the structure is known at test time [S1].

**When to Use:**
- Use Prop-known for training and testing when the output structure is known.
- Use Prop-pred fine-tuning only when the output structure is unknown at test time.

## Background
Originated in the Octree Generating Networks (OGN) paper [S1]. The method addresses the challenge of training neural networks to generate sparse octree structures where the final topology may not be known during inference. The distinction between Prop-known and Prop-pred allows for robust handling of structural uncertainty in 3D output generation [S1].

## Sources
[S1] Octree Generating Networks: Efficient Convolutional Architectures for High-resolution 3D Outputs, https://doi.org/10.1109/iccv.2017.230, fetched 2026-10-26
