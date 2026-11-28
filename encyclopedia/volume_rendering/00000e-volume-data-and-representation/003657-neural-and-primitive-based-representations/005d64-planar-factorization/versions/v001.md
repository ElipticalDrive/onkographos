# Planar Factorization
Also called: Planar Neural Fields, Factorized Neural Fields

## Overview
Planar Factorization methods represent continuous volumetric or spatiotemporal fields by decomposing high-dimensional feature tensors into a set of lower-dimensional 2D planes . This approach solves the memory and computational bottlenecks of dense 3D/4D voxel grids and the black-box nature of pure MLP fields by using explicit, learnable planar structures . Unlike sibling methods that aggregate features via concatenation or summation, this family typically employs element-wise multiplication (Hadamard product) of plane features to achieve spatial localization, enabling efficient decoding .

## Shared principles
All methods in this family define a scene function $F_\Theta: \mathbb{R}^d \rightarrow \mathbb{R}^k$ using a set of parameterized 2D feature planes . The input is a continuous coordinate vector $\mathbf{q} \in \mathbb{R}^d$ (typically spatial $\mathbf{x}$, viewing direction $\mathbf{d}$, or time $t$) . The output is a signal vector (e.g., density $\sigma$ and color $\mathbf{c}$) [G].

The core mechanism involves projecting the $d$-dimensional query point onto $k = \binom{d}{2}$ orthogonal 2D planes . For each plane $c$, features are retrieved via interpolation (bilinear or trilinear) from the plane grid $\mathbf{P}_c$ . These features are then combined, typically via element-wise multiplication (Hadamard product) $\mathbf{f}(\mathbf{q}) = \prod_{c \in C} \mathbf{f}_c(\mathbf{q})$, to form a fused feature vector . This product operation ensures that the resulting feature is non-zero only if the point lies within the active regions of all constituent planes, providing spatial localization . The fused features are then decoded into density and color, either via a linear decoder with a learned basis or a small MLP .

## Subtopics

### [K-planes](3317-k-planes.md)
A white-box hybrid representation for volumetric rendering in arbitrary dimensions $d$ that factorizes the scene function into $k = \binom{d}{2}$ learned 2D feature planes. It solves the scalability and black-box nature of implicit neural fields by using explicit planar grids combined with a decoder, achieving high compression (e.g., 1000x over 4D grids) and fast optimization. Unlike sibling methods that add features from planes, K-planes multiplies them element-wise, which enables spatial localization and allows the use of a linear decoder with a learned color basis instead of a non-linear MLP.

### [Forplane](5177-forplane.md)
A neural implicit representation method designed for efficient 4D (spatiotemporal) reconstruction of deformable scenes, specifically targeting surgical video where tools cause occlusions and tissues deform. It solves the high computational cost of 4D volume traversal by factorizing the 4D volume into a static 3D field and a dynamic 3D field, both represented by 2D orthogonal planes. Unlike methods that treat time steps as independent static volumes or rely on displacement fields, Forplane separates static scene priors from dynamic deformations, reducing computational complexity from $O(N^4)$ to $O(N^2)$ while maintaining reconstruction quality. It builds on [Neural and primitive-based representations](10-neural-and-primitive-based-representations.md) by using a factorized grid structure instead of a monolithic MLP or voxel grid.

### [Feature Grid as Radiance Field](5508-feature-grid-as-radiance-field.md)
Represents a scene using explicit, low-rank tensor-decomposed feature grids for density and appearance, replacing the implicit multi-layer perceptron (MLP) regression used in standard Neural Radiance Fields. It solves the memory bottleneck of dense 3D grids by factorizing the feature tensors into lower-dimensional components, reducing space complexity from cubic to quadratic relative to resolution. Unlike sibling methods that rely on hash grids [Hash Grid Neural Fields](3855-hash-grid-neural-fields.md) or Gaussian primitives [Gaussian Head Avatar (GHA)](1478-gaussian-head-avatar-gha.md), this method maintains a regular grid structure optimized for spherical parameterization, specifically targeting egocentric view synthesis.

## Background
Planar factorization emerged as a solution to the high memory costs of 3D/4D neural fields. K-planes introduced the concept of factorizing 3D/4D fields into 2D planes using Hadamard products to enable linear decoding . Forplane adapted this for dynamic surgical scenes, separating static and dynamic components . Feature Grid as Radiance Field applied tensor factorization to spherical grids for egocentric views .

## Sources
[S1] Forplane: Efficient 4D Neural Implicit Representation for Surgical Video, URL, date fetched
[S2] K-planes: Expressive Neural Fields for Volumetric Rendering, URL, date fetched
