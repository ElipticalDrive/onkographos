# Domain Decomposition Scene Representation

Also called: Domain Decomposition of Models, Model-Parallel Domain Decomposition

## Overview
Domain Decomposition Scene Representation is a model-parallel technique for representing large-scale volumetric data by partitioning the spatial domain into fixed-size bricks, each modeled by a dedicated neural network [S1]. It solves the memory and I/O bottlenecks of single-network approaches by ensuring each brick fits within GPU memory, eliminating the need for out-of-core sampling [S1]. Unlike single-model approaches, it enables parallel training and inference without inter-process communication, scaling accuracy and speed for massive datasets [S1].

## How it works
**Inputs:** Large-scale volumetric dataset; grid dimensions defining the spatial decomposition [S1].
**Outputs:** A set of independent neural networks (one per brick) and a spatial indexing structure to map query coordinates to the correct brick and network [S1].

**Data Structures:**
1. **Brick Grid:** A 3D grid partitioning the scene, where each cell corresponds to one neural network [S1].
2. **Expert Networks:** A collection of identical neural network architectures, where each network is trained exclusively on data points falling within its assigned brick [S1].
3. **Spatial Index:** A mapping structure that takes a 3D query coordinate and returns the index of the brick containing that point [G].

**Steps:**
1. **Partition Data:** Divide the input dataset into disjoint sets based on spatial coordinates [S1]. Each point is assigned to exactly one brick based on its position in the grid [S1].
2. **Train Experts:** For each brick, train a dedicated neural network using only the data points assigned to that brick [S1]. Training is performed in parallel across multiple GPUs [S1]. No communication occurs between networks during training [S1].
3. **Build Index:** Construct a spatial hash or grid lookup table that maps any 3D coordinate $(x, y, z)$ to the corresponding brick index [G].
4. **Inference Query:** To evaluate the volume at a query point $q$:
   a. Use the spatial index to identify the brick containing $q$ [G].
   b. Pass $q$ to the neural network associated with that brick [S1].
   c. Return the output of that network [S1].

**Formulas:**
Let $B$ be the set of bricks, and $N_b$ be the neural network for brick $b \in B$ [G].
For a query point $q$, let $idx(q)$ be the brick index returned by the spatial index [G].
The volume value $V(q)$ is:
$$ V(q) = N_{idx(q)}(q) $$
[S1]

**Parameters:**
* **Brick Size:** Must be small enough to fit the training data and network weights into GPU memory [S1]. Not given in the sources.
* **Grid Resolution:** Determined by the scene size and desired brick granularity [G]. Not given in the sources.

**Edge Cases and Pitfalls:**
* **Boundary Points:** Points lying exactly on brick boundaries must be assigned to exactly one brick to avoid ambiguity or double-counting during training [G].
* **Empty Bricks:** Bricks with no data points may result in untrained networks [G].
* **Indexing Overhead:** The spatial hash lookup must be efficient to maintain real-time performance [G].

## Subtopics

### [Domain Decomposition Training and Inference](11132-domain-decomposition-training-and-inference.md)
A model-parallel strategy for fitting large-scale volumes by partitioning the data into a grid of bricks, where one SRN is trained per brick in parallel on multiple GPUs. It avoids out-of-core I/O overhead by ensuring each brick fits in GPU memory and accelerates inference using spatial hashing to map query points to the correct model. It is distinguished from single-model approaches by enabling the reconstruction of volumes (e.g., 450 GB) in under 7 minutes on 8 GPUs while increasing accuracy over a single model with the same total parameter count.

## Operating characteristics
* **Time Complexity:** Training is $O(N)$ where $N$ is the total number of data points, but parallelized across bricks [G]. Inference is $O(1)$ for spatial lookup plus the cost of one network evaluation [G].
* **Memory Complexity:** Total memory is the sum of memory for all expert networks plus the spatial index [S1]. Each brick's data and network fit in GPU memory, avoiding out-of-core constraints [S1].
* **Measured Performance:** Reconstructed a 450 GB volume in under 7 minutes on 8 GPUs [S1].
* **Strengths:** Eliminates out-of-core I/O bottlenecks; enables parallel training; scales to very large datasets; higher accuracy than single models with equivalent parameter counts [S1].
* **Limitations:** Requires careful partitioning; potential discontinuities at brick boundaries [G]; overhead of managing multiple networks [G].
* **When to Use:** Large-scale volumetric data that exceeds single-GPU memory capacity [S1].
* **When Not to Use:** Small datasets where a single network suffices; scenes requiring high continuity across arbitrary boundaries without post-processing [G].

## Background
Originated in the context of large-scale data visualization, specifically to address the limitations of single-network neural volume representations [S1]. The approach was proposed by Wu et al. in "Adaptively Placed Multi-Grid Scene Representation Networks for Large-Scale Data Visualization" (2023) [S1]. It builds on the concept of sparsely-gated Mixture-of-Experts but applies it to spatial domain decomposition without gating, relying on strict spatial partitioning [G].

## Sources
[S1] Adaptively Placed Multi-Grid Scene Representation Networks for Large-Scale Data Visualization, https://doi.org/10.1109/tvcg.2023.3327194, fetched 2026-10-26
