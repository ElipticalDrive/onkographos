# Uncertainty and Reliability


## Overview
This family extends NeRF architectures to quantify prediction confidence, addressing artifacts, redundancy, and geometry collapse in sparse or unstructured data. It differs from standard NeRF by modeling outputs as distributions (Gaussian) rather than deterministic points, and from sibling uncertainty methods by specific integrations like conformal prediction or semantic consistency constraints.

## Shared principles
The core task of a NeRF network architecture is to learn a continuous volumetric scene representation $F_\Theta: (\mathbf{x}, \mathbf{d}) \rightarrow (\sigma, \mathbf{c})$, where $\mathbf{x} \in \mathbb{R}^3$ is the spatial position, $\mathbf{d} \in \mathbb{R}^3$ is the viewing direction, $\sigma$ is the volume density, and $\mathbf{c}$ is the view-dependent color. The rendering process integrates these outputs along rays using standard volume rendering equations.

## Subtopics

### [Uncertainty-aware NeRF](12968-uncertainty-aware-nerf.md)
Renders RGB-D images with explicit uncertainty estimates for color and depth. It models rendered pixel values as Gaussian distributions and uses squared weights for variance aggregation, correcting the theoretical grounding of prior methods like NeRF-W.

### [UA-PlantNeRF Architecture](13768-ua-plantnerf-architecture.md)
Reconstructs volumetric radiance fields and uncertainty from sparse plant video. It jointly predicts density, color, and aleatoric uncertainty via a dual-head architecture, estimates epistemic uncertainty via Monte Carlo Dropout, and uses CLIP-based semantic consistency to prevent geometry collapse.

### Error Correction and Scene Conditioning

### Error Correction and Scene Conditioning
- **Definition**: The use of an error correction network and conditioning on a scene NeRF to prevent the canonical NeRF from hallucinating clothing dynamics or background colors [S1].
- **Pitfall**: Training only the human NeRF in isolation leads to worse performance because the human NeRF model may encode background pixels into its radiance fields due to segmentation errors [S1]. Without error correction or scene conditioning, the canonical NeRF creates fogs around the human to hallucinate the clothing dynamics or the background colors [S1].

## Background
Originating from NeuMan [S1], this approach addresses artifacts in human radiance fields from single videos. It was developed to handle segmentation errors and background leakage in dynamic scenes.

## Sources
[S1] NeuMan: Neural Human Radiance Field from a Single Video, https://doi.org/10.1007/978-3-031-19824-3_24, fetched 2026-10-29
