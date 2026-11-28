# MIMO-NeRF-self-R2


## Overview
MIMO-NeRF-self-R2 is a specific configuration of the MIMO-NeRF architecture that utilizes two reformulated Multi-Input Multi-Output (MIMO) Multi-Layer Perceptrons (MLPs) with reduction factors $R^1=1$ and $R^2=1$ [S1]. It solves the problem of high computational cost in Neural Radiance Fields (NeRF) by parallelizing the evaluation of multiple points along rays within a single forward pass, thereby reducing training time while maintaining rendering quality [S1]. It sets itself apart from the base MIMO-NeRF by employing a specific "self" reformulation strategy that balances the number of parallel MLPs against the reduction factor to optimize throughput [S1].

## How it works
MIMO-NeRF-self-R2 builds on [Mixture-of-Experts and Modular Architectures](14429-mixture-of-experts-and-modular-architectures.md) by replacing the standard sequential ray-point evaluation with a batched MIMO approach. The method groups points from multiple rays into batches to be processed by parallel MLP instances.

**Inputs and Outputs:**
*   **Input:** A batch of rays, where each ray contains $N$ sampled points. The parameter $N_p$ denotes the number of parallel MLP instances (or processing units) available.
*   **Output:** Radiance and density values for all sampled points, reconstructed from the parallel MLP outputs.

**Data Structures:**
The implementation relies on the reformulated MIMO MLP structure. While the exact internal tensor layout of the MIMO MLP is defined in the parent MIMO-NeRF article, the self-R2 configuration specifically instantiates **two** such MLPs [S1].

**Parameters:**
*   $N_p$: The number of parallel processing units (MLP instances). Typical values cited are $N_p=4$ and $N_p=8$ [S1].
*   $R^m$: The reduction factor for the $m$-th MLP. For MIMO-NeRF-self-R2, $R^1 = 1$ and $R^2 = 1$ [S1].
*   $N$: The total number of points sampled per ray (or total points in the batch context relative to the formula) [S1].

**Algorithm Steps:**
1.  **Configuration:** Initialize two reformulated MIMO MLPs, denoted as $M_1$ and $M_2$.
2.  **Reduction Factor Assignment:** Assign reduction factors $R^1 = 1$ and $R^2 = 1$ to $M_1$ and $M_2$ respectively [S1].
3.  **Batching/Dispatch:** Distribute the input points across the two MLPs. The total number of MLPs running concurrently is determined by the sum of the capacities defined by the reduction factors [S1]:
    $$
    \text{Total MLPs} = \sum_{m=1}^2 \frac{N}{N_p} R^m = N \frac{2}{N_p}
    $$
    This configuration is valid and efficient when $N_p > 2$ [S1].
4.  **Forward Pass:** Execute the forward pass of both MIMO MLPs in parallel on their assigned subsets of points.
5.  **Aggregation:** Combine the outputs from both MLPs to form the final radiance and density fields for volume rendering [S1].

**Edge Cases and Pitfalls:**
*   **Constraint on $N_p$:** The formula $\sum_{m=1}^2 \frac{N}{N_p} R^m < N$ holds only when $N_p > 2$. If $N_p \le 2$, the parallelization advantage or the specific mathematical bound described may not hold as intended [S1].
*   **Padding:** When group shifts are conducted, padding is performed. The number of group shifts is equal to or smaller than the number of reformulated MIMO MLPs, adding a small overhead, but the effect is noted as small [S1].

## Subtopics

### MIMO-NeRF-self-R5
A variant using two reformulated MIMO MLPs with reduction factors $R^1=1$ and $R^2=N_p/2$ [S1]. The total number of MLPs running is calculated as $\sum_{m=1}^2 \frac{N}{N_p} R^m = N \left( \frac{1}{2} + \frac{1}{N_p} \right) < N$ when $N_p > 2$ [S1]. On the Blender dataset with $N_p=4$, it achieves PSNR 29.81, SSIM 0.939, LPIPS 0.076, and T-time 4.38h. On the LLFF dataset with $N_p=4$, it achieves PSNR 26.61, SSIM 0.838, LPIPS 0.202, and T-time 3.37h [S1]. This method is identical to MIMO-NeRF-self-R1 when $N_p=4$ [S1].

### MIMO-NeRF-self-R4
A variant adopting two reformulated MIMO MLPs with reduction factors $R^1=1$ and $R^2=2$ [S1]. The total number of MLPs running is calculated as $\sum_{m=1}^2 \frac{N}{N_p} R^m = N \frac{3}{N_p} < N$ when $N_p > 3$ [S1]. On the Blender dataset with $N_p=4$, it achieves PSNR 29.86, SSIM 0.939, LPIPS 0.077, and T-time 3.33h. On the LLFF dataset with $N_p=4$, it achieves PSNR 26.61, SSIM 0.836, LPIPS 0.205, and T-time 2.41h [S1]. This method is identical to MIMO-NeRF-self-R1 when $N_p=4$ [S1].

### MIMO-NeRF-self-R6
A variant using three reformulated MIMO MLPs with reduction factors $R^1=1, R^2=1, R^3=1$ [S1]. The total number of MLPs running is calculated as $\sum_{m=1}^3 \frac{N}{N_p} R^m = N \frac{3}{N_p} < N$ when $N_p > 3$ [S1]. On the Blender dataset with $N_p=4$, it achieves PSNR 30.11, SSIM 0.941, LPIPS 0.074, and T-time 3.76h. On the LLFF dataset with $N_p=4$, it achieves PSNR 26.41, SSIM 0.830, LPIPS 0.208, and T-time 2.73h [S1]. This method is identical to MIMO-NeRF-self-R3 when $N_p=4$ [S1].

## Operating characteristics
**Time and Memory Complexity:**
The method reduces training time compared to the original NeRF by parallelizing MLP evaluations. The total number of MLPs running is $N \frac{2}{N_p}$, which is less than $N$ when $N_p > 2$ [S1].

**Measured Performance:**
*   **Blender Dataset ($N_p=4$):** PSNR 30.95, SSIM 0.950, LPIPS 0.060, Training Time 3.65h [S1].
*   **LLFF Dataset ($N_p=4$):** PSNR 27.35, SSIM 0.861, LPIPS 0.169, Training Time 2.70h [S1].
*   **Blender Dataset ($N_p=8$):** PSNR 30.02, SSIM 0.940, LPIPS 0.076, Training Time 2.61h [S1].
*   **LLFF Dataset ($N_p=8$):** PSNR 26.52, SSIM 0.833, LPIPS 0.207, Training Time 2.13h [S1]

**Scaling and Trade-offs:**
MIMO-NeRF-self-R2 achieves performance comparable to MIMO-NeRF-distill while being faster than the original NeRF [S1]. It is one of the configurations that achieves the best or comparable performance in terms of PSNR, SSIM, and LPIPS among the reformulation methods tested, though MIMO-NeRF-self-R1 is generally preferred for the main experiments [S1].

## Background
MIMO-NeRF-self-R2 was introduced in the paper "MIMO-NeRF: Fast Neural Rendering with Multi-input Multi-output Neural Radiance Fields" [S1]. The paper explores various reformulation methods (R1 through R6) to optimize the training speed of NeRF by leveraging multi-input multi-output neural networks. MIMO-NeRF-self-R2 is one of the specific variants analyzed in Table 6 of the source, demonstrating the effect of changing the reformulation method on rendering quality and training time [S1].

## Sources
[S1] MIMO-NeRF: Fast Neural Rendering with Multi-input Multi-output Neural Radiance Fields, https://arxiv.org/abs/2310.01821v1, fetched 2026-10-26
