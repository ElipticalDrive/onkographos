# Soft Shadows via Ray Tracing
Also called: Packet-based Soft Shadow Ray Tracing

## Overview
This method computes soft shadows in participating media by generating coherent packets of secondary rays from primary ray hit points toward an area light source [S1]. It distinguishes itself from hard shadow techniques by sampling a spatial distribution of directions to approximate the penumbra, leveraging the coherence of these secondary rays to maintain interactive performance [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Primary ray hit point $\mathbf{p}$, area light source geometry (defined by a set of sample points), and volume data structure.
*   **Output:** Transmittance value $T$ representing the fraction of light reaching $\mathbf{p}$ from the area light.

**Data Structures:**
*   **Shadow Ray Packet:** A collection of $N$ shadow rays $\{\mathbf{r}_0, \mathbf{r}_1, \dots, \mathbf{r}_{N-1}\}$ sharing a common origin $\mathbf{p}$ [S1].
*   **Light Samples:** A set of $N$ points $\{\mathbf{l}_0, \mathbf{l}_1, \dots, \mathbf{l}_{N-1}\}$ distributed over the area light source [S1].

**Steps:**
1.  **Generate Shadow Ray Packet:** For a given primary ray hit point $\mathbf{p}$, generate $N$ shadow rays. Each ray $\mathbf{r}_i$ originates at $\mathbf{p}$ and is directed toward a corresponding sample point $\mathbf{l}_i$ on the area light source [S1]. The direction vector is $\mathbf{d}_i = \mathbf{l}_i - \mathbf{p}$ [G].
2.  **Traverse Volume:** Traverse the volume data structure along each ray $\mathbf{r}_i$ in the packet. Since all rays share the common origin $\mathbf{p}$, they can be traversed in a manner identical to primary ray packets, exploiting spatial coherence [S1].
3.  **Evaluate Transmittance:** For each ray $\mathbf{r}_i$, compute the transmittance $T_i$ by integrating the extinction coefficient $\sigma_t$ along the path from $\mathbf{p}$ to $\mathbf{l}_i$ [G].
    $$ T_i = \exp\left(-\int_{\mathbf{p}}^{\mathbf{l}_i} \sigma_t(\mathbf{x}) \, d\mathbf{x}\right) $$
    In discrete implementations, this is approximated by summing $\sigma_t \cdot \Delta s$ over samples along the ray [G].
4.  **Aggregate Result:** The final soft shadow factor is the average of the transmittances of all rays in the packet:
    $$ T_{soft} = \frac{1}{N} \sum_{i=0}^{N-1} T_i $$
    This average approximates the integral of light over the area light source [G].

**Parameters:**
*   **Number of shadow rays ($N$):** Controls the quality of the penumbra approximation. Higher $N$ yields smoother shadows but increases computational cost [S1].
*   **Light source size:** The spatial distribution of sample points $\mathbf{l}_i$ defines the effective size of the area light. Larger sources produce softer shadows [S1].

**Edge Cases and Pitfalls:**
*   **Small $N$:** Results in noisy or aliased penumbra edges [G].
*   **Large $N$:** Increases memory bandwidth and traversal overhead due to packet size [G].
*   **Degenerate Light:** If the light source is a point light ($N=1$), the method degrades to hard shadow casting [G].

## Subtopics

### [Soft Shadows via Ray Packets](5179-soft-shadows-via-ray-packets.md)
Extends this method by optimizing the traversal of coherent ray packets for further performance gains in large-scale volumes.

## Operating characteristics
*   **Time Complexity:** $O(N \cdot K)$, where $N$ is the number of shadow rays per packet and $K$ is the number of volume samples traversed per ray.
*   **Memory Complexity:** $O(N)$ for storing the ray packet directions and intermediate transmittance values.
*   **Performance:** Interactive performance is maintained by leveraging the coherence of shadow rays sharing a common origin, allowing for efficient batched traversal [S1].
*   **Strengths:** Easily integrated into ray tracing frameworks; supports arbitrary area light shapes; allows interactive control over quality vs. performance [S1].
*   **Limitations:** Computationally expensive for high-quality soft shadows due to the need for multiple ray traversals per pixel [G].

## Background
This approach was introduced in the context of coherent grid traversal for particle visualization, where soft shadows were needed to avoid misinterpreting shadowed regions as data discontinuities [S1]. It leverages the inherent coherence of ray packets to make soft shadow computation feasible in real-time or near-real-time volume rendering applications.

## Sources
[S1] A Coherent Grid Traversal Approach to Visualizing Particle ..., https://www.sci.utah.edu/~thiago/papers/pcgt_revised.pdf, fetched 2026-10-25
