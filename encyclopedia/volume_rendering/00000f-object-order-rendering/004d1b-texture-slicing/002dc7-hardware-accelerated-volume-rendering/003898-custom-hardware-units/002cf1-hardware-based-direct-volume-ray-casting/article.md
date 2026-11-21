# Hardware-based Direct Volume Ray Casting

Also called: VIRIM, VIRIM DSP shading, VIRIM Ray-Casting Unit

## Overview
Hardware-based Direct Volume Ray Casting, specifically the VIRIM (Volume Image Rendering on Interactive Machines) architecture, is a specialized DSP-based system designed for high-quality, object-order volume rendering [S1, S2]. It solves the computational bottleneck of real-time ray casting by offloading the intensive sample processing, shading, and compositing tasks from the CPU to programmable Digital Signal Processors (DSPs) [S1, S2]. Unlike general-purpose GPU ray casting, VIRIM decouples the high-speed geometric resampling stage from the shading stage using a dedicated bus and FIFO buffers, allowing for flexible illumination models including shadows and complex material interactions [S1, S2]. It is distinguished from sibling architectures like the Array-Based Ray Casting by its use of programmable DSPs rather than fixed-function logic, prioritizing image quality and algorithmic flexibility over raw throughput scaling [S1, S2].

## How it works
The VIRIM architecture operates in two distinct hardware stages: a Geometry Unit (resampling) and a Ray-Casting Unit (shading/compositing) [S2].

**Inputs and Outputs:**
*   **Input:** 3D volume dataset (density and gradient values, 16-bit precision) [S2].
*   **Output:** Final 2D image (scanlines) [S2].

**Data Structures:**
*   **Volume Data:** Stored in memory accessible by the host system for real-time reloading [S2].
*   **Sample Plane:** A temporary storage area where density and gradient values for a specific sample plane reside during DSP calculation [S2].
*   **FIFO Buffer:** Asynchronous buffer between the Geometry Unit and DSPs [S2].
*   **Local DSP Memory:** Stores calculated scanlines before transfer to the host [S2].

**Steps:**
1.  **Resampling (Geometry Unit):** The Geometry Unit generates sample density and gradient values for the rotated dataset. It uses a memory architecture with buffers and a pre-fetch mechanism to enhance bandwidth utilization [S1]. This mechanism increases the voxel re-fetch factor $V$ but achieves a sample processing efficiency $P_s$ of 0.2 [S1].
2.  **Transmission:** Sample density and gradient values are transmitted from the Geometry Unit to the Ray-Casting Units over a global bus [S1, S2]. The bus has a peak transfer rate of 240 MBytes/sec at 40 MHz [S2].
3.  **Buffering:** Each DSP receives values from a First-In-First-Out (FIFO) buffer that can be accessed asynchronously [S2]. This decouples the fast Geometry Unit from the slower DSPs; for the Heidelberg Raytracing algorithm, the Geometry Unit is approximately 16 times faster than a DSP [S2].
4.  **Shading and Compositing (DSP):** Programmable DSPs compute the interaction of light and material for each volume element [S2].
    *   **Precision:** All calculations, including density and gradient values, use 16-bit precision [S2].
    *   **Algorithm:** The Heidelberg Raytracing algorithm is used, which supports flexible illumination including shadows [S1, S2].
    *   **Timing:** Each DSP computes the interaction for its assigned volume elements within 15 clock cycles [S2].
5.  **Scanline Storage:** One or more scanlines of the resulting projection are calculated per DSP and stored in local DSP memory [S2].
6.  **Aggregation:** A local master CPU collects all scanlines from the DSP memories and transfers the final image results to the host system [S2]. No inter-DSP communication is necessary for the current visualization model [S2].

**Parameters:**
*   **Bus Clock:** 40 MHz [S2].
*   **Bus Peak Bandwidth:** 240 MBytes/sec [S2].
*   **Data Precision:** 16 bits for density and gradient values [S2].
*   **DSP Compute Time:** 15 clock cycles per volume element interaction [S2].
*   **Geometry/DSP Speed Ratio:** ~16:1 (Geometry is faster) [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# VIRIM Hardware Pipeline Execution
# Note: This pseudocode represents the logical flow of data through the hardware units.

def virim_render(volume_data, resolution):
    # 1. Geometry Unit: Resampling
    # The Geometry Unit generates sample planes.
    # It uses a pre-fetch mechanism with buffers.
    # Efficiency Ps = 0.2. Voxel re-fetch factor V is increased.
    sample_planes = geometry_unit.resample(volume_data, resolution)
    
    # 2. Transmission over Global Bus
    # Peak rate: 240 MBytes/sec at 40 MHz.
    # Data: 16-bit density and gradient values.
    bus_bandwidth = 240 * 1024 * 1024  # Bytes/sec
    bus_clock = 40 * 1024 * 1024      # Hz
    
    # 3. Ray-Casting Unit: DSP Processing
    # DSPs receive from FIFO buffers.
    # Decoupling factor: Geometry is ~16x faster than DSP.
    dsp_count = 4  # Per module (from performance estimation context)
    scanlines = []
    
    for plane in sample_planes:
        # FIFO buffer handles asynchronous transfer
        fifo_buffer = FIFO()
        fifo_buffer.push(plane)
        
        # DSPs process the plane
        # Each DSP handles one or more scanlines
        # Precision: 16-bit
        # Algorithm: Heidelberg Raytracing (supports shadows)
        # Time: 15 clock cycles per element
        for dsp in range(dsp_count):
            # DSP computes light/material interaction
            # Data remains in one sample plane
            scanline = dsp.compute_shading(fifo_buffer.pop(), precision_bits=16)
            scanlines.append(scanline)
            
    # 4. Aggregation
    # Local master CPU collects scanlines
    final_image = local_master_cpu.collect(scanlines)
    
    # 5. Transfer to Host
    host_system.receive(final_image)
    
    return final_image
```

## Subtopics

### Ray Projection Tree
The Ray Projection Tree (RPT) is a hardware mechanism structured as a hierarchical binary tree of $n-1$ Voxel Combination Units (VCUs) [S3]. It accepts $n$ voxels along a viewing ray at the leaves and produces the final pixel color at the root [S3]. Each VCU combines two input voxels into an output voxel value in 7 time units [S3]. The tree processes $\log n$ rays simultaneously in a pipeline fashion after an initialization time of $7 \log n$ time units, producing a new pixel color every 7 time units [S3]. Unlike the original beam projection mechanism, the RPT supports various projection schemes including first/last opaque, max/min voxel value, weighted summation, and compositing [S3].

## Operating characteristics
*   **Performance:** A single VIRIM module consists of four boards (36x39 cm² each), with four modules fitting into one crate [S2]. For a 256x256x128 dataset, a single module achieves a frame rate of 2.5 Hz [S2].
*   **Throughput:** The sample processing efficiency $P_s$ is 0.2 with the new memory architecture [S1]. The bus peak transfer rate is 240 MBytes/sec [S2].
*   **Scalability:** Multiple engines can be used to increase performance, but each engine must duplicate the entire dataset [S1]. In large multi-module configurations, sample processing efficiency per module may decrease due to network overhead [S1].
*   **Strengths:** High image quality; flexible illumination models including shadows; decoupled architecture allows independent optimization of resampling and shading [S1, S2].
*   **Limitations:** Sample processing efficiency is limited by the global bus between resampling and rendering hardware [S1]. The requirement to duplicate the dataset for each engine increases memory requirements [S1]. The hardware required for array-based variants scales with $O(n^2)$, though VIRIM uses DSPs which may have different scaling characteristics [S1].
*   **When to use:** When high image quality and flexible shading (e.g., shadows) are prioritized over maximum raw throughput or scalability [S1, S2].

## Background
The VIRIM architecture was developed as part of research into hardware-based volume rendering architectures, specifically focusing on DSP-based solutions [S1, S2]. It was presented in papers detailing three architectures for volume rendering and ray casting architectures for volume visualization [S1, S2]. The architecture was designed to address the limitations of software-based rendering and earlier hardware approaches by leveraging programmable DSPs for flexibility in shading algorithms [S1, S2]. The Heidelberg Raytracing algorithm is the primary shading model implemented in VIRIM [S2].

## Sources
[S1] Ray Casting Architectures for Volume Visualization, https://vcg.seas.harvard.edu/publications/19990101-ray-casting-architetures-for-volume-visualization/paper, fetched 2026-10-27
[S2] Three Architectures for Volume Rendering, https://vcg.seas.harvard.edu/publications/19950801-three-architectures-for-volume-rendering/paper, fetched 2026-10-29
[S3] An Extended Volume Visualization System for Arbitrary Parallel Projection, https://doi.org/10.2312/eggh/eggh92/064-069, fetched 2026-10-29
