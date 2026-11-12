# Hardware-based Direct Volume Ray Casting

Also called: VIRIM, VIRIM DSP shading, VIRIM Ray-Casting Unit

## Overview
Hardware-based Direct Volume Ray Casting is a specialized architecture that offloads the ray-casting, shading, and compositing computations to programmable Digital Signal Processors (DSPs) [S2]. It solves the performance bottleneck of CPU-based volume rendering by decoupling the high-speed geometry resampling stage from the slower, compute-intensive shading stage using dedicated hardware units [S2]. Unlike general-purpose GPU implementations, this method focuses on image quality and flexible illumination models (including shadows) via object-order ray casting, utilizing a custom memory architecture to manage bandwidth constraints [S1].

## How it works
The system operates in two main stages: geometry resampling and ray casting. The geometry unit generates sample density and gradient values, which are then processed by the ray-casting units.

**Inputs and Outputs**
*   **Input:** Rotated volume dataset (density and gradient values, 16-bit precision) [S2].
*   **Output:** Final rendered image scanlines [S2].

**Data Structures**
*   **FIFO Buffer:** Asynchronous buffer receiving sample and gradient values from the geometry unit [S2].
*   **Local Memory:** DSP-internal memory storing calculated scanlines [S2].
*   **Voxel Combination Units (VCUs):** Hardware nodes in the Ray Projection Tree variant, structured as a hierarchical binary tree [S3].

**Steps**
1.  **Data Transmission:** The geometry unit transmits sample density and gradient values to the ray-casting units over a bus. The peak transfer rate is 240 MBytes/sec at 40 MHz [S2].
2.  **Buffering:** DSPs receive values from a FIFO buffer. This mechanism decouples the fast geometry unit from the slower DSPs; for the Heidelberg Raytracing algorithm, the geometry unit is approximately 16 times faster than a DSP [S2].
3.  **Computation:** Within 15 clock cycles, the DSPs compute the interaction of light and material for each volume element [S2].
4.  **Scanline Storage:** One or more scanlines of the resulting projection are calculated per DSP and stored in local memory [S2].
5.  **Result Collection:** A local master CPU collects all scanlines from the DSP memories and transfers the results to the host system [S2]. No inter-DSP communication is necessary for the current visualization model [S2].

**Memory Architecture Optimization**
A specific memory architecture uses buffers and a pre-fetch mechanism to enhance bandwidth utilization [S1]. This increases the sample processing efficiency ($P_s$) to 0.2, at the cost of a higher voxel re-fetch factor ($V$) [S1]. In large multi-module configurations, sample processing efficiency per module may decrease due to network overhead [S1].

**Pseudocode (DSP Shading Loop)**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode representing the DSP operation within the VIRIM architecture
# Assumes data is available in FIFO buffer and local memory is accessible

def process_scanline(dsp_id, num_voxels):
    # 1. Receive data from FIFO buffer (decoupled from geometry unit)
    # Data format: 16-bit density, 16-bit gradient
    voxels = fifo_buffer.read(num_voxels)
    
    # 2. Compute light and material interaction
    # Completes within 15 clock cycles per volume element
    scanline = []
    for voxel in voxels:
        density = voxel.density
        gradient = voxel.gradient
        
        # Shading model implementation (e.g., Heidelberg Raytracing)
        # Uses local LUTs and memories accessible by host for real-time reloading
        color = compute_shading(density, gradient)
        scanline.append(color)
        
    # 3. Store result in local memory
    local_memory[dsp_id].write(scanline)

def collect_results(master_cpu, num_dsp):
    # 4. Local master CPU collects scanlines
    final_image = []
    for i in range(num_dsp):
        scanline = local_memory[i].read()
        final_image.append(scanline)
        
    # 5. Transfer to host system
    host_system.write(final_image)
```

**Parameters**
*   **Data Precision:** 16 bits for density and gradient values [S2].
*   **Bus Transfer Rate:** 240 MBytes/sec at 40 MHz [S2].
*   **DSP Computation Time:** 15 clock cycles per volume element [S2].
*   **Geometry vs DSP Speed Ratio:** Geometry unit is ~16x faster than DSP (for Heidelberg Raytracing) [S2].

**Edge Cases and Pitfalls**
*   **Bandwidth Bottleneck:** Sample processing efficiency is limited by the global bus between the re-sampling hardware and the rendering hardware [S1].
*   **Multi-Module Overhead:** In large configurations, network overhead can reduce sample processing efficiency per module [S1].
*   **Data Locality:** All data for shading calculations must remain in one sample plane to allow maximum flexibility [S2].

## Subtopics

### Ray Projection Tree
A hardware mechanism structured as a hierarchical binary tree of $n-1$ Voxel Combination Units (VCUs) that accepts $n$ voxels along a viewing ray at the leaves and produces the final pixel color at the root [S3]. It supports various projection schemes including first/last opaque, max/min voxel value, weighted summation, and compositing, unlike the original beam projection mechanism which was limited [S3].

**Operation:**
1.  Feed $n$ input voxels comprising a ray into $n/2$ VCUs [S3].
2.  Each VCU combines two input voxels into an output voxel value in 7 time units, producing $n/2$ results [S3].
3.  Feed the $n/2$ results into the next stage of the tree (containing $n/4$ VCUs) while simultaneously feeding the next ray of $n$ voxels into the first stage [S3].
4.  After an initialization time of $7 \log n$ time units, the tree processes $\log n$ rays simultaneously in a pipeline fashion, producing a new pixel color every 7 time units [S3].

**Parameters:**
*   **VCU Latency:** 7 time units per combination [S3].
*   **Initialization Time:** $7 \log n$ time units [S3].

## Operating characteristics
*   **Performance:** VIRIM performance is estimated using the Heidelberg Raytracing algorithm. A single VIRIM module consists of four boards (36x39 cm²), and four modules fit into one crate [S2]. For a 256x256x128 dataset, a single module achieves a frame rate of 2.5 Hz [S2].
*   **Scalability:** Multiple engines can be used to increase performance, but each engine must duplicate the entire dataset [S1].
*   **Strengths:** High image quality, flexible illumination models (including shadows), and decoupled architecture allowing asynchronous operation between geometry and shading units [S1, S2].
*   **Limitations:** Sample processing efficiency is limited by the global bus bandwidth [S1]. The Array Based Ray Casting variant (a sibling architecture) scales with $O(n^2)$ hardware requirements for volume dimension $n$, though VIRIM uses DSPs which may have different scaling characteristics [S1].
*   **When to use:** When image quality and flexible shading (shadows) are prioritized over raw throughput, and when custom hardware integration is feasible.

## Background
The VIRIM architecture was developed as part of research into hardware-accelerated volume visualization. It was presented in papers detailing three architectures for volume rendering and ray casting architectures for volume visualization [S1, S2]. The system was designed to address the limitations of CPU-based rendering and early hardware approaches by leveraging programmable DSPs for flexibility in shading algorithms [S2]. The Ray Projection Tree variant was introduced in the context of an extended volume visualization system for arbitrary parallel projection [S3].

## Sources
[S1] Ray Casting Architectures for Volume Visualization, https://vcg.seas.harvard.edu/publications/19990101-ray-casting-architetures-for-volume-visualization/paper, fetched 2026-10-27
[S2] Three Architectures for Volume Rendering, https://vcg.seas.harvard.edu/publications/19950801-three-architectures-for-volume-rendering/paper, fetched 2026-10-29
[S3] An Extended Volume Visualization System for Arbitrary Parallel Projection, https://doi.org/10.2312/eggh/eggh92/064-069, fetched 2026-10-29
