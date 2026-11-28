# Self-Organizing Map

Also called: Kohonen Map, Kohonen's maps

## Overview
A Self-Organizing Map (SOM) is a neural network trained using unsupervised learning to produce a low-dimensional (typically two-dimensional), discretized representation of the input space of training samples [S1]. It maps high-dimensional data points onto a 2D grid organized by similarity, facilitating efficient data compression [S1]. It is used in volume rendering to structure voxel attributes (signatures) for transfer function design, building on [Self-Organizing Map Dimensional Reduction for Transfer Functions](6242-self-organizing-map-dimensional-reduction-for-transfer-funct.md).

## How it works
**Inputs:** A set of $N$ input vectors $\mathbf{x}_i \in \mathbb{R}^D$ (e.g., voxel signatures) and a grid of $M$ neurons with weight vectors $\mathbf{w}_j \in \mathbb{R}^D$.
**Outputs:** A mapping assigning each input vector to a neuron index $j$, and updated weight vectors $\mathbf{w}_j$ that represent the grid topology.

**Data Structures:**
- Grid: A 2D array of neurons. Each neuron $j$ has a weight vector $\mathbf{w}_j$.
- Input: List of vectors $\mathbf{x}_i$.

**Algorithm Steps:**
1. **Initialization:** Initialize weight vectors $\mathbf{w}_j$ randomly or using principal components.
2. **Competition:** For each input vector $\mathbf{x}_i$, find the Best Matching Unit (BMU) $c$ such that:
   $$ c = \arg \min_j \| \mathbf{x}_i - \mathbf{w}_j \| $$
   where $\| \cdot \|$ is the Euclidean distance [S1].
3. **Cooperation:** Determine the neighborhood of the BMU. Neurons closer to $c$ in the grid topology have a larger influence.
4. **Adaptation:** Update the weights of the BMU and its neighbors to move them closer to $\mathbf{x}_i$:
   $$ \mathbf{w}_j(t+1) = \mathbf{w}_j(t) + \alpha(t) h_{cj}(t) (\mathbf{x}_i - \mathbf{w}_j(t)) $$
   where $\alpha(t)$ is the learning rate and $h_{cj}(t)$ is the neighborhood function [S1].
5. **Iteration:** Repeat steps 2-4 for all inputs over multiple epochs, decreasing $\alpha(t)$ and the neighborhood radius over time.

**Pitfalls:** Due to the sequential nature of SOM training, the last input vectors can only be assigned to the few remaining unassigned map positions, resulting in isolated and poorly positioned vectors within the map [S1].

## Subtopics

### Self-Sorting Map
A variant that avoids the isolation problem of SOM by swapping assigned positions of four input vectors at a time [S1]. However, adding more candidates for swapping is computationally too complex due to the factorial number of permutations [S1].

### Fast Linear Assignment Sorting (Fast LAS)
An adaptation of Linear Assignment Sorting (LAS) that outperforms other sorting methods in speed and quality for non-parallel computations by swapping all (or several) vectors simultaneously using a continuously filtered map [S1]. It is designed to handle millions of items (e.g., Gaussians) where standard grid-sorting algorithms are inadequate [S1].

## Operating characteristics
**Time Complexity:** Sequential training is $O(N \cdot M \cdot D \cdot T)$ where $T$ is epochs. Self-Sorting Map swaps are limited to 4 vectors to avoid factorial complexity [S1]. LAS is faster for non-parallel computations [S1].
**Memory Complexity:** $O(M \cdot D)$ for weights.
**Strengths:** Produces a topologically ordered 2D map of high-dimensional data, facilitating visualization and compression [S1].
**Limitations:** Sequential nature leads to poor positioning of late-processed vectors [S1]. Standard grid-sorting is inadequate for millions of items [S1].

## Background
The Self-Organizing Map was introduced by Teuvo Kohonen [S1]. It is widely used in data visualization and clustering. In volume rendering, it is adapted to structure millions of Gaussians or voxel signatures for transfer function design, overcoming limitations of standard grid-sorting algorithms [S1].

## Sources
[S1] Compact 3D Scene Representation via Self-Organizing Gaussian Grids, https://doi.org/10.1007/978-3-031-73013-9_2, fetched 2026-10-26
