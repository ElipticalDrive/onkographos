# MIMO-NeRF-self-R2

Also called: MIMO-NeRF with reduction factors R1=1, R2=1

## Overview
MIMO-NeRF-self-R2 is a specific configuration of the MIMO-NeRF architecture [MIMO-NeRF](13765-mixture-of-experts-implicit-neural-representation.md) that utilizes two reformulated Multi-Input Multi-Output (MIMO) Multi-Layer Perceptrons (MLPs) with reduction factors $R^1=1$ and $R^2=1$. It solves the problem of high computational cost in standard NeRF training by grouping rays and processing them in parallel through shared MLPs, thereby reducing the total number of MLP evaluations per training step. It sets itself apart from the base MIMO-NeRF [MIMO-NeRF](13765-mixture-of-experts-implicit-neural-representation.md) by employing a specific self-grouping strategy with fixed reduction factors, and from its sibling MIMO-NeRF-self-R1 [MIMO-NeRF-self-R1](13765-mixture-of-experts-implicit-neural-representation.md) by using a different distribution of reduction factors across the two MLP stages.

## How it works
MIMO-NeRF-self-R2 builds on [MIMO-NeRF](13765-mixture-of-experts-implicit-neural-representation.md) by defining the reduction factors for its two-stage reformulated MIMO MLPs.

**Inputs and Outputs:**
*   **Inputs:** A batch of rays $\mathcal{R}$, each defined by origin $\mathbf{o}$ and direction $\mathbf{d}$. The batch is partitioned into $N_p$ groups.
*   **Outputs:** Rendered color images for the batch.

**Data Structures:**
*   **Reduction Factors:** Two scalar values $R^1$ and $R^2$. For MIMO-NeRF-self-R2, $R^1 = 1$ and $R^2 = 1$ [S1].
*   **Grouping:** Rays are grouped into $N_p$ sets. The number of rays per group is $N/N_p$, where $N$ is the total number of rays in the batch.

**Number of MLPs:**
The total number of MLPs running in a single step is calculated as [S1]:
$$
\sum_{m=1}^2 \frac{N}{N_p} R^m = N \frac{2}{N_p} < N \text{ when } N_p > 2.
$$
This indicates that the method reduces the computational load compared to the standard NeRF which requires $N$ MLP evaluations (one per ray), provided $N_p > 2$ [S1].

**Parameters:**
*   $N_p$: The number of ray groups. Typical values cited are 4 and 8 [S1].
*   $R^1, R^2$: Reduction factors, fixed at 1 for this variant [S1].

**Edge Cases and Pitfalls:**
*   The inequality $N \frac{2}{N_p} < N$ holds only when $N_p > 2$. If $N_p \le 2$, the computational advantage in terms of MLP count diminishes or disappears [S1].
*   The method assumes that grouping rays does not significantly degrade rendering quality due to the "near points" issue, where searching for near points across different rays is non-trivial because points are sampled unevenly via hierarchical sampling [S1].

## Subtopics

### MIMO-NeRF-self-R5
A variant using two reformulated MIMO MLPs with reduction factors $R^1=1$ and $R^2=N_p/2$ [S1]. The total number of MLPs running is $N \left( \frac{1}{2} + \frac{1}{N_p} \right) < N$ when $N_p > 2$ [S1]. On the Blender dataset with $N_p=4$, it achieves PSNR 29.81, SSIM 0.939, LPIPS 0.076, and T-time 4.38h. On the LLFF dataset with $N_p=4$, it achieves PSNR 26.61, SSIM 0.838, LPIPS 0.202, and T-time 3.37h [S1]. This method is identical to MIMO-NeRF-self-R1 when $N_p=4$ [S1].

### MIMO-NeRF-self-R4
A variant adopting two reformulated MIMO MLPs with reduction factors $R^1=1$ and $R^2=2$ [S1]. The total number of MLPs running is $N \frac{3}{N_p} < N$ when $N_p > 3$ [S1]. On the Blender dataset with $N_p=4$, it achieves PSNR 29.86, SSIM 0.939, LPIPS 0.077, and T-time 3.33h. On the LLFF dataset with $N_p=4$, it achieves PSNR 26.61, SSIM 0.836, LPIPS 0.205, and T-time 2.41h [S1]. This method is identical to MIMO-NeRF-self-R1 when $N_p=4$ [S1].

### MIMO-NeRF-self-R6
A variant using three reformulated MIMO MLPs with reduction factors $R^1=1, R^2=1, R^3=1$ [S1]. The total number of MLPs running is $N \frac{3}{N_p} < N$ when $N_p > 3$ [S1]. On the Blender dataset with $N_p=4$, it achieves PSNR 30.11, SSIM 0.941, LPIPS 0.074, and T-time 3.76h. On the LLFF dataset with $N_p=4$, it achieves PSNR 26.41, SSIM 0.830, LPIPS 0.208, and T-time 2.73h [S1]. This method is identical to MIMO-NeRF-self-R3 when $N_p=4$ [S1].

## Operating characteristics
**Time and Memory Complexity:**
The method reduces the number of MLP evaluations from $N$ to $N \frac{2}{N_p}$ [S1]. This leads to faster training times compared to the original NeRF. For example, on the Blender dataset with $N_p=4$, training time is 3.65h compared to NeRF's 4.70h [S1]. On the LLFF dataset with $N_p=4$, training time is 2.70h compared to NeRF's 3.39h [S1].

**Measured Performance:**
*   **Blender ($N_p=4$):** PSNR 30.95, SSIM 0.950, LPIPS 0.060, T-time 3.65h [S1].
*   **LLFF ($N_p=4$):** PSNR 27.35, SSIM 0.861, LPIPS 0.169, T-time 2.70h [S1].
*   **Blender ($N_p=8$):** PSNR 30.02, SSIM 0.940, LPIPS 0.076, T-time 2.61h [S1].
*   **LLFF ($N_p=8$):** PSNR 26.52, SSIM 0.833, LPIPS 0.207, T-time 2.13h [S1].

**Scaling:**
Increasing $N_p$ from 4 to 8 reduces training time further (e.g., Blender T-time from 3.65h to 2.61h) but slightly decreases accuracy (PSNR from 30.95 to 30.02) [S1].

**Strengths and Limitations:**
*   **Strengths:** Faster training than original NeRF while maintaining comparable or better performance than some other MIMO variants (e.g., MIMO-NeRF-distill) [S1].
*   **Limitations:** Performance is generally outperformed by MIMO-NeRF-self-R1 in most cases, although R2 achieves comparable performance to MIMO-NeRF-distill [S1]. The grouping strategy may introduce errors if near points are not handled correctly across different rays [S1].

**When to use:**
Use MIMO-NeRF-self-R2 when a balance between training speed and accuracy is desired, and when $N_p > 2$ to ensure computational efficiency. It is particularly suitable when $N_p=4$ or $N_p=8$ as these are the tested configurations [S1].

## Background
MIMO-NeRF-self-R2 was introduced in the paper "MIMO-NeRF: Fast Neural Rendering with Multi-input Multi-output Neural Radiance Fields" [S1]. It is part of the family of MIMO-NeRF variants that explore different reduction factors for reformulated MIMO MLPs to optimize training speed and rendering quality [S1].

## Sources
[S1] MIMO-NeRF: Fast Neural Rendering with Multi-input Multi-output Neural Radiance Fields, https://arxiv.org/abs/2310.01821v1, fetched 2026-10-26
