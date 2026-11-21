# Loss Function Configuration

Also called: Loss Weighting Scheme, Training Loss Weights

## Overview
This method defines the specific hyperparameters and weighting strategies used to balance multiple loss components during the optimization of radiance fields. It addresses the problem of stabilizing training by preventing any single loss term (e.g., photometric error vs. regularization) from dominating the gradient updates. It differs from sibling methods by focusing on the static configuration of scalar weights for the total loss function rather than the dynamic computation of loss values or visibility masks.

## How it works
The total loss $\mathcal{L}$ is a weighted sum of individual loss terms. The configuration involves setting scalar weights for each term, which may vary between training stages (coarse vs. fine) or scene characteristics.

**Inputs:**
- Target photometric loss $\mathcal{L}_{photo}$
- Point-to-Point correspondence loss $\mathcal{L}_{ptc}$
- Background loss $\mathcal{L}_{bg}$
- Deformation norm regularization loss $\mathcal{L}_{d\_norm}$
- Total Variation regularization loss $\mathcal{L}_{d\_tv}$

**Outputs:**
- Scalar weights $w_{ptc}, w_{bg}, w_{d\_norm}, w_{d\_tv}$ for the current training stage.

**Data Structures:**
- A configuration object or dictionary mapping loss term identifiers to float values.

**Steps:**
1. Determine the current training stage (Coarse or Fine).
2. Retrieve the base weight set for the stage.
3. If the scene exhibits large motion, adjust $w_{d\_norm}$ to a smaller value.
4. Apply the weights to the respective loss terms during the backward pass.

**Formula:**
The overall loss is computed as [S1]:
$$
\mathcal{L} = \mathcal{L}_{photo} + w_{ptc} \cdot \mathcal{L}_{ptc} + w_{bg} \cdot \mathcal{L}_{bg} + w_{d\_norm} \cdot \mathcal{L}_{d\_norm} + w_{d\_tv} \cdot \mathcal{L}_{d\_tv}
$$

**Parameters and Typical Values:**
- **Coarse Stage:**
  - $w_{ptc} = 0.1$
  - $w_{bg} = 0.01$
  - $w_{d\_norm} = 0.1$
  - $w_{d\_tv} = 1$ [S1]
- **Fine Stage:**
  - $w_{ptc} = 0.01$
  - $w_{bg} = 0.001$
  - $w_{d\_norm} = 0.01$
  - $w_{d\_tv} = 1$ [S1]
- **Large Motion Scenes:** (e.g., 'Hell Warrior', 'Jumping Jacks', 'T-Rex')
  - $w_{d\_norm}$ is reduced to $0.01$ (Coarse) and $0.001$ (Fine) [S1].

**Pitfalls:**
- Using high $w_{d\_norm}$ in scenes with large motion can over-constrain the deformation field, leading to artifacts or convergence issues [S1].
- The weights are empirical; they may require tuning for different datasets or resolution scales.

## Subtopics

### Hybrid Gradients
A strategy combining RGBXY derivatives and standard derivatives to leverage global matching in early stages and precise local alignment in late stages [S2]. Use RGBXY derivatives (Eq. 15 in [S2]) for the first 75% of optimization iterations, then switch to standard derivatives (Eq. 5 in [S2]) for the last 25% [S2]. This is chosen when initial and target shapes are misaligned globally but require fine local alignment later. A pitfall is that when initial and target shapes are already aligned very well, the approximated optimal transport method used in RGBXY derivatives may lead to inaccurate matching, making them less effective [S2].

## Operating characteristics
- **Time Complexity:** Negligible overhead; weight application is a scalar multiplication per loss term.
- **Memory Complexity:** $O(1)$ for storing the weight configuration.
- **Strengths:** Provides a stable trade-off between photometric fidelity and geometric regularization. The coarse-to-fine weight reduction prevents overfitting in the fine stage [S1].
- **Limitations:** Requires manual tuning for new scenes or motion magnitudes. The fixed 75%/25% split in Hybrid Gradients may not be optimal for all convergence speeds [S2].
- **When to use:** Standard radiance field optimization where multiple loss terms are present.
- **When not to use:** When a single loss term dominates or when the scene geometry is static and simple.

## Background
Originated in the context of Neural Deformable Voxel Grids for dynamic view synthesis [S1] and Differentiable Rendering using RGBXY Derivatives [S2]. The specific weight configurations for coarse/fine stages were empirically determined to balance PSNR and training time, with 20k iterations found to be a practical limit for convergence [S1].

## Sources
[S1] Neural Deformable Voxel Grid for Fast Optimization of Dynamic View Synthesis, https://doi.org/10.1007/978-3-031-26319-4_27, fetched 2026-10-26
[S2] Differentiable Rendering Using RGBXY Derivatives and Optimal Transport, https://doi.org/10.1145/3550454.3555479, fetched 2026-10-27
