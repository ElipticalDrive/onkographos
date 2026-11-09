# GPU-based Fluid Simulation with Voxel Rendering

Also called: Integrating Real-Time Fluid Simulation with a Voxel Engine, Poxels Fluid Integration

## Overview
This method integrates real-time incompressible fluid simulation with sparse voxel-based rasterization on graphics hardware, specifically leveraging Direct3D geometry shaders for rendering and compute shaders for simulation [S1]. It solves the problem of efficiently visualizing fluid dynamics within a voxel engine by avoiding CPU-GPU data transfer bottlenecks through a fully GPU-resident pipeline [S1]. It differs from its parent method, [Simulation output](15-simulation-output.md), by defining specific numerical kernels (advection, divergence, Jacobi solve, projection) and data layouts required to perform simulation steps on the GPU [S1]. It differs from [Stable Fluids (GPU Implementation)](7906-stable-fluids-gpu-implementation.md) by explicitly coupling the simulation output to a voxel rendering engine using stream amplification and geometry shaders, rather than focusing solely on the solver mechanics [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D simulation domain defined as a "Chunk" with resolution $(N_x, N_y, N_z)$, typically a cube. External forces (static or dynamic) and initial velocity/pressure fields.
*   **Output:** A 3D speed grid texture (single precision floats) representing the fluid state, which is rendered as active voxels.

**Data Structures:**
The simulation uses GPU compute textures composed of single-precision float 4 types (`float4`).
*   **Velocity Textures:** Three textures are used to store velocity fields (current, previous, and temporary) [S1].
*   **Pressure Textures:** Two textures are used for pressure values (current and temporary) [S1].
*   **Speed Grid Texture:** The final result of the five GPU compute stages, filled with single-precision floats, serving as the source fluid field for the voxel engine [S1].
*   **Non-Zero Texture:** A texture containing only non-zero values, used by the geometry shader to determine voxel activation [S1].
*   **Staging Texture:** Used only in the CPU approach to transfer data from GPU to CPU memory [S1].

**Parameters:**
*   **Chunk Size:** The simulation domain size. Typical tested sizes include $32^3$, $64^3$, $96^3$, and $128^3$ blocks [S1].
*   **Alignment:** CPU memory buffers require appropriate memory address alignment when mapping staging textures [S1].
*   **Color Ramp:** A function mapping a float value between 0 and 1 to an RGB color [S1].

**Numbered Steps:**
1.  **Define Processing Stages:** Establish the Navier–Stokes processing stages: advection of the velocity field, acceleration caused by pressure, diffusion of momentum, and external forces [S1].
2.  **Update Velocity:** Perform three computations to update the velocity at each time step: advection, diffusion, and force application [S1].
3.  **Compute Pressure:** Calculate the pressure field and subtract the pressure gradient from the velocity field to enforce incompressibility [S1].
4.  **Output Speed Grid:** The final result of the five GPU compute stages is output in a 3D speed grid texture [S1].
5.  **Voxel Activation (CPU Approach):**
    *   Read the speed grid texture back to CPU memory via a staging texture [S1].
    *   Map the staging texture in CPU memory and copy the result from the 3D float array [S1].
    *   Activate voxels if the corresponding value in the dense 3D array is not zero [S1].
    *   Apply a color ramp function to the float value to return an RGB color passed to the geometry shader [S1].
6.  **Voxel Activation (GPU Optimization):**
    *   Move voxel activation logic from CPU to the geometry shader [S1].
    *   Add a texture in the fluid simulation compute shader containing only non-zero values [S1].
    *   Dispatch to geometry shader stream amplification with stream-out to a cached vertex buffer [S1].
    *   Apply the color ramp directly in the geometry shader [S1].
    *   Regular triangle geometry rasterization with the generated vertex buffer [S1].

**Pseudocode:**
The following pseudocode illustrates the GPU-optimized pipeline, as the CPU approach involves explicit memory copying and mapping which is platform-specific.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# GPU Compute Shader Stage
def compute_fluid_state(chunk_size, velocity_textures, pressure_textures, forces):
    """
    Performs Navier-Stokes simulation steps on GPU.
    """
    # 1. Advection
    advected_velocity = advect(velocity_textures.current, velocity_textures.previous, chunk_size)
    
    # 2. Diffusion
    diffused_velocity = diffuse(advected_velocity, chunk_size)
    
    # 3. Force Application
    forced_velocity = diffused_velocity + forces
    
    # 4. Pressure Projection
    divergence = calculate_divergence(forced_velocity, chunk_size)
    pressure = solve_jacobi(divergence, pressure_textures.current, chunk_size)
    pressure_gradient = calculate_gradient(pressure, chunk_size)
    
    # 5. Final Velocity
    final_velocity = forced_velocity - pressure_gradient
    
    # 6. Output Speed Grid
    speed_grid_texture = final_velocity
    
    # 7. Create Non-Zero Mask for Rendering
    non_zero_texture = create_non_zero_mask(final_velocity, chunk_size)
    
    return speed_grid_texture, non_zero_texture

def render_voxels(non_zero_texture, color_ramp_function):
    """
    Geometry Shader Stage for Voxel Rendering.
    """
    # Dispatch geometry shader with stream amplification
    for each voxel in non_zero_texture:
        if voxel.value != 0:
            # Determine voxel position and size
            voxel_pos = get_voxel_position(voxel)
            voxel_size = get_voxel_size(voxel)
            
            # Apply color ramp
            intensity = normalize(voxel.value)
            color = color_ramp_function(intensity)
            
            # Generate vertex data for the voxel (e.g., a cube)
            vertices = generate_cube_vertices(voxel_pos, voxel_size)
            
            # Stream-out to cached vertex buffer
            stream_out(vertices, color)
            
    # Rasterize the generated vertex buffer
    rasterize_vertex_buffer()
```

**Edge Cases and Pitfalls:**
*   **Obstacle Detection:** The described simulation does not detect or respond to obstacles [S1].
*   **Single Chunk:** The simulation is based on a single chunk and needs extension to multiple chunks for larger scenes [S1].
*   **Precision:** The use of single-precision floats is specified; precision issues may arise in large-scale simulations [S1].
*   **Memory Alignment:** In the CPU approach, inappropriate memory address alignment for the CPU buffer can cause errors [S1].

## Subtopics

### [Stable Fluids (GPU Implementation)](7906-stable-fluids-gpu-implementation.md)
A method for computing incompressible fluid velocity by discretizing the momentum equation on a fixed Eulerian grid and solving for velocity via advection, external forces, and pressure projection. It solves the problem of real-time fluid simulation on graphics hardware by leveraging 3D textures and slice-based rasterization to execute computational kernels. It differs from its parent method, [GPU-based Fluid Simulation with Voxel Rendering](7400-gpu-based-fluid-simulation-with-voxel-rendering.md), by specifically defining the numerical kernels (advection, divergence, Jacobi solve, projection) and data layouts required to perform the simulation steps on the GPU.

## Operating characteristics
**Time and Memory Complexity:**
*   The simulation scales linearly with the number of voxels in the chunk [S1].
*   Memory usage is proportional to the chunk size, requiring multiple textures (3 velocity, 2 pressure) of single-precision float 4 types [S1].

**Measured Performance:**
*   **Hardware:** NVIDIA GeForce GTX 950M (GTX950M) and NVIDIA GeForce GT540M [S1].
*   **CPU Approach (GTX950M, 64x64x64 chunk):**
    *   FPS: 100 FPS (10.000 ms frame time) [S1].
    *   Copy from staging texture to CPU: 23,088,969 ns [S1].
    *   Activate voxels and render: 976,276 ns [S1].
    *   Other steps (Advect, Backward advect, etc.) are significantly faster, taking < 35,000 ns each [S1].
*   **GPU-Only Approach (GT540M, 64x64x64 chunk):**
    *   FPS: 180 FPS (5.556 ms frame time) [S1].
    *   Performance improvement: Up to 1100 FPS observed on GT540M with GPU-only method for 64x64x64 chunk [S1].
    *   Original method (CPU approach) on GT540M: 30 FPS [S1].
*   **Scaling (GTX950M, CPU Approach):**
    *   32x32x32 (32,768 blocks): 401 FPS (2.494 ms) [S1].
    *   64x64x64 (262,144 blocks): 100 FPS (10.000 ms) [S1].
    *   96x96x96 (884,736 blocks): 20 FPS (50.000 ms) [S1].
    *   128x128x128 (2,097,152 blocks): 14 FPS (71.429 ms) [S1].
*   **Scaling (GT540M, GPU Approach):**
    *   32x32x32 (32,768 blocks): 652 FPS (1.534 ms) [S1].
    *   64x64x64 (262,144 blocks): 180 FPS (5.556 ms) [S1].
    *   96x96x96 (884,736 blocks): 58 FPS (17.241 ms) [S1].
    *   128x128x128 (2,097,152 blocks): 25 FPS (40.000 ms) [S1].

**Strengths and Limitations:**
*   **Strengths:** Significant performance improvement by avoiding CPU-GPU data transfer (GPU-only method). Deterministic ordering of rendered primitives via stream-out [S1].
*   **Limitations:** Does not handle obstacles. Single-chunk limitation. Performance degrades with increasing chunk size, though GPU-only method is significantly faster than CPU approach [S1].

**When to Use:**
*   Use when real-time fluid visualization in a voxel engine is required and CPU-GPU bandwidth is a bottleneck.
*   Use when obstacle interaction is not required or can be handled separately.

**When Not to Use:**
*   Do not use if obstacle collision detection is required within the simulation.
*   Do not use if multi-chunk scalability is needed without further optimization.

## Background
This method was proposed by Vlietinck in 2016, integrating recent results in GPU-accelerated fluid simulation (Crane et al. 2007 and Harris 2004) with voxel-based rendering (Miller et al. 2014) [S1]. It is based on the Navier–Stokes differential equations (Stam 2003) [S1]. The method was implemented using Direct3D, leveraging geometry shaders for rendering and compute shaders for simulation [S1]. It addresses the hurdles of integrating these technologies efficiently, specifically the performance cost of copying data between CPU and GPU [S1].

## Sources
[S1] Integrating Real-Time Fluid Simulation with a Voxel Engine, https://doi.org/10.1007/s40869-016-0020-5, fetched 2026-10-26
