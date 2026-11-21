# Coarse-to-Fine Hierarchical Sampling

Also called: Hierarchical Sampling, Two-Stage Sampling, Coarse-Fine NeRF

## Overview
Coarse-to-Fine Hierarchical Sampling is a strategy that improves the learning of high-frequency details in Neural Radiance Fields by performing two tiers of volume sampling . It solves the problem of inefficient uniform sampling, where most samples fall in empty space, by using a coarse network to identify opaque regions and then clustering a second set of samples near those regions to train a fine network . It sets itself apart from standard uniform sampling by adaptively concentrating computational effort on visible geometry, and from its sibling [TermiNeRF](8828-terminerf.md) by not explicitly modeling termination probabilities but rather relying on opacity-based importance sampling .

## Shared principles
The family shares the deviation from a fixed, uniform sampling strategy [G]. Instead of casting a constant number of rays or samples per pixel/ray, the method computes a metric (such as gradient magnitude, rendering error, or temporal difference) and uses it to modulate the sampling rate or resolution [G].

Mathematically, this often involves a control loop:
1. **Metric Evaluation:** Compute a scalar value $M$ representing local complexity or error [G].
2. **Adaptation:** Adjust a parameter $P$ (step size $\Delta t$, resolution $R$, or sampling probability $w$) based on $M$ [G].
3. **Rendering/Training:** Execute the volume integral or loss computation with the adjusted parameter [G].

Common adaptation laws include:
* **Step Size Adaptation:** $\Delta t \propto \frac{1}{M}$ or $\Delta t = \min(\Delta t_{max}, \max(\Delta t_{min}, \frac{k}{M}))$ [G].
* **Resolution Control:** $R_{new} = R_{old} + \alpha \cdot (t_{target} - t_{actual})$ or threshold-based switching [G].
* **Probability Weighting:** $w_i \propto M_i$ for importance sampling [G].

All methods in this family aim to maintain visual fidelity while reducing the total number of volume queries .

## Subtopics

### [Coarse-to-Fine NeRF](3406-coarse-to-fine-nerf.md)
Uses a coarse MLP to generate weights, then samples fine points from the resulting PDF. It sets itself apart by using a separate fine network trained jointly with the coarse one, relying on density-based heuristics rather than learned proposals. Choose when you need a simple, effective baseline for high-frequency detail recovery without complex architectural changes.

### [TermiNeRF](8828-terminerf.md)
Trains a dedicated sampling network to predict importance weights for fixed bins, replacing the heuristic coarse-to-fine step. It sets itself apart by using a single evaluation of a sampling network per ray to predict weights, rather than hierarchical refinement. Choose when you want to eliminate the non-differentiable sampling step and optimize the sampling distribution directly.

### [NeRF-ID](8867-nerf-id.md)
Replaces heuristic sampling with a differentiable 'proposer' module that outputs fine sample locations from coarse features. It sets itself apart by enabling end-to-end training of the entire system (coarse, proposer, fine) to improve view synthesis quality. Choose when you need a fully differentiable pipeline that learns optimal sampling locations rather than relying on opacity-based PDFs.

## Background
The concept was introduced in the original NeRF paper (Mildenhall et al., 2020) as a way to mitigate the inefficiency of uniform sampling . It became a standard component in subsequent NeRF variants due to its ability to allocate samples to geometrically relevant regions.

## Sources
[S1] Mildenhall, B., et al. "NeRF: Representing Scenes as Neural Radiance Fields for View Synthesis." ECCV, 2020.
[S2] Unofficial implementation notes.
[S3] Mildenhall, B., et al. "NeRF: Representing Scenes as Neural Radiance Fields for View Synthesis." ECCV, 2020.
[S4] Unofficial implementation notes.
[S5] Unofficial implementation notes.
