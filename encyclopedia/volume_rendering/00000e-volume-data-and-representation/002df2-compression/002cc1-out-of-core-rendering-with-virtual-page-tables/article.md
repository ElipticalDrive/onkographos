# Out-of-Core Rendering with Virtual Page Tables

Also called: GPU-Based Out-of-Core Virtual Addressing, On-Demand Brick Transfer, Out-of-Core Algorithm, Page Table and LOD Management, Virtual Addressing Structure, Virtual Memory Page Table, Virtual Page Table Out-of-Core

## Overview
Out-of-Core Rendering with Virtual Page Tables is a rendering technique for very large volume caches that exceed memory limits by loading only visible bricks on demand [S1, unofficial]. It solves the problem of managing simultaneous multivariate and temporal data access without requiring High-Performance Computing (HPC) systems by employing out-of-core brick-based storage and on-demand GPU transfers [S4, unofficial]. Unlike in-core methods, it uses a GPU-managed caching system that handles data exceeding GPU/CPU capacity, allowing interactive navigation of large-scale data without storing everything in CPU memory [S3, unofficial]. It differs from its siblings, such as [Sparse structures](5-sparse-structures.md) and [Deferred Filtering Architecture for Compressed Volumes](857-deferred-filtering-architecture-for-compressed-volumes.md), by using a virtual addressing structure to map voxel requests to physical GPU memory locations via the BFAT, minimizing CPU-GPU communication bottlenecks [S3, unofficial]. The page table itself is implemented as a single flat array to reduce indirections and page misses [S2].

## How it works
**Inputs and Outputs:**
*   **Input:** A preprocessed, mipmapped, and bricked volume dataset stored on disk [S4, unofficial]. The data may include multiple Level of Detail (LOD) levels and temporal steps [S3, unofficial].
*   **Output:** Rendered image pixels, with volume data dynamically loaded into GPU memory as needed [S4, unofficial].

**Data Structures:**
1.  **Brick Layout:** The volume is divided into bricks [S3, unofficial]. Each brick is identified by a Space-Filling Curve (SFC) index [S3, unofficial].
2.  **Brick Format Address Table (BFAT):** A 2D logical structure used to locate bricks [S3, unofficial].
    *   **X-axis:** LOD level index [S3, unofficial].
    *   **Y-axis:** Time step index ($\gamma$) [S3, unofficial].
    *   **Content:** Stores the SFC index of the brick for a given (LOD, Time Step) pair [S3, unofficial].
3.  **Page Table:** A single flat array of entries (avoiding hierarchical page tables to reduce indirections and page misses [S2]).
    *   **Entry Layout:**
        *   `Key`: Composite key derived from (LOD, Time Step, SFC Index).
        *   `State`: Flag indicating if the brick is `Mapped` (in GPU memory) or `Not Mapped` [S3, unofficial].
        *   `GPU_Address`: The 3D GPU memory position where the brick is currently loaded [S3, unofficial].
        *   `Linked_Pages`: Optional list of pointers to other page table entries (e.g., for LOD blending) [S2].
4.  **Staging Buffer:** A host-visible memory buffer used for transferring data from CPU to GPU via Zero-Copy communication [S3, unofficial].

**Parameters:**
*   **Voxels per Brick:** $64^3$ [S3, unofficial].
*   **LOD Thresholds:** Target memory usage range of 50–80% [S2].
*   **Threshold Adjustment Step:** 1% [S2].

**Numbered Steps:**

1.  **Preprocessing:** Volumes are preprocessed, mipmapped, and bricked [S4, unofficial]. The BFAT is constructed to map (LOD, Time Step) coordinates to SFC indices [S3, unofficial].
2.  **Voxel Request:** The application interface sends a voxel request (3D position) to the GPU addressing data structure [S3, unofficial].
3.  **LOD Selection:** For the requested voxel at time step $\gamma$, identify the finest available LOD level [S3, unofficial]. If the finest LOD (e.g., LOD3) is empty or not available, select the next finest (e.g., LOD2) [S3, unofficial].
4.  **Address Calculation:** Calculate the offset in the BFAT using the time step ($\gamma$) on the Y-axis and the selected LOD on the X-axis [S3, unofficial]. Retrieve the SFC index of the target brick [S3, unofficial].
5.  **Cache Lookup:** Check the Page Table entry corresponding to the (LOD, SFC Index) key [S3, unofficial].
    *   If `State` is `Mapped`: Return the `GPU_Address` to the application [S3, unofficial].
    *   If `State` is `Not Mapped`: Proceed to Step 6 [S3, unofficial].
6.  **Asynchronous Load Request:** Send an asynchronous request to the CPU to load the data from disk (if not already in CPU memory) [S3, unofficial].
7.  **Data Transfer:** The CPU sends the data back to the GPU via Zero-Copy communication into the staging buffer [S3, unofficial].
8.  **Cache Update:** The addressing structure updates the page table entry: sets `State` to `Mapped`, records the `GPU_Address`, and returns the result to the application [S3, unofficial].
9.  **Rendering:** The rendering engine samples the data from the GPU memory [S4, unofficial]. Dynamic requests are handled throughout the visualization process [S4, unofficial].

**LOD Threshold Management:**
To maintain quality under high load, LOD transitions are adjusted at run-time based on the used-to-free memory ratio, targeting 50–80% usage [S2].
*   If usage is outside the 50–80% range, thresholds are moved [S2].
*   The step size for adjustment increases by 1% after successive similar adjustments or decreases by 1% if the direction of adjustment changes quickly [S2].
*   To maintain quality, important copies are prioritized, and LOD transitions are delayed to later frames if necessary [S2].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class PageTableEntry:
    state: int  # 0: Not Mapped, 1: Mapped
    gpu_address: int
    linked_pages: list

class BFAT:
    # 2D array: bfat[time_step][lod_level] -> sfc_index
    def get_sfc_index(self, time_step: int, lod_level: int) -> int:
        return self.data[time_step][lod_level] # [G]

class VolumeRenderer:
    def __init__(self, bfat: BFAT, page_table: dict, max_lod: int, staging_buffer_size: int):
        self.bfat = bfat
        self.page_table = page_table
        self.max_lod = max_lod
        self.staging_buffer_size = staging_buffer_size
        self.current_usage = 0.0
        self.adjustment_step = 0.01
        self.last_adjustment_direction = 0

    def get_voxel_value(self, x: int, y: int, z: int, time_step: int) -> float:
        lod_level = self._find_finest_available_lod(time_step)
        sfc_index = self.bfat.get_sfc_index(time_step, lod_level)
        key = (lod_level, sfc_index)
        entry = self.page_table.get(key)
        if entry is None:
            entry = PageTableEntry(state=0, gpu_address=0, linked_pages=[])
            self.page_table[key] = entry
        if entry.state == 1:
            return self._sample_from_gpu(entry.gpu_address, x, y, z)
        else:
            self._load_brick_to_gpu(time_step, lod_level, sfc_index)
            return self._sample_from_gpu(entry.gpu_address, x, y, z)

    def _find_finest_available_lod(self, time_step: int) -> int:
        for lod in range(self.max_lod, -1, -1):
            if self.bfat.has_data(time_step, lod):
                return lod
        return 0

    def _load_brick_to_gpu(self, time_step: int, lod_level: int, sfc_index: int):
        self._manage_memory_thresholds()
        # Asynchronous request to CPU to load data from disk if not in CPU memory [S3, unofficial]
        cpu_data = self._transfer_cpu_disk_to_host(time_step, lod_level, sfc_index)
        # Transfer to GPU via Zero-Copy communication into staging buffer [S3, unofficial]
        gpu_address = self._copy_to_gpu_staging_buffer(cpu_data)
        key = (lod_level, sfc_index)
        entry = self.page_table[key]
        entry.state = 1
        entry.gpu_address = gpu_address

    def _manage_memory_thresholds(self):
        if self.current_usage < 0.5 or self.current_usage > 0.8:
            if self.last_adjustment_direction > 0:
                self.adjustment_step = min(self.adjustment_step + 0.01, 0.1)
            elif self.last_adjustment_direction < 0:
                self.adjustment_step = max(self.adjustment_step - 0.01, 0.001)
            self.last_adjustment_direction = 1 if self.current_usage < 0.5 else -1
        # [G] Standard proportional adjustment logic based on source description

    def _transfer_cpu_disk_to_host(self, time_step: int, lod_level: int, sfc_index: int) -> bytes:
        # [G] Standard disk I/O and CPU memory staging
        return b""

    def _copy_to_gpu_staging_buffer(self, data: bytes) -> int:
        # [G] Standard Zero-Copy memory mapping and GPU transfer
        return 0

    def _sample_from_gpu(self, gpu_address: int, x: int, y: int, z: int) -> float:
        # [G] Standard GPU memory read and interpolation
        return 0.0
```

**Edge Cases and Pitfalls:**
*   **Blending:** LOD transitions require both levels to be in memory simultaneously during the transition, necessitating modification of page table entries to support linked pages [S2].
*   **Copy Budget:** LOD transitions and adding linked pages are less urgent than rendering visible Gaussian pages. The copy budget is limited by the staging buffer size and the time required [S2].
*   **Synchronization:** Strict synchronization during frame rendering causes pipeline stalls. Asynchronous task separation is recommended to speed up rendering, especially data copying [S2].
*   **Structure Requirement:** The cache must be structured as loadable bricks; otherwise, the entire file must be loaded [S1, unofficial].
*   **Multivariate Limitations:** Previous implementations were designed for simultaneous single-variable dataset visualization (3D+t+w with fixed monovariate storage per voxel), limiting flexibility for multivariate support [S4, unofficial].

## Subtopics

### [Out-of-Core Volume Rendering Pipeline](5386-out-of-core-volume-rendering-pipeline.md)
The Out-of-Core Volume Rendering Pipeline is an adaptive, modular architecture for the interactive visualization of large-scale 3D temporal multivariate (3D+t+w) gas and fluid datasets. It solves the problem of managing simultaneous multivariate and temporal data access without requiring High-Performance Computing (HPC) systems by employing out-of-core brick-based storage and on-demand GPU transfers. Unlike AMR-based bricking approaches, this method leverages prestructured datasets to enable parallel brick loading with a shared ID across different variables. It builds on the virtual page table strategy described in [Out-of-Core Rendering with Virtual Page Tables](5359-out-of-core-rendering-with-virtual-page-tables.md), extending it to handle temporal and multivariate dimensions.

### [Sparse Texture Binding](11857-sparse-texture-binding.md)
Sparse Texture Binding is a lossless, hardware-level mechanism in modern graphics APIs (Vulkan, Direct3D 12, OpenGL) that decouples the virtual volume size from the physical memory footprint by using hardware-managed page tables. It allocates physical memory pages only for specific subregions actively being sampled. Unlike its parent method, [Out-of-Core Rendering with Virtual Page Tables](5359-out-of-core-rendering-with-virtual-page-tables.md), it utilizes uniform page tables managed in hardware, where the fragment shader queries page residency and decides how to handle missing pages.

### Dynamic VRAM Pruning
Dynamic VRAM Pruning is a memory optimization technique that integrates an intelligent algorithm to prune points that do not lie in the camera's field of view (Frustum Culling) or points hidden behind opaque surfaces (Occlusion Culling) before sending data to CUDA cores, saving VRAM space [S5, unofficial].

## Operating characteristics
*   **Time Complexity:** Loading time depends on the number of bricks requested and the I/O bandwidth [G]. The method ensures that only visible bricks are loaded, minimizing unnecessary I/O [S3, unofficial].
*   **Memory Complexity:** Memory usage is bounded by the GPU VRAM capacity and the staging buffer size [S2].
*   **Measured Performance:** Evaluated on datasets with $64^3$ voxel bricks (4–1 MB per brick) [S3, unofficial]. The method allows interactive navigation of large-scale data that exceeds GPU/CPU capacity [S3, unofficial].
*   **Scaling:** The single-array page table avoids hierarchical indirections, reducing page misses and improving scalability [S2].
*   **Strengths:** Minimizes CPU-GPU communication bottlenecks via Zero-Copy and asynchronous requests [S3, unofficial]. Allows rendering of volumes larger than physical memory [S1, unofficial].
*   **Limitations:** Requires preprocessed, bricked data [S1, unofficial]. LOD transitions can cause pipeline stalls if not managed asynchronously [S2]. Limited flexibility for multivariate data in original implementations [S4, unofficial].
*   **When to use:** Interactive visualization of large-scale 3D datasets (e.g., AMR, fluid simulations) where the entire volume cannot fit in memory [S3, unofficial].

## Background
Initially demonstrated for 3D datasets on GPUs [6], this method has since been adapted for Adaptive Mesh Refinement (AMR) data in 3D + t contexts [5] [S4, unofficial]. It was developed to address the limitations of in-core methods, which require storing all data in CPU/GPU memory, thus limiting the scale of visualizable datasets [S3, unofficial]. The approach leverages virtual addressing structures to map voxel requests to physical GPU memory locations via the BFAT, minimizing CPU-GPU communication bottlenecks [S3, unofficial].

## Sources
[S1] Volume Rendering Based On 3D Voxel Grids, https://scratchapixel.com/lessons/3d-basic-rendering/volume-rendering-for-developers/volume-rendering-voxel-grids.html, fetched 2026-10-24 (unofficial)
[S2] Virtual memory for 3D Gaussian Splatting, https://zenodo.org/records/19690968, fetched 2026-10-26
[S3] GPU-based out-of-core architecture for interactive volume visualization of AMR time series., https://www.youtube.com/watch?v=pbikH-Rd9hI, fetched 2026-10-27 (unofficial)
[S4] Towards Scalable Out-of-Core Volume Rendering for High- ..., https://hal.science/hal-05141659v1/file/1030-file-i7.pdf, fetched 2026-10-30 (unofficial)
[S5] Optimizing Real-Time Neural Radiance Fields (NeRF) for Dynamic 3D Scene Rendering on Consumer GPUs, https://doi.org/10.5281/zenodo.21889839, fetched 2026-10-26 (unofficial)
