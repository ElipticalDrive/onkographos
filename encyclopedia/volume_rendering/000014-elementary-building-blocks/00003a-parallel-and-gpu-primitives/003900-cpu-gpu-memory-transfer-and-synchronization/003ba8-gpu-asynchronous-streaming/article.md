# GPU Asynchronous Streaming

Also called: GPU Async - Implementation, GPU Asynchronous

## Overview
GPU Asynchronous Streaming is a technique for performing truly asynchronous data transfers between the CPU and GPU by utilizing separate OpenGL contexts for download and drawing threads, leveraging the GPU's Copy Engine (CE) [S1, unofficial]. It solves the problem of GPU stalls caused by blocking data transfers in the same thread as rendering [S1, unofficial]. It sets itself apart from naive out-of-core loops or single-context PBO usage by ensuring the GPU never waits for data, maintaining high utilization even when PCIe traffic is minimal [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: Host-side volume data (e.g., `m_pVolume[m_curTimeStep]`), dimensions (`m_w`, `m_h`, `m_d`), and a Device Context (`hDC`) [S1, unofficial].
- Output: Rendered frames using textures updated asynchronously [S1, unofficial].

**Data Structures:**
- `HGLRC`: OpenGL Rendering Context handle [S1, unofficial].
- `HANDLE`: Thread handle [S1, unofficial].
- `texID`: Texture identifier generated via `glGenTextures` [S1, unofficial].
- PBOs (Pixel Buffer Objects): Used for data staging, managed by the driver [S1, unofficial].

**Steps:**
1. Create two OpenGL contexts from the same Device Context (`hDC`) using `wglCreateContext` [S1, unofficial].
2. Share textures between the two contexts using `wglShareLists` before any loading occurs [S1, unofficial].
3. Launch a separate download thread [S1, unofficial].
4. In the main draw thread, make the draw context current and perform texture access and drawing [S1, unofficial].
5. In the download thread, make the download context current and perform PBO-based texture downloads [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Initialization
hDC = get_device_context()
downloadRC = wglCreateContext(hDC)
drawRC = wglCreateContext(hDC)
wglShareLists(downloadRC, drawRC)

downloadThread = create_thread(download_func)

def main_loop():
    while not done:
        wglMakeCurrent(hDC, drawRC)
        # Texture access and drawing
        render_frame()

def download_func(lpParam):
    wglMakeCurrent(hDC, downloadRC)
    texID = glGenTextures(1)
    while True:
        # PBOs + glTexSubImage download as before
        pass
```

**Parameters:**
- `wglCreateContext`: API for context creation [S1, unofficial].
- `wglShareLists`: API for sharing lists [S1, unofficial].

**Edge Cases and Pitfalls:**
- The order of `wglShareLists` arguments matters [S1, unofficial].
- PBOs can block the GPU if used in the same thread as drawing [S1, unofficial].

## Subtopics

### [Asynchronous Pipeline (Overlap)](10166-asynchronous-pipeline-overlap.md)
A pipelining strategy that overlaps SSD read/prefetch, Host-to-Device (H2D) staging, and GPU rendering/backpropagation to hide I/O latency in out-of-core volume rendering. It sets itself apart from naive out-of-core loops by ensuring the GPU never stalls waiting for data, maintaining high utilization even when PCIe traffic is minimal.

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Measured Performance:** Not given in the sources.
- **Scaling:** Not given in the sources.
- **Strengths:** Allows the GPU's Copy Engine to overlap compute, download, and readback [S1, unofficial].
- **Limitations:** Requires separate threads and contexts [S1, unofficial].
- **When to use:** When using Fermi architecture or later GPUs with Copy Engines [S1, unofficial].
- **When not:** When using older architectures without dedicated Copy Engines [S1, unofficial].

## Background
Origin: NVIDIA GTC 2009 presentation [S1, unofficial].
History: Introduced as a method to leverage the Fermi architecture's Copy Engine for truly asynchronous downloads [S1, unofficial].
Usage: Part of NVIDIA Application Performance Optimization [S1, unofficial].

## Sources
[S1] PDF Slide 1, https://www.nvidia.com/content/GTC/documents/1102_GTC09.pdf, fetched 2026-10-29 (unofficial)
