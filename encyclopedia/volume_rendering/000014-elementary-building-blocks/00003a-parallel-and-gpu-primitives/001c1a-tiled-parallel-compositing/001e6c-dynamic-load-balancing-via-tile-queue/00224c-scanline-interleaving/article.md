# Scanline Interleaving

Also called: Image Partitioning for Load Balancing, Image partitioning strategy, Pixel Interleaving, Scanline Interleaving Image Partitioning

## Overview
Scanline Interleaving is an image-space partitioning strategy that distributes pixels across processors in a striped, round-robin pattern to balance the computational load of ray-merging operations [S1, S2, unofficial][S3]. It addresses the problem of uneven depth complexity caused by local variations in cell sizes within unstructured meshes, which would otherwise cause load imbalance if pixels were assigned to single processors [S2, unofficial][S3]. It is distinguished from sibling methods like [Load-Balanced Cell Assignment](4194-load-balanced-cell-assignment.md) by operating on the image plane rather than the volume data space, and from [Farm on Demand (FD)](7455-farm-on-demand-fd.md) by its static, cyclic assignment rather than dynamic task queuing.

## How it works
**Inputs:**
- Image dimensions: width $W$, height $H$ (number of scanlines).
- Number of processors: $P$.
- Per-pixel data structure: A linked list of ray segments maintained at each pixel location, which are merged to form the final pixel value [S3].

**Outputs:**
- Assignment map: For each pixel $(x, y)$, the ID of the processor responsible for merging its ray segments.

**Data Structures:**
- `processor_id`: Integer in range $[0, P-1]$.
- `pixel_list`: Linked list of ray segments at each $(x, y)$ [S3].

**Steps:**
1. For each scanline $y$ in the image, where $0 \le y < H$:
2.   Calculate the processor ID for this scanline using the formula:
       $$ \text{processor\_id} = y \pmod P $$
3.   Assign all pixels $(x, y)$ for $0 \le x < W$ to the calculated `processor_id`.
4.   Each processor collects the ray segments routed to its assigned pixels from the volume data [S1, unofficial].
5.   Each processor merges the ray segments in its assigned pixels to produce the final color and opacity values [S3].

**Parameters:**
- $P$: Number of processors.
- $H$: Vertical resolution.

**Edge Cases and Pitfalls:**
- If $H$ is not significantly larger than $P$, load balancing degrades because the granularity of the partitioning is too coarse [S2, unofficial][S3].
- When using a large number of processors ($P$ is large relative to $H$), scanline interleaving may fail to distribute high-density regions effectively. In such cases, finer-grained pixel interleaving (e.g., assigning individual pixels or small blocks in a round-robin fashion) is required to improve scalability and load balancing [S2, unofficial][S3].
- The method assumes that the vertical distribution of depth complexity is relatively uniform. If complexity is clustered in specific horizontal bands, scanline interleaving may still result in imbalance [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def assign_scanlines(width, height, num_processors):
    """
    Assigns each pixel in the image to a processor using scanline interleaving.
    
    Args:
        width (int): Number of columns (x-resolution).
        height (int): Number of rows (y-resolution).
        num_processors (int): Total number of processors P.
        
    Returns:
        dict: A dictionary mapping (x, y) coordinates to processor IDs.
    """
    assignment = {}
    for y in range(height):
        # Round-robin assignment of scanlines to processors
        processor_id = y % num_processors
        for x in range(width):
            assignment[(x, y)] = processor_id
    return assignment
```

## Subtopics

### Pixel Interleaving
A finer-grained variant where individual pixels (or small blocks) are assigned in round-robin order rather than entire scanlines. It is chosen when the number of processors is large relative to the image height, or when depth complexity is highly localized. It offers better load balancing at the cost of increased communication overhead for small pixel blocks. See [Scanline Interleaving](#scanline-interleaving) for the base method.

### Hybrid Image Partitioning
Combines scanline or pixel interleaving with other strategies to handle varying scene complexities. It is chosen when a single static partitioning scheme fails to balance load across diverse views. See [Hybrid Image Partitioning](7675-hybrid-image-partending.md).

## Operating characteristics
- **Time Complexity**: $O(W \times H)$ for the assignment phase, where $W$ and $H$ are image dimensions. The merging phase is proportional to the number of ray segments per pixel, which is balanced across processors by the partitioning.
- **Memory Complexity**: $O(W \times H)$ to store the assignment map and per-pixel ray segment lists [S3].
- **Measured Performance**: Pixel interleaving (a finer-grained variant) speeds up overall rendering by nearly 10 percent on large systems compared to coarser partitioning [S2, unofficial].
- **Strengths**: Simple to implement; effective at dispersing ray-merging operations when vertical resolution is high relative to the number of processors [S2, unofficial][S3].
- **Limitations**: Performance degrades if the vertical resolution is not several times larger than the number of processors [S2, unofficial][S3]. It does not account for horizontal variations in depth complexity.
- **When to Use**: When $H \gg P$ and depth complexity is relatively uniform across horizontal bands.
- **When Not to Use**: When $P$ is large relative to $H$, or when depth complexity is clustered in specific horizontal regions. Use Pixel Interleaving or [Hybrid Image Partitioning](7675-hybrid-image-partending.md) instead.

## Background
Scanline Interleaving was developed as part of a scalable parallel cell-projection volume rendering algorithm for three-dimensional unstructured data [S1, unofficial][S3]. It was introduced in the late 1990s by researchers including Ma and colleagues, who sought to balance the load in parallel rendering by distributing image-space operations [S1, unofficial][S3]. The method is used in conjunction with cell-projection rendering algorithms, where cells are scan-converted into ray segments and routed to image space for merging [S1, unofficial][S3]. It addresses the challenge of local variations in cell sizes leading to variations in depth complexity in image space [S2, unofficial][S3].

## Sources
[S1] DTIC ADA329328: A Scalable Parallel Cell-Projection Volume Rendering Algorithm for Three-Dimensional Unstructured Data, https://archive.org/details/DTIC_ADA329328, fetched 2026-10-26 (unofficial)
[S2] Parallel Software Rendering for Large-Scale Volume Visualization, https://escholarship.org/uc/item/489126f2, fetched 2026-10-27 (unofficial)
[S3] PDF A Scalable Parallel Cell-Projection Volume Rendering Algorithm for ..., https://web.cs.ucdavis.edu/~ma/papers/PRS97.pdf, fetched 2026-10-27
