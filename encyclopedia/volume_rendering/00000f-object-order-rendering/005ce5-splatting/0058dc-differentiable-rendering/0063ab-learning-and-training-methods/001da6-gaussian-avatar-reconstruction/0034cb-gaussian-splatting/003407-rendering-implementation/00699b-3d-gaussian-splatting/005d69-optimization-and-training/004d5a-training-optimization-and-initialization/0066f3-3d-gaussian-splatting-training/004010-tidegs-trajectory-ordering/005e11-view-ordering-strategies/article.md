# View Ordering Strategies

Also called: Camera Order, GS Count Order, TSP Order

## Overview
View ordering strategies determine the sequence in which training views are processed during 3D Gaussian Splatting optimization to improve training efficiency and stability [S1]. Unlike randomized shuffling, these methods leverage geometric or visibility information to reduce communication overhead between CPU and GPU memory [S1]. They are primarily used in conjunction with Gaussian caching mechanisms to minimize redundant data transfers [S1].

## How it works
**Inputs:** A set of training views $V = \{v_1, \dots, v_N\}$, their camera poses, and the current set of 3D Gaussians $G$.
**Outputs:** A permutation $\pi$ of view indices representing the processing order.

The strategies operate by sorting the views based on specific criteria:

1.  **Random Order:** Views are shuffled uniformly at random. This serves as the baseline for comparison [S1].
2.  **Camera Order:** Views are sorted by the coordinate of their camera center along the scene’s principal axis [S1]. This creates a spatially coherent traversal path [S1].
3.  **GS Count Order:** Views are sorted in descending order by the number of Gaussians visible in each view [S1]. Prioritizing views with higher Gaussian counts allows the CPU Adam optimizer to update a larger number of Gaussians earlier in the training process, thereby reducing its relative overhead [S1].
4.  **TSP Order:** Views are ordered using a Traveling Salesperson Problem (TSP) heuristic to find a sequence that maximizes the overlap of visible Gaussians between successive views [S1]. This strategy is specifically designed to enhance the effectiveness of Gaussian caching by keeping frequently accessed Gaussians resident in GPU memory [S1].

**Edge Cases and Pitfalls:**
*   **Visibility Computation:** GS Count Order and TSP Order require computing view-Gaussian visibility, which adds preprocessing overhead [S1].
*   **Scene Scale:** In scenes with very low Gaussian sharing between images (low $\rho$), such as "BigCity," the benefits of TSP-based caching are reduced because fewer Gaussians are shared between successive views [S1]. Conversely, in scenes with high Gaussian density per image, such as "Bicycle," caching benefits are more significant [S1].

## Subtopics

### Gaussian Caching & Order Optimization
This is a combined optimization technique used in CLM that caches Gaussian parameters across views and applies TSP-based ordering to minimize redundant CPU-to-GPU transfers [S1]. By maintaining a cache of recently accessed Gaussians, the system avoids reloading them from CPU memory. When combined with TSP order, CLM reduces communication volume by 37% (in the Alameda scene) to 82% (in the BigCity scene) compared to naive offloading without caching [S1]. In scenes with low inter-image Gaussian sharing, the caching benefit is smaller, but the ordering still contributes to overall efficiency [S1].

## Operating characteristics
*   **Communication Reduction:** CLM with Gaussian Caching and TSP Order significantly reduces the volume of parameters transferred from CPU to GPU memory compared to naive offloading [S1].
*   **Overhead:** GS Count Order and TSP Order require additional processing to compute visibility or solve the TSP problem, unlike Camera Order or Random Order [S1].
*   **Performance:** The choice of ordering affects training throughput. TSP Order is shown to improve throughput on hardware such as RTX 2080 Ti and RTX 4090 when used with CLM [S1].
*   **Scalability:** The effectiveness of caching-dependent strategies (TSP) scales with the degree of Gaussian overlap between views; it is less effective in sparse or highly fragmented scenes [S1].

## Background
These strategies were introduced in the context of CLM (CLM: Removing the GPU Memory Barrier for 3D Gaussian Splatting) to address the bottleneck of transferring Gaussian parameters between CPU and GPU memory during training [S1]. The ablation study in the source paper evaluates these ordering methods to isolate the impact of view sequence on communication volume and training speed [S1].

## Sources
[S1] CLM: Removing the GPU Memory Barrier for 3D Gaussian Splatting, https://doi.org/10.1145/3779212.3790140, fetched 2026-10-31
