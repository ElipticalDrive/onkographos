# GPU Asynchronous Streaming

Also called: GPU Async - Implementation, GPU Asynchronous

## Overview
GPU Asynchronous Streaming is a technique for performing truly asynchronous data transfers between the CPU and GPU by utilizing separate OpenGL contexts for download and drawing threads, leveraging the GPU's Copy Engine (CE) on architectures such as Fermi [S1, unofficial]. It solves the problem of CPU-GPU synchronization bottlenecks where data download blocks the main draw thread, allowing the GPU's Copy Engine to overlap compute, download, and readback operations [S1, unofficial]. Unlike standard PBO usage where the GPU blocks if data download occurs in the same thread as drawing, this method uses a separate download thread and OpenGL contexts to enable true concurrency [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A Windows Device Context (hDC) representing the GPU device.
- Output: Two OpenGL rendering contexts (`downloadRC`, `drawRC`) and shared texture resources that allow asynchronous data transfer without stalling the main rendering loop.

**Data Structures:**
- `HGLRC`: OpenGL rendering context handle.
- `HDC`: Windows Device Context handle.
- `texID`: Texture object ID shared between contexts.
- `PBO`: Pixel Buffer Object used for data staging (details of PBO layout are inherited from standard OpenGL practices; specific PBO field sizes are not given in the sources).

**Parameters:**
- `hDC`: The device context from which both OpenGL contexts are created [S1, unofficial].
- `downloadRC`: The OpenGL context assigned to the download thread [S1, unofficial].
- `drawRC`: The OpenGL context assigned to the main draw thread [S1, unofficial].

**Steps:**
1. Create two OpenGL contexts from the same Device Context (hDC) using `wglCreateContext` [S1, unofficial].
2. Share textures between the two contexts using `wglShareLists(downloadRC, drawRC)` before any loading occurs. The order of arguments is noted as significant in the source [S1, unofficial].
3. Create a separate download thread from the main render thread [S1, unofficial].
4. In the main draw thread:
   - Make `drawRC` current using `wglMakeCurrent(hDC, drawRC)`.
   - Perform texture access and drawing operations.
5. In the download thread:
   - Make `downloadRC` current using `wglMakeCurrent(hDC, downloadRC)`.
   - Generate a texture ID (`glGenTextures`) [S1, unofficial].
   - Perform PBO-based downloads (e.g., `glTexSubImage`) as in standard asynchronous patterns [S1, unofficial].
6. The driver manages synchronization and scheduling between the contexts, allowing the Copy Engine to operate independently [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Initialization
hDC = get_device_context() # Platform-specific call to obtain HDC
downloadRC = wglCreateContext(hDC)
drawRC = wglCreateContext(hDC)
wglShareLists(downloadRC, drawRC) # Note: Order matters per source [S1, unofficial]

# Main Draw Thread Loop
def main_draw_loop():
    while not done:
        wglMakeCurrent(hDC, drawRC)
        # Texture access and drawing happens here
        # Note: Drawing code is called here after texture updates are scheduled
        pass

# Download Thread Function
def download_thread_func(lpParam):
    wglMakeCurrent(hDC, downloadRC)
    texID = glGenTextures(1)
    while True:
        # PBOs + glTexSubImage download as before
        # Implementation details for PBO binding and data upload are standard OpenGL
        # Specific PBO management steps are not detailed beyond "as before"
        pass
    return True

# Thread Creation
downloadThread = CreateThread(NULL, NULL, download_thread_func, NULL, NULL, NULL)
```

**Edge Cases and Pitfalls:**
- The order of arguments in `wglShareLists` is noted as significant ("Note order") [S1, unofficial].
- On Quadro cards, there are 2 Copy Engines, which allows overlapping compute, download, and readback [S1, unofficial].
- If the GPU blocks data download in the same thread as drawing, performance degrades; this method avoids that by using separate contexts [S1, unofficial].

## Subtopics

### Asynchronous Pipeline (Overlap)
A pipelining strategy that overlaps SSD read/prefetch, Host-to-Device (H2D) staging, and GPU rendering/backpropagation to hide I/O latency [S2, unofficial]. It executes four operations concurrently: (i) SSD read/prefetch into CPU cache, (ii) H2D transfer to VRAM, (iii) GPU compute, and (iv) Device-to-Host (D2H) transfer of evicted blocks plus async SSD flush [S2, unofficial]. Implementation uses dedicated I/O threads for SSD operations, CPU-side caching/dirty tracking, and separate CUDA streams for GPU compute and copies, with double-buffered GPU block buffers [S2, unofficial]. Serializing these operations increases iteration time to 210.5 ms compared to 90.7 ms with overlap, while keeping PCIe traffic at 0.10 GB/iter [S2, unofficial].

## Operating characteristics
- **Time Complexity:** Overlapping data movement and compute reduces iteration time from 210.5 ms (serialized) to 90.7 ms [S2, unofficial].
- **Memory Complexity:** Uses double-buffered GPU block buffers and CPU-side caching [S2, unofficial].
- **Performance:** Maintains low PCIe traffic (0.10 GB/iter) while maximizing GPU utilization by hiding I/O latency [S2, unofficial].
- **Strengths:** Truly asynchronous transfers on supported hardware (e.g., Fermi+ with Copy Engine); overlaps compute with data transfer; reduces iteration time significantly [S1, unofficial][S2, unofficial].
- **Limitations:** Requires specific API support (wglShareLists for OpenGL, CUDA streams for TideGS); dependent on hardware capabilities (e.g., Copy Engine availability) [S1, unofficial].
- **When to use:** When data transfer latency is a bottleneck and the GPU architecture supports asynchronous copy engines or stream-based execution [S1, unofficial][S2, unofficial].
- **When not to use:** On hardware without dedicated Copy Engines or when strict single-threaded context constraints exist [S1, unofficial].

## Background
The concept of GPU Asynchronous Streaming using separate OpenGL contexts was presented by NVIDIA in a GTC 2009 presentation, highlighting the Fermi architecture's Copy Engine capabilities [S1, unofficial]. The Asynchronous Pipeline variant is part of TideGS, a system for scalable training of over one billion 3D Gaussian Splatting primitives via out-of-core optimization, which addresses I/O bottlenecks in large-scale volume rendering training [S2, unofficial].

## Sources
[S1] PDF Slide 1, https://www.nvidia.com/content/GTC/documents/1102_GTC09.pdf, fetched 2026-10-29 (unofficial)
[S2] TideGS: Scalable Training of Over One Billion 3D Gaussian Splatting Primitives via Out-of-Core Optimization, https://doi.org/10.48550/arxiv.2605.20150, fetched 2026-10-26 (unofficial)
