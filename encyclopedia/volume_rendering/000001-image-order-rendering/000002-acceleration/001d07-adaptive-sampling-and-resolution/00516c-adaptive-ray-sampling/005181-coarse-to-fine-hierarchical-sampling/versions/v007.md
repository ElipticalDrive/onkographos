# Coarse-to-Fine Hierarchical Sampling
## Overview
Coarse-to-Fine Hierarchical Sampling is a family of adaptive ray sampling strategies that improve the learning of high-frequency details in Neural Radiance Fields by performing multiple tiers of volume sampling [G]. It solves the problem of inefficient uniform sampling, where most samples fall in empty space, by using a coarse pass to identify opaque or high-complexity regions and then clustering a second set of samples near those regions to train a fine network [G]. It sets itself apart from standard uniform sampling by adaptively concentrating computational effort on visible geometry, and from its sibling [TermiNeRF](8828-terminerf.md) by not explicitly modeling termination probabilities but rather relying on opacity-based importance sampling [G].
## Shared principles
The core mechanism shared by all children is the deviation from a fixed, uniform sampling strategy [G].
## Subtopics

### [Coarse-to-Fine NeRF](3406-coarse-to-fine-nerf.md)
Uses a coarse MLP to generate weights, then samples fine points from the resulting PDF. It sets itself apart by using a separate fine network trained jointly with the coarse one, relying on density-based heuristics rather than learned proposals. Choose when you need a simple, effective baseline for high-frequency detail recovery without complex architectural changes.

### [Context-Based Adaptive Ray Sampling with Quadtree Subdivision](8164-context-based-adaptive-ray-sampling-with-quadtree-subdivisio.md)
Uses a quadtree structure to dynamically adjust ray distribution across input views during training. It sets itself apart by using rendering error to prune redundant blocks and integrating context-based probability density for intra-block sampling. Choose when you want to accelerate training by focusing rays on high-complexity image regions.

### [TermiNeRF](8828-terminerf.md)
Trains a dedicated sampling network to predict importance weights for fixed bins, replacing the heuristic coarse-to-fine step. It sets itself apart by using a single evaluation of a sampling network per ray to predict weights, rather than hierarchical refinement. Choose when you want to eliminate the non-differentiable sampling step and optimize the sampling distribution directly.

### [NeRF-ID](8867-nerf-id.md)
Replaces heuristic sampling with a differentiable 'proposer' module that outputs fine sample locations from coarse features. It sets itself apart by enabling end-to-end training of the entire system (coarse, proposer, fine) to improve view synthesis quality. Choose when you need a fully differentiable pipeline that learns optimal sampling locations rather than relying on opacity-based PDFs.

### [Two-Stage Sampling with Occupancy Grid and Nonlinear Function](9146-two-stage-sampling-with-occupancy-grid-and-nonlinear-functio.md)
Combines spatial skipping with biased sampling using an occupancy grid and a nonlinear piecewise exponential function. It sets itself apart by skipping empty space and biasing samples toward surfaces within occupied cells. Choose when you need to reduce wasted computation in empty regions while avoiding random scatter near surfaces.

### Simple Sampling
A sampling scheme proposed by the authors that preserves rendering quality while significantly increasing runtime efficiency compared to Hierarchical Sampling [S1].

## Background
Not given in the sources.
## Sources
[S1] Learning Compositional Radiance Fields of Dynamic Human Heads, https://arxiv.org/abs/2012.09955v1, fetched 2026-10-26
