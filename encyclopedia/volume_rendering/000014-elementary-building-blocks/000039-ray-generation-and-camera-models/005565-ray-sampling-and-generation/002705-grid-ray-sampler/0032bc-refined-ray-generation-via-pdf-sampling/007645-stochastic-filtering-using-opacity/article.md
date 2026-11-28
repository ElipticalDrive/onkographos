# Stochastic Filtering Using Opacity

Also called: Importance Sampling Filtering

## Overview
Stochastic Filtering Using Opacity is a ray sampling refinement technique used in Explicit Radiance Field (ERF) reconstruction to reduce computational cost during optimization. It filters a pre-sampled set of ray points by querying the Sparse Voxel Octree (SVO) for opacity estimates, then applies importance sampling to retain only the samples most likely to be near true scene surfaces [S1, unofficial]. It differs from [CDF-based Ray Resampling](20642-cdf-based-ray-resampling.md) by using a simple additive weight model ($c + o$) rather than cumulative distribution functions, and from its parent [Refined Ray Generation via PDF Sampling](6837-refined-ray-generation-via-pdf-sampling.md) by operating on a fixed budget of previously stratified samples rather than generating new samples from a PDF.

## How it works
**Inputs:**
- A set of limited ray sampling points $s'_{r,i}$ for ray $r$, resulting from a previous stratified sampling step.
- The current SVO geometry estimate.
- A user-defined constant $c$.

**Outputs:**
- A further reduced set of ray sampling points $s''_{r,i}$ with count $N_{max,r} < N_{max}$.

**Data Structures:**
- **SVO Opacity Query:** The SVO stores opacity values. Access requires 4D interpolation (spatial $x,y,z$ and temporal $t$) as defined in Eq. 5 of the source [S1, unofficial]. The exact interpolation kernel is not detailed in the provided notes, but standard trilinear or higher-order interpolation over the voxel grid is implied by "4D interpolation scheme".
- **Sampling Weights:** A scalar weight $w_{s,r}(t_{i,k})$ is computed for each sample point at depth $t_{i,k}$.

**Steps:**
1. **Opacity Query:** For each sample point $t_{i,k}$ in the limited set $s'_{r,i}$, query the SVO to obtain the opacity sample $o(t_{i,k})$ using 4D interpolation [S1, unofficial].
2. **Weight Calculation:** Compute the sampling weight for each point using the formula:
   $$w_{s,r}(t_{i,k}) = c + o(t_{i,k})$$
   where $c$ is a constant and $o(t_{i,k})$ is the queried opacity [S1, unofficial].
3. **Importance Sampling:** Use the weights $w_{s,r}(t_{i,k})$ to perform importance sampling. Select a subset of points $s''_{r,i} \subset s'_{r,i}$ such that the final count $N_{max,r}$ does not exceed the secondary budget limit [S1, unofficial]. Points with higher opacity (closer to surfaces) are preferentially retained.
4. **Radiance Retrieval:** Use the final reduced set $s''_{r,i}$ to retrieve outgoing SLF radiance samples from the SVO [S1, unofficial].

**Parameters:**
- **Constant $c$:** Set to $0.05$. This ensures that the entire ray is sampled at least infrequently, handling intermediate false free space regions where opacity is low but gradients are still needed [S1, unofficial].
- **Sample Budgets:**
  - $N_{max}$: Maximum samples per ray after initial stratified sampling. Typical value: $256$ [S1, unofficial].
  - $N_{max,r}$: Maximum samples per ray after stochastic filtering. Typical value: $32$ [S1, unofficial].
  - Samples per node edge length $\alpha$: Set to $8$ [S1, unofficial].

**Pitfalls:**
- **Gradient Noise:** Randomly skipping sampling points induces noise on loss gradients. However, the optimizer (Adam/SGD) is robust to this noise [S1, unofficial].
- **False Occluders:** This method helps avoid false occluders consistently blocking true surface points, which would otherwise result in missing gradients for correct reconstruction [S1, unofficial].
- **Optimization Only:** This stochastic limiting is primarily required to limit gradient computation costs during optimization. It is optional for final rendering once the SVO is built, as no loss gradients are computed and SVO nodes tightly bound observed surfaces [S1, unofficial].

## Subtopics

### Stochastic Filtering of Ray Sampling Points
This variant refers to the initial random subsampling step that occurs before opacity filtering. It randomly selects a subset of stratified samples to fit within a budget $N_{max}$. It exploits the fact that Adam tracks gradient histories, allowing dense ray sampling to be deferred over multiple iterations. Skipping points randomly induces gradient noise, but SGD is robust against it. Crucially, it prevents false occluders from consistently occluding true surface points, which would cause missing gradients. This step precedes the opacity-based filtering described in the main method [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Dominated by SVO opacity queries (4D interpolation) and the sorting/selection step for importance sampling. The reduction from $N_{max}$ to $N_{max,r}$ significantly reduces subsequent radiance retrieval costs.
- **Memory Complexity:** Requires storing weights for all intermediate samples before selection. The use of a capped budget $N_{max}$ prevents unbounded memory growth on GPU [S1, unofficial].
- **Performance:** Reduces the number of samples used for gradient computation from 256 to 32 per ray in typical configurations, lowering optimization cost [S1, unofficial].
- **Strengths:** Efficiently focuses computational resources on surface-proximate regions. Handles false free space via the constant $c$.
- **Limitations:** Introduces gradient noise. Not suitable for final rendering where full sampling may be preferred for quality, though the source notes it is optional in that phase [S1, unofficial].

## Background
The method originates from the Explicit Radiance Field (ERF) reconstruction framework. It was introduced to manage the high computational cost of gradient-based optimization in neural radiance fields by using explicit voxel structures (SVO) and stochastic sampling strategies. The approach leverages the properties of the Adam optimizer to tolerate noisy gradients from subsampled rays [S1, unofficial].

## Sources
[S1] ERF: Explicit Radiance Field Reconstruction From Scratch, https://arxiv.org/abs/2203.00051v1, fetched 2026-10-26 (unofficial)
