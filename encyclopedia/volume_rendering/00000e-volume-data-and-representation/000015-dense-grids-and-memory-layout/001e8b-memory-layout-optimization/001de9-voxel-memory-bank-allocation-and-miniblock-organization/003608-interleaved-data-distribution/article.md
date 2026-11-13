# Interleaved Data Distribution
Also called: Page-based Round-robin Distribution
## Overview
Interleaved Data Distribution is a memory layout strategy for shared-memory MIMD architectures that distributes pages of a voxel array in round-robin fashion among local memories to avoid hot spotting [S1]. It sets itself apart from its siblings by prioritizing load balancing and cache coherence across processors over the conflict-free access patterns of [Skewed Memory Organization](2351-skewed-memory-organization-for-conflict-free-ray-access.md) or the specific bank-avoidance of [Cubic Addressing Scheme](7285-cubic-addressing-scheme.md).
## How it works
**Inputs and Outputs:**
- Input: A 3D voxel dataset of size $N_x \times N_y \times N_z$ with $P$ processors.
- Output: A mapping of voxel data to $P$ local memory banks.

**Data Structures:**
The voxel data is stored in a linearized array structure. The array is divided into fixed-size pages. Let $M$ be the number of pages. Each page $p$ (where $0 \le p < M$) is assigned to a processor.

**Steps:**
1. Linearize the 3D voxel grid into a 1D array $V$ using a standard ordering (e.g., row-major or Z-order).
2. Divide $V$ into contiguous pages of size $S_{page}$.
3. Assign page $p$ to processor $i$ using the formula:
   $$ i = p \mod P $$
   where $P$ is the total number of processors.
4. Each processor stores its assigned pages in its local memory.

**Parameters:**
- $S_{page}$: Page size. Not given in the sources.
- $P$: Number of processors.

**Edge Cases and Pitfalls:**
- If the number of pages is not evenly divisible by $P$, some processors will hold one more page than others.
- Hot spotting is avoided by ensuring that adjacent spatial regions (which may be accessed by different rays) are distributed across different processors, reducing contention on shared memory resources.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def assign_voxels_to_processors(voxel_array, num_processors, page_size):
    """
    Assigns pages of a linearized voxel array to processors in round-robin fashion.
    
    Args:
        voxel_array: List of voxels, linearized.
        num_processors: Number of processors P.
        page_size: Size of each page in voxels.
        
    Returns:
        A dictionary mapping processor_id to list of page indices.
    """
    total_voxels = len(voxel_array)
    num_pages = (total_voxels + page_size - 1) // page_size
    
    # Initialize processor assignments
    processor_pages = {i: [] for i in range(num_processors)}
    
    # Distribute pages in round-robin fashion
    for page_idx in range(num_pages):
        processor_id = page_idx % num_processors
        processor_pages[processor_id].append(page_idx)
        
    return processor_pages
```
## Subtopics

### [Grouped vs Interleaved Macro Distribution](13385-grouped-vs-interleaved-macro-distribution.md)
This method classifies how macro-sized blocks of voxel data are assigned to processing nodes in a distributed memory system. It determines whether a node holds a spatially contiguous subset of the dataset (grouped) or a scattered subset (interleaved/random). The primary distinction lies in the trade-off between cache coherence and communication overhead during ray casting resampling.

## Operating characteristics
- **Time Complexity:** Assignment is $O(P)$ or $O(M)$ depending on implementation, where $M$ is the number of pages. Access time is $O(1)$ for local memory, but remote access may occur if the architecture allows.
- **Memory Complexity:** $O(N)$ for storing the voxel data, distributed across $P$ processors.
- **Performance:** Reduces hot spotting and improves load balancing in shared-memory systems [S1].
- **Strengths:** Simple to implement, effective for avoiding memory contention in shared-memory MIMD architectures [S1].
- **Limitations:** May lead to increased remote memory access if rays traverse multiple pages assigned to different processors. Not given in the sources for specific hardware metrics.
## Background
The method was proposed for the DASH (Directory Architecture for SHared memory) Multiprocessor, a scalable shared-memory MIMD architecture [S1]. It was introduced to eliminate voxel communication associated with ray tracing by distributing data in an interleaved fashion among local memories to avoid hot spotting [S1].
## Sources
[S1] Volume rendering on scalable shared-memory MIMD architectures, https://doi.org/10.1145/147130.147141, fetched 2026-10-25
