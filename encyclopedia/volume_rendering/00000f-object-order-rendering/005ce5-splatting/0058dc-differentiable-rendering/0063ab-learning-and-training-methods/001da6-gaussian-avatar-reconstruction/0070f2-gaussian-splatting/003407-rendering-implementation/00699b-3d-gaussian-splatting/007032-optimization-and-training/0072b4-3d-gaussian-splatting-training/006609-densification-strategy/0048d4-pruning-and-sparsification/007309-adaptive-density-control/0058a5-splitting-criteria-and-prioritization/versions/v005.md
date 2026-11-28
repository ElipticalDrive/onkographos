# Splitting Criteria and Prioritization

Also called: Densification scoring, Gradient-based splitting metrics

## Overview
Splitting Criteria and Prioritization determines which 3D Gaussians should be subdivided (split or cloned) during the training of Gaussian Splatting models to adapt their density to scene complexity. It solves the problem of static primitive distribution by dynamically adding primitives in high-error regions while removing redundant ones. It differs from its siblings by offering specific mathematical formulations for computing the "importance" or "error" score of a Gaussian, such as using squared gradient norms for multi-view consistency, complexity-aware sampling for LoD, or average contribution for occlusion handling.

## Shared principles
The core mechanism involves computing a scalar score $S(G)$ for each Gaussian $G$ based on training gradients, opacity, or reconstruction error. This score is compared against a threshold $\tau$. If $S(G) > \tau$, the Gaussian is marked for densification. The specific definition of $S(G)$ varies by child method. Common inputs include the loss gradient $\nabla L$, the Gaussian's opacity $\alpha$, and its screen-space footprint. The decision is typically made periodically during training steps.

## Subtopics

### [Multi-View Density Control Criterion](5663-multi-view-density-control-criterion.md)
A densification strategy for multi-view training that replaces the summation of 2D positional gradient vectors with a sum of squared norms. It resolves invalid vector arithmetic across different image planes by ensuring error metrics remain valid. It yields higher reconstruction quality with fewer primitives compared to standard positional-gradient rules.

### [Complexity-Aware Adaptive Density Control](16126-complexity-aware-adaptive-density-control.md)
Replaces opacity-only heuristics with a composite importance score incorporating image complexity, opacity, and scale. It solves inefficient primitive distribution by biasing densification toward high-complexity regions. It differs from standard 3DGS by using a complexity field as a surrogate for reconstruction error and from siblings by explicitly modeling the exponential relationship between particle count and visual complexity.

### [Average Contribution Criterion](16892-average-contribution-criterion.md)
Determines split/clone decisions based on the mean blending weight per view rather than a binary view count. It solves occlusion bias where fully occluded views inflate the denominator and dilute the gradient signal. It intrinsically weights views by their actual contribution, preventing occluded views from biasing the average downward.

### Distortion-based Gaussian Splitting Prioritization
A criterion for prioritizing which Gaussians to split, based on rate-distortion optimization to maximize PSNR gain [S1]. It associates a reconstruction Squared Error $SE_{c,i}$ to each Gaussian $i$ in camera view $c$, defined as:
$$SE_{c,i} = \sum_{p \in I_c} w_i^{(c)} (R^{(c)}(p) - I^{(c)}(p))^2$$
where $I^{(c)}$ is the ground-truth image, $R^{(c)}(p)$ is the rendered pixel value, and $w_i^{(c)}$ is the blending weight [S1]. This formulation spreads the error metric across Gaussians, allowing the selection of Gaussians that will induce the highest gain in PSNR quality when split [S1].

## Background
The concept of adaptive density control in Gaussian Splatting was introduced in the original 3D Gaussian Splatting paper [G]. The specific splitting criteria discussed here, such as the Multi-View Density Control Criterion and Average Contribution Criterion, are refinements proposed in subsequent research to address limitations in multi-view consistency and occlusion handling. The distortion-based prioritization is described in BOGausS [S1].

## Sources
[S1] BOGausS: Better Optimized Gaussian Splatting, https://doi.org/10.23919/eusipco63237.2025.11226626, fetched 2026-10-26
