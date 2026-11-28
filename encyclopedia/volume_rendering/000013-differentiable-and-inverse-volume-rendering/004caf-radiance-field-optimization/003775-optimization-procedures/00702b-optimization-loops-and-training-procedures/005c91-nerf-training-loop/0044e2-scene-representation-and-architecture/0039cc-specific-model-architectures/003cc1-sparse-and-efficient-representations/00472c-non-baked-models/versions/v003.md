# Non-Baked Models

Also called: Hybrid Representations, Learning separate scene features from MLP parameters

## Overview
Non-baked models are scene representations that decouple the learning of geometric/appearance features from the neural network parameters, typically using a hybrid approach where features are stored in explicit data structures and queried by a smaller MLP. This approach solves the trade-off between model capacity and inference speed by enabling smaller MLPs, which improves both training and inference speed at the cost of increased memory usage [S1]. Unlike baked models, which pre-compute and store the final output in accessible data structures to speed up inference without affecting training time, non-baked models retain the neural network for the final mapping step [S1].

## Shared principles
The core mechanism of non-baked models is the separation of scene representation into two distinct components: an explicit feature storage structure and a compact neural network (MLP).

1.  **Explicit Feature Storage**: The scene is represented by a data structure (e.g., hash tables, grids, or other spatial partitions) that stores intermediate features (such as color, density, or embeddings) at specific spatial locations or along rays. These features are learned during training.
2.  **Compact MLP**: A smaller, parameter-efficient MLP takes the queried features as input and predicts the final radiance and density values. The MLP parameters are shared across the scene or spatial regions, while the features are location-specific.
3.  **Hybrid Nature**: The term "hybrid" refers to the combination of explicit, learnable feature storage (like a lookup table) and implicit neural representation (the MLP). This contrasts with purely implicit models (like original NeRF, which uses only an MLP) and purely explicit models (which omit neural networks entirely) [S1].
4.  **Performance Trade-off**: The primary benefit is improved inference speed due to the smaller MLP, and potentially faster training due to reduced computational load per sample. The cost is increased memory consumption due to the storage of explicit features [S1].

## Subtopics

### Purely Explicit Representations
Methods that omit neural networks entirely and use only explicit scene representations. While not strictly NeRF models, they are included in this family due to their relevance and similarity [S1].

### Hybrid Feature-MLP Models
The standard non-baked approach where separate scene features are learned and stored explicitly, while a smaller MLP processes these features. This is the most common form of non-baked models [S1].

## Background
The concept of non-baked models emerged as an optimization strategy to address the computational bottlenecks of standard NeRF architectures. The original NeRF paper utilized hierarchical sampling and coarse/fine MLPs, which are often combined with per-paper innovations in non-baked approaches [S1]. The distinction between "baked" and "non-baked" models was formalized in comprehensive reviews of Neural Radiance Fields, where baked models were defined as those storing final outputs in accessible data structures, while non-baked models included innovations like hybrid representations [S1].

## Sources
[S1] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
