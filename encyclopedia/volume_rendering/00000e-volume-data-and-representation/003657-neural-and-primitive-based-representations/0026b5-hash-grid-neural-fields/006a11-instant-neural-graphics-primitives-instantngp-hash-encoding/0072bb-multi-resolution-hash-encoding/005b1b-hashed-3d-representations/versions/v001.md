# Hashed 3D Representations

Also called: Hybrid 3D Representations, Hybrid Representation with 3D Hash Structure

## Overview
Hashed 3D Representations are a family of neural 3D field parameterizations that decompose a field $f$ into a hashing scheme $h$ and a small MLP $g$, such that $f = g \circ h$ [S1]. They solve the memory bottleneck inherent in dense 3D storage by using sparse or structured hash-like access patterns (e.g., voxel grids, triplanes) to sample features efficiently, enabling fast, memory-efficient training and rendering [S1]. What sets them apart from dense voxel-based methods is the decoupling of spatial indexing from the neural network, allowing the use of compact structures like triplanes where cells lack explicit 3D coordinates, and from pure neural fields by providing a differentiable, interpolation-based sampling mechanism [S1].

## Shared principles
The core mechanism is the decomposition of the neural field $f: \mathbb{R}^3 \to \mathbb{R}^C$ into two components:
1.  **Hashing Scheme ($h$):** A sampling operation $h: \mathbb{R}^3 \to \mathbb{R}^K$ that retrieves feature vectors from a structured parameter set $\theta$ based on a 3D input position $x$. This operation is typically differentiable via interpolation [S1].
2.  **MLP ($g$):** A small neural network $g: \mathbb{R}^K \to \mathbb{R}^C$ that takes the sampled features and outputs the final field values (e.g., color, density) [S1].

The parameter set $\theta$ represents the "hash structure." While termed "hashed," this family includes structured grids where the "hashing" refers to the efficient lookup and interpolation rather than a collision-based hash table [S1]. The key constraint is that $h$ must be efficient and differentiable [S1].

## Subtopics

### Voxel-based Hashed Representation
A specific instantiation where $\theta$ is a dense 3D tensor of dimensions $H \times W \times D \times K$ [S1]. The hashing scheme $h$ performs tri-linear interpolation on $\theta$ given the 3D position $x$ [S1]. This method is chosen when explicit 3D spatial locality is required and the memory budget allows for a dense grid [S1]. It supports direct 3D-to-2D lifting and projection operations [S1].

### Triplane-based Hashed Representation
A specific instantiation where $\theta$ consists of three 2D tensors with dimensions $H \times W \times K$, $W \times D \times K$, and $D \times H \times K$, representing the XY, YZ, and ZX planes respectively [S1]. The hashing scheme $h(x, y, z)$ is computed by performing bilinear interpolation on each plane at the projected 2D coordinates $(x, y)$, $(y, z)$, and $(z, x)$, and then summing the resulting three feature vectors [S1]. This method is chosen for its superior memory efficiency compared to 3D voxels and its ability to model complex geometries with fewer parameters [S1]. However, because the cells are defined by 2D positions on planes rather than 3D coordinates, they lack clearly-defined 3D positions, complicating direct 3D lifting operations [S1].

## Background
The concept was formalized in the context of the Lightplane architecture, which aimed to address the memory bottlenecks of previous neural 3D fields like NeRF and PixelNeRF [S1]. Prior methods were limited by the memory required to store intermediate values for backpropagation, restricting input view numbers and representation sizes [S1]. The Lightplane paper generalized existing structures like voxel grids [78], triplanes [9], TensoRF [14], and HexPlane [7, 10] under the unified framework of "hybrid 3D representations with 3D hash structures" [S1]. The term "hashed" is used broadly to encompass any structure where information is accessed via a small amount of stored data in a particular bucket, including actual hash tables [56] [S1]. The approach was adopted because these representations are fast, relatively memory efficient, and familiar to practitioners, while allowing components like the Lightplane Renderer and Splatter to be easily extended to other hashed representations [S1].

## Sources
[S1] Lightplane: Highly-Scalable Components for Neural 3D Fields, https://doi.org/10.1109/3dv66043.2025.00058, fetched 2026-10-29
