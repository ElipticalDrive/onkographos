# Multi-Resolution Multi-Channel Page Table (MRMCPT)
Also called: page table approach

## Overview
Multi-Resolution Multi-Channel Page Table (MRMCPT) is a baseline out-of-core volume rendering method that manages a multi-resolution hierarchy using page tables for spatial subdivision and cache residency [S1]. It relies on explicit page table lookups to translate virtual addresses to physical brick locations, distinguishing it from tree traversal or hash-based spatial indexing [S1]. Its primary trade-off is limited empty-space skipping capabilities compared to octree-based methods, which results in higher cache usage and lower performance on sparse or multi-channel datasets [S1].

## How it works
**Inputs and Outputs:** Not given in the sources. [G]
**Data Structures:** The core structure is a page table mapping virtual brick coordinates to physical memory addresses [S2, unofficial]. Each page table entry stores a physical address and residency/validity flags [G]. The virtual address space represents the logical 3D volume divided into uniform bricks [G].
**Algorithm Steps:**
1. Initialize the page table hierarchy in memory [G].
2. For each pixel, cast a ray and compute the virtual brick index $(x, y, z)$ for the current sample position [G].
3. Perform address translation using the page table [S2, unofficial]:
   - Compute the page table index: `idx = virtAddx // brickSize` [S2, unofficial].
   - Retrieve the entry: `pt_entry = pageTable[idx]` [S2, unofficial].
   - Compute the physical address: `physAddx = pt_entry.physAddx + (virtAddx % brickSize)` [S2, unofficial].
4. Check residency: if the brick is resident, sample the data at `physAddx` [G]; if not, trigger a page fault to load the brick from storage into the brick pool, update the entry, and retry [G].
5. Accumulate color and opacity along the ray [G].
6. Apply culling: skip only entirely empty bricks [S1].
**Parameters:** Not given in the sources for the baseline method. [G]
**Edge Cases and Pitfalls:** The method only skips entirely empty bricks, potentially accessing bricks containing values outside the current visible transfer function range [S1]. This limited culling leads to higher cache usage and lower performance on sparse or multi-channel datasets compared to octrees [S1].
**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_ray(ray_origin, ray_direction, brick_size, page_table):
    t = ray_origin.t_min
    while t < ray_origin.t_max:
        sample_pos = ray_origin + ray_direction * t
        # Linearize 3D voxel coordinate to 1D index for address translation [S2, unofficial]
        virt_addx = sample_pos.x + sample_pos.y * 1024 + sample_pos.z * 1048576
        # Compute page table index [S2, unofficial]
        idx = virt_addx // brick_size
        # Retrieve page table entry [S2, unofficial]
        pt_entry = page_table[idx]
        if pt_entry.is_resident:
            # Compute physical address [S2, unofficial]
            phys_addx = pt_entry.phys_addx + (virt_addx % brick_size)
            color, opacity = sample_brick(phys_addx, sample_pos)
            accumulate(color, opacity)
        else:
            load_brick_into_pool(idx)
            pt_entry.is_resident = True
            pt_entry.phys_addx = get_pool_address(idx)
            phys_addx = pt_entry.phys_addx + (virt_addx % brick_size)
            color, opacity = sample_brick(phys_addx, sample_pos)
            accumulate(color, opacity)
        t += brick_size / ray_direction.magnitude
```

## Subtopics

### Multi-Level Page Tables
Multi-Level Page Tables is a variant introduced by Hadwiger et al. in 2012 [S2, unofficial]. It virtualizes page tables recursively, similar to CPU multi-level page tables, treating pages of page table entries like pages of voxels [S2, unofficial]. This creates a recursive page table hierarchy that is decoupled from the data resolution levels, ensuring the number of page table levels is significantly less than the number of data resolution levels [S2, unofficial].
- **Parameters:** Voxel block size is $32^3$ voxels; page table block size is $32^3$ page table entries [S2, unofficial].
- **Scalability Examples:**
  - For a $32,000 \times 32,000 \times 4,000$ volume (4 TB, 11 resolution levels): 2 page table levels with $32 \times 32 \times 4$ page table blocks [S2, unofficial].
  - For a $128,000 \times 128,000 \times 16,000$ volume (196 TB, 13 resolution levels): 2 page table levels with $128 \times 128 \times 16$ page table blocks [S2, unofficial].
  - For a $512,000 \times 512,000 \times 64,000$ volume (15 PB, 15 resolution levels): 3 page table levels with $16 \times 16 \times 2$ page table blocks [S2, unofficial].
  - For a $2,000,000 \times 2,000,000 \times 250,000$ volume (888 PB, 17 resolution levels): 3 page table levels with $64 \times 64 \times 8$ page table blocks [S2, unofficial].

### Uniform Page Tables
Uniform Page Tables use page tables that are uniform in each level and can be mipmapped [S2, unofficial]. A page table can be created for each resolution level, resulting in a page table mipmap [S2, unofficial]. This approach is only feasible when the page table is not too large; for medium-sized volumes or large page/brick sizes, it requires a full-size page table, which may be impractical [S2, unofficial].

## Operating characteristics
- **Performance:** Ray casting performance is limited by the number of bricks accessed [S1]. On sparse or multi-channel datasets, MRMCPT performs worse than octree-based approaches due to its inability to perform fine-grained empty-space skipping [S1]. It requires more bricks to be resident in the cache compared to residency octrees [S1].
- **Memory Usage:** Not given in the sources. [G] Multi-level page tables reduce the page table overhead for extremely large volumes [S2, unofficial].
- **Strengths:** Simple address translation logic; decoupled hierarchy allows independent management of page table structure and data resolution [S2, unofficial].
- **Limitations:** High cache usage on sparse data; cannot skip non-empty but invisible bricks [S1].

## Background
MRMCPT was introduced as a baseline out-of-core rendering method using a multi-resolution hierarchy managed via page tables [S1]. The multi-level page table variant was introduced by Hadwiger et al. in 2012 [S2, unofficial]. It is used in large-scale scientific visualization where volumes exceed GPU memory capacity [S2, unofficial].

## Sources
[S1] Residency Octree: A Hybrid Approach for Scalable Web ..., https://pmc.ncbi.nlm.nih.gov/articles/PMC10840607/, fetched 2026-10-29
[S2] GPU-Based Large-Scale Scientific Visualization, https://johanna-b.github.io/LargeSciVis2018/slides/LargeScaleVisualization_Architectures.pdf, fetched 2026-10-29 (unofficial)
