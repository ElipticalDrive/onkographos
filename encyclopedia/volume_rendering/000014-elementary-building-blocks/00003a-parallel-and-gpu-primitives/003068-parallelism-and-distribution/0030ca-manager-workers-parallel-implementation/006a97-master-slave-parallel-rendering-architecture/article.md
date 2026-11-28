# Master/Slave Parallel Rendering Architecture
Also called: MovieMaker Parallel Rendering Architecture

## Overview
The Master/Slave Parallel Rendering Architecture is a shared-memory parallelization strategy designed to generate visualization movies from large-scale (TB-scale), three-dimensional, time-evolving simulation data within strict time limits (e.g., 12 hours) [S1]. It distinguishes itself from other parallel rendering systems by utilizing standard non-parallel disk I/O combined with task overlapping to hide storage latency, whereas alternatives like Yu et al. require complex parallel disk I/O infrastructure [S1]. It builds on [Manager/Workers Parallel Implementation](6359-manager-workers-parallel-implementation.md) by specifically addressing the I/O bottleneck inherent in shared-memory clusters through asynchronous data loading.

## How it works
**Inputs and Outputs:**
*   **Input:** Simulation data defined in Cartesian geometry with a rectilinear grid system (uniform and/or non-uniform in x, y, and z directions) [S1]. Data is typically stored on disk in time steps (e.g., 1 GB per step) [S1].
*   **Output:** A sequence of visualization images (movie) generated via volume rendering, isocontouring, or streamlines [S1].

**Data Structures:**
*   **Shared Memory Area:** A contiguous memory region accessible by all processes (master and slaves) where simulation data is loaded [S1]. The maximum grid size is constrained only by the available shared memory size [S1].
*   **Configuration File:** Contains parameters for visualization methods and fine-tuning settings [S1].
*   **Partial Images:** Rendered sub-images returned by slave processes to the master via shared memory [S1].

**Numbered Steps:**
1.  **Initialization:** The master process reads the configuration file [S1].
2.  **Data Loading:** The master process reads the simulation data for the current time step from disk into the shared memory area [S1].
3.  **Task Distribution:** The master process controls slave processes, assigning rendering tasks while maintaining dynamic load balance through active monitoring [S1].
4.  **Rendering Execution:** Slave processes execute rendering commands (volume rendering, isocontouring, or streamlines) using the data in shared memory. They render partial images (sub-images) and return them to the master via shared memory [S1].
5.  **Task Overlapping (I/O Hiding):** While slave processes render the current time step, the master process simultaneously reads the data for the *next* time step from disk into shared memory [S1]. This overlaps disk I/O latency with computation time [S1].
6.  **Image Composition:** The master process assembles the partial images returned by the slaves into the final frame [S1].
7.  **Iteration:** Steps 2–6 repeat for each time step in the simulation dataset [S1].

**Formulas and Parameters:**
*   **Target Performance:** Processing one GB of data within 40 seconds [S1].
*   **I/O Latency Hiding:** The system assumes reading 1 GB of data takes approximately 1 second [S1]. The total read time for a movie (e.g., 1,024 steps) could reach ~1 hour, which is hidden by the rendering time of the slaves [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Master Process
def master_process():
    config = read_configuration_file()
    shared_memory = allocate_shared_memory()
    
    # Pre-load first time step
    current_step_data = read_disk_to_shared_memory(shared_memory, step=0)
    
    for step in range(total_steps):
        # Task Overlapping: Read next step while current step renders
        if step + 1 < total_steps:
            next_step_data = read_disk_to_shared_memory(shared_memory, step=step+1)
        
        # Dispatch rendering tasks to slaves
        # Slaves render current_step_data and return partial images
        partial_images = dispatch_rendering_tasks(shared_memory, step, config)
        
        # Compose final image from partial images
        final_image = compose_image(partial_images)
        save_image(final_image, step)

# Slave Process
def slave_process(slave_id):
    while True:
        command = receive_command_from_master()
        if command.type == "RENDER":
            # Render using OpenGL off-screen
            sub_image = render_offscreen(command.data_pointer, command.params)
            # Return sub-image to shared memory
            write_to_shared_memory(command.return_address, sub_image)
        elif command.type == "EXIT":
            break
```

**Edge Cases and Pitfalls:**
*   **Memory Constraints:** The maximum grid number is constrained only by the shared memory size [S1]. Large grids may exceed available RAM.
*   **I/O Bottleneck:** If disk read speed is significantly slower than assumed (1s/GB), the overlapping may not fully hide latency, leading to idle time [S1].
*   **Load Imbalance:** Dynamic load balancing is required; static assignment may lead to inefficient resource usage if rendering tasks vary in complexity [S1].

## Subtopics

### Dynamic Load Balancing
Active monitoring and control of slave processes to ensure efficient resource utilization [S1].

### Task Overlapping
Asynchronous disk I/O to hide storage latency during rendering [S1].

### Off-Screen Rendering
Use of OpenGL for rendering without display output [S1].

## Operating characteristics
*   **Time Complexity:** The system aims to process 1 GB of data within 40 seconds [S1].
*   **Measured Performance:** On an SGI Onyx 3800 (12 CPU, 24 GB memory, 11 MPI processes), average processing times for a 640x640x640 grid (4-byte float) were: Volume rendering 40 seconds, Isocontouring 33 seconds, Streamlines 10 seconds [S1].
*   **Memory Complexity:** Limited by shared memory size; supports TB-scale data through time-step processing [S1].
*   **Strengths:** Simple disk I/O implementation (no parallel file system required); effective hiding of I/O latency; supports multiple visualization methods (volume, isocontour, streamlines) [S1].
*   **Limitations:** Requires shared-memory architecture; performance limited by single-disk I/O speed for large datasets; maximum grid size limited by RAM [S1].
*   **When to Use:** When generating movies from large-scale simulation data on shared-memory clusters where parallel disk I/O is unavailable or too complex to implement [S1].
*   **When Not to Use:** When data exceeds available shared memory; when disk I/O is significantly slower than 1s/GB; when parallel file system access is available and preferred [S1].

## Background
The Master/Slave Parallel Rendering Architecture was developed by Fumiaki Araki and Akira Kageyama at the Earth Simulator Center, Japan Agency for Marine-Earth Science and Technology [S1]. It was introduced in the paper "MovieMaker: A Parallel Movie-Making Software for Large Scale Simulations" (arXiv:physics/0601211) on January 30, 2006 [S1]. The system was designed to address the growing output data size of high-performance computer simulations, specifically targeting the generation of movies from 1 TB-scale data within 12 hours [S1]. It contrasts with earlier parallel rendering systems that required parallel disk I/O solutions [S1].

## Sources
[S1] MovieMaker: A Parallel Movie-Making Software for Large Scale Simulations, https://archive.org/details/arxiv-physics0601211, fetched 2026-10-26
