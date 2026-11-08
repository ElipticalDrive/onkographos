# CPU Multithreaded Rendering

Also called: Multi-core Volume Rendering, Parallel CPU Rendering

## Overview
CPU Multithreaded Rendering maps volumetric data to a 2D output by distributing the computational load of ray casting or projection across multiple CPU cores [S1]. It solves the performance bottleneck of single-threaded software rendering by splitting the rendering domain into parallel sections [S1]. Unlike [GPU, web and parallel renderers](50-gpu-web-and-parallel-renderers.md), which rely on hardware shaders and coalesced memory access, this method leverages general-purpose CPU instruction sets and explicit thread management, often requiring careful handling of OpenGL context binding if integrated with immediate-mode graphics APIs [S3].

## How it works
### Inputs and Outputs
*   **Inputs**: Volumetric dataset (e.g., $256^3$ medical data [S2, unofficial]), rendering parameters (transfer functions, clipping), and scene configuration.
*   **Outputs**: Rendered image frames (e.g., stereo pairs [S2, unofficial]) or memory buffers.

### Data Structures
The sources do not specify a universal data structure layout for the volume data itself, noting that performance depends on the "size and structure of the visualized features" rather than the monolithic volume size [S2, unofficial]. For scene management, the VRE engine uses a structure where each scene is associated with two persistent threads [S3].

### Steps
1.  **Domain Decomposition**: The CPU software version splits the world (or image plane) into sections for parallel rendering [S1].
2.  **Thread Assignment**:
    *   For general volume rendering, threads are spawned or managed to process these sections [S1].
    *   For scene management with OpenGL, each scene is assigned two persistent threads to handle asynchronous operations [S3].
3.  **Execution**:
    *   **Synchronous**: Execution occurs on the calling thread [S3].
    *   **Asynchronous**: Execution occurs on a separate persistent thread. The thread falls asleep until the next operation request arrives, preserving the OpenGL context binding [S3].
4.  **Context Handling**:
    *   One thread operates on a user-provided OpenGL context (e.g., a user-created window) [S3].
    *   A second thread operates on an internal invisible OpenGL context, used when no context is provided (e.g., command-line applications) [S3].
5.  **Rendering**:
    *   Standard volume raycasting or projection is performed on the assigned sections [S1].
    *   In voxel-set rendering, only voxels representing structures of interest are processed, allowing for object-aware rendering modes (direct volume rendering, MIP, surface rendering) combined via compositing [S2, unofficial].
    *   Stereo pairs can be rendered in parallel using multi-threading [S2, unofficial].

### Parameters
*   **Core Count**: Performance scales with the number of cores. Tested on systems ranging from 1 core to 32 cores (Opteron) [S1].
*   **Data Size**: Typical visualizations use $256^3$ data sets [S2, unofficial].

### Pseudocode
The following pseudocode illustrates the persistent thread model for asynchronous OpenGL rendering described in [S3].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import threading
import time

class SceneRenderer:
    def __init__(self, user_context=None):
        # Create an internal invisible context if none provided
        self.internal_context = self._create_opengl_context() if user_context is None else None
        self.user_context = user_context
        
        # Two persistent threads per scene
        self.thread_user = threading.Thread(target=self._render_loop, args=(self.user_context,))
        self.thread_internal = threading.Thread(target=self._render_loop, args=(self.internal_context,))
        
        # Flags for asynchronous requests
        self.request_lock = threading.Lock()
        self.pending_render = False
        self.render_data = None
        
        # Start threads
        self.thread_user.start()
        self.thread_internal.start()

    def _create_opengl_context(self):
        # [G] Placeholder for OpenGL context creation logic
        # In a real implementation, this would create a hidden window/context
        return "OpenGL_Context_Instance"

    def _render_loop(self, context):
        while True:
            with self.request_lock:
                if self.pending_render:
                    data = self.render_data
                    self.pending_render = False
                    self.render_data = None
            
            if data is not None:
                # Bind context
                self._bind_context(context)
                
                # Perform rendering
                self._execute_render(data)
                
                # Unbind context
                self._unbind_context()
            
            # Fall asleep until next request
            time.sleep(0.01)

    def render_async(self, data):
        with self.request_lock:
            self.render_data = data
            self.pending_render = True
        # Return immediately to caller
        return

    def _bind_context(self, context):
        # [G] Placeholder for glMakeCurrent or equivalent
        pass

    def _unbind_context(self):
        # [G] Placeholder for context unbinding
        pass

    def _execute_render(self, data):
        # [G] Placeholder for actual volume rendering logic
        pass
```

### Edge Cases and Pitfalls
*   **OpenGL Context Binding**: Creating a new thread for each asynchronous operation causes costly OpenGL context binding operations. This is mitigated by using persistent threads that fall asleep between requests [S3].
*   **Context Availability**: If a user does not provide an OpenGL context (e.g., in command-line applications), an internal invisible context must be used [S3].

## Subtopics

### Voxel-set rendering
A software-based volume rendering approach that achieves high performance by working with sets of voxels representing structures of interest, rather than rendering the entire monolithic volume [S2, unofficial]. It allows for object-aware rendering modes (direct volume rendering, compositing, MIP, surface rendering, NPR) combined via compositing [S2, unofficial]. It supports multi-threading for stereo pairs [S2, unofficial]. Performance metric: Up to 35 frames per second on a single CPU for typical visualizations of $256^3$ data sets from medicine [S2, unofficial]. Choose this when visualization features are sparse or structured, and performance depends on feature size rather than volume size [S2, unofficial].

### Multi-threaded Scene Management with Persistent OpenGL Contexts
A threading strategy for managing multiple scenes and asynchronous rendering operations that minimizes the cost of OpenGL context binding by using persistent threads per scene [S3]. Each scene uses two persistent threads (one for user context, one for internal context) that fall asleep between requests [S3]. Choose this when integrating CPU rendering with OpenGL-based UI frameworks where asynchronous rendering is required without blocking the main thread [S3].

## Operating characteristics
*   **Time Complexity**: Performance increases with the number of cores. Tested on 1 to 32 cores (Opteron) [S1].
*   **Memory Complexity**: Not given in the sources.
*   **Measured Performance**: Up to 35 frames per second on a single CPU for $256^3$ medical data sets using voxel-set rendering [S2, unofficial].
*   **Strengths**: High flexibility with visualization parameters (e.g., two-level volume rendering) [S2, unofficial]. Supports stereo pairs rendered in parallel [S2, unofficial].
*   **Limitations**: Performance depends on the structure of visualized features [S2, unofficial]. OpenGL context binding is costly if not managed with persistent threads [S3].
*   **When to Use**: When hardware acceleration is unavailable or when specific software-based features (like object-aware rendering) are required [S2, unofficial].
*   **When Not to Use**: When real-time performance on large monolithic volumes is required without feature sparsity [S2, unofficial].

## Background
The concept of CPU multithreaded volume rendering has been implemented in various software frameworks. The Voxel Game Engine describes a CPU software version that splits the world for parallel rendering, tested on Opteron processors [S1]. The Real Time Volume Rendering Library (RTVR) introduced voxel-set rendering for interactive volume rendering, achieving high performance by focusing on structures of interest [S2, unofficial]. The VRE volume rendering engine implemented a scene management system with persistent OpenGL contexts to handle asynchronous rendering efficiently [S3].

## Sources
[S1] Voxel Game Engine, http://voxelgames.com/, fetched 2026-10-27
[S2] Distributed Software-Based Volume Visualization in a Virtual Environment, https://doi.org/10.2312/egve/ipt_egve2005/129-140, fetched 2026-10-26 (unofficial)
[S3] The VRE volume rendering engine, https://doi.org/10.1145/1925059.1925072, fetched 2026-11-05
