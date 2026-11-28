# SDF Training with Chamfer Distance

Also called: CAPUDF training loop, UDF network training with Chamfer distance loss, gradient descent surface projection

## Overview
This method trains a neural network to predict a Signed Distance Field (SDF) or Unoriented Distance Field (UDF) by projecting sampled query points onto the implicit surface using the field's gradient and signed distance values, then minimizing the Chamfer distance between these projected points and ground truth point clouds [S1][S2, unofficial]. It differs from standard SDF training by explicitly enforcing surface alignment through geometric projection rather than relying solely on volume rendering losses or Eikonal regularization [S1][S2, unofficial].

## How it works
**Inputs:**
- Ground truth point cloud $P_{GT}$.
- A set of sampled query points $Q = \{q_j\}$ in 3D space.
- Camera poses or eye positions $e(q)$ associated with each query point $q$ [S1].
- Neural network parameters $\theta$ defining the distance field function $f_\theta$ [S1].

**Outputs:**
- Updated network parameters $\theta$ that minimize the Chamfer distance loss.
- Projected points $P' = \{p'_j\}$ lying on the zero level set of $f_\theta$ [S1].

**Data Structures:**
- Query points $q$: 3D vectors $[x, y, z]$ [S1].
- Distance field function $f_\theta(q, e(q))$: Returns scalar distance value [S1].
- Gradient $\nabla f_\theta(q, e(q))$: 3D vector of partial derivatives [S1].

**Steps:**
1. **Sample Query Points:** Generate a set of query points $Q$ in the domain of interest. In implementations, these may be structured grids or random samples [S2, unofficial].
2. **Evaluate Distance and Gradient:** For each query point $q \in Q$, compute the distance value $d = f_\theta(q, e(q))$ and the gradient $\nabla f_\theta(q, e(q))$ [S1]. Ensure `requires_grad` is set to `True` on the input samples to enable backpropagation [S2, unofficial].
3. **Project to Surface:** Move each point $q$ onto the zero level set (the implicit surface) using the gradient descent step defined by the signed distance and normalized gradient [S1][S2, unofficial]:
   $$ p'_j = q - \frac{f_\theta(q, e(q))}{\|\nabla f_\theta(q, e(q))\|} \nabla f_\theta(q, e(q)) $$
   This formula moves $q$ along the negative gradient direction by a distance equal to its signed distance value [S1][S2, unofficial].
4. **Compute Chamfer Distance:** Calculate the Chamfer distance between the projected points $P' = \{p'_j\}$ and the ground truth point cloud $P_{GT}$ [S2, unofficial]. This involves finding the nearest neighbor in $P_{GT}$ for each $p'_j$ and summing the squared distances [S2, unofficial].
5. **Optimize:** Minimize the Chamfer distance loss with respect to $\theta$ using gradient descent [S2, unofficial].

**Parameters:**
- $f_\theta$: The SDF/UDF function [S1].
- $e(q)$: Eye position or camera pose for point $q$ [S1].
- $df\_filter$: A threshold for UDF values used to filter points during extra point cloud generation in some implementations [S2, unofficial].

**Pitfalls:**
- The calculation of $\nabla f_\theta(p'_j, e(p'_j))$ involves second-order derivatives because $p'_j$ depends on the gradient at $q$ [S1]. This can increase computational cost and complexity but helps the network find better solutions [S1].
- Ensure `requires_grad = True` is set on the query points before computing gradients to enable backpropagation through the projection step [S2, unofficial].

## Operating characteristics
Not given in the sources.

## Background
This method is used in the CAPUDF framework for inferring signed distance functions from volume rendering [S1]. It leverages the property that points on a level set can be moved to the zero level set using the field's gradient and signed distance [S1]. The approach is also referenced in unofficial implementations of distance field training [S2, unofficial].

## Sources
[S1] Sensing Surface Patches in Volume Rendering for Inferring Signed Distance Functions, https://doi.org/10.1609/aaai.v39i4.32426, fetched 2026-10-26
[S2] jjjkkyz/DCx, https://github.com/jjjkkyz/DCx, fetched 2026-10-26 (unofficial)
