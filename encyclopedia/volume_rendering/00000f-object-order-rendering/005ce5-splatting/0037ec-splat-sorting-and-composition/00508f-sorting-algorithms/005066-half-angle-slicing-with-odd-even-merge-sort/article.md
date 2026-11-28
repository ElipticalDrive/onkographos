# Half-Angle Slicing with Odd-Even Merge Sort

Also called: Half angle slicing technique

## Overview
Half-angle slicing is a GPU-based volume rendering technique for particle systems that sorts particles along a view-dependent axis (the half-angle between the light and view directions) using an odd-even merge sort, then renders them in slices to handle transparency and shadows correctly [S1, unofficial]. It sets itself apart from standard front-to-back splatting by decoupling the sorting axis from the view direction, allowing for efficient accumulation of opacity from the light's perspective to compute accurate volumetric shadows in a single pass [S1, unofficial]. Unlike [Bitonic Sort](14217-bitonic-sort.md) which sorts strictly by depth, this method sorts by a projected coordinate that aligns with the light transport, enabling the "slicing" strategy where particles are rendered in batches to build a light-space opacity texture [S1, unofficial].

## How it works
The method operates in three main phases: simulation, GPU-based sorting, and slice-based rendering with opacity accumulation [S1, unofficial].

### Inputs and Outputs
*   **Inputs:** Particle positions, velocities, lifetimes, and spawn data; camera view/projection matrices; light view/projection matrices; time delta [S1, unofficial].
*   **Outputs:** Final composited image with volumetric shadows; opacity texture for light-space accumulation [S1, unofficial].

### Data Structures
The implementation relies on specific Framebuffer Objects (FBOs) and textures, typically stored in a grid layout for efficient GPU access [S1, unofficial].

1.  **Simulation FBOs (Ping-Pong):**
    *   Two FBOs, `mSimulationFbo[0]` and `mSimulationFbo[1]` [S1, unofficial].
    *   Size: `PARTICLES` x `PARTICLES` (256x256) [S1, unofficial].
    *   Internal Format: `GL_RGBA32F` [S1, unofficial].
    *   Data Type: `GL_FLOAT` [S1, unofficial].
    *   Purpose: Store particle state (position, velocity, etc.) for simulation updates [S1, unofficial].
    *   No depth buffer [S1, unofficial].

2.  **Particle Data Texture:**
    *   Size: `PARTICLES` x `PARTICLES` (256x256) [S1, unofficial].
    *   Format: `GL_RGBA32F` [S1, unofficial].
    *   Layout: Each texel `vec4` contains `position.x`, `position.y`, `position.z`, and `lifetime` (or spawn data) [S1, unofficial].

3.  **Opacity FBO:**
    *   One FBO [S1, unofficial].
    *   Size: `OPACITY_TEXTURE_RESOLUTION` x `OPACITY_TEXTURE_RESOLUTION` (1024x1024) [S1, unofficial].
    *   Internal Format: `GL_RGBA` [S1, unofficial].
    *   Data Type: `GL_UNSIGNED_BYTE` [S1, unofficial].
    *   Purpose: Accumulate opacity from the light's perspective [S1, unofficial].
    *   No depth buffer [S1, unofficial].

### Parameters
*   `PARTICLES`: Square root of total particle count. Default: 256 [S1, unofficial].
*   `mMaxParticleCount`: Total particles = `PARTICLES` * `PARTICLES`. Default: 65536 [S1, unofficial].
*   `SLICES`: Number of depth slices for rendering. Default: 128 [S1, unofficial].
*   `OPACITY_TEXTURE_RESOLUTION`: Resolution of the light-space opacity texture. Default: 1024 [S1, unofficial].

### Algorithm Steps

1.  **Calculate Half-Vector for Sorting:**
    Determine the sorting axis based on the angle between the view direction ($\mathbf{v}$) and the light direction ($\mathbf{l}$) [S1, unofficial].
    *   If $\mathbf{v} \cdot \mathbf{l} > 0.0$:
        *   $\mathbf{halfVector} = \text{normalize}(\mathbf{l} + \mathbf{v})$
        *   $mFlipped = \text{false}$
    *   Else:
        *   $\mathbf{halfVector} = \text{normalize}(\mathbf{l} - \mathbf{v})$
        *   $mFlipped = \text{true}$
    *   [S1, unofficial]

2.  **Simulate Particle Movement:**
    *   Bind the previous frame's position texture (`mSimulationFbo[1 - mCurrent]`) to texture unit 0 [S1, unofficial].
    *   Bind the spawn data texture to texture unit 1 [S1, unofficial].
    *   Execute the simulation shader for 1 frame (or `BASE_LIFETIME / PRESIMULATION_DELTA_TIME` frames on the first frame to initialize) [S1, unofficial].
    *   Update `mCurrent` to swap ping-pong buffers [S1, unofficial].

3.  **Execute Odd-Even Merge Sort:**
    Sort the particles based on their projection onto the $\mathbf{halfVector}$ [S1, unofficial].
    *   Calculate total sort steps:
        $$ m_{\text{TotalSortSteps}} = \frac{\log_2(m_{\text{MaxParticleCount}}) \cdot (\log_2(m_{\text{MaxParticleCount}}) + 1)}{2} $$
    *   [S1, unofficial]
    *   Loop `mTotalSortSteps` times:
        *   Swap read/write FBOs: `mCurrent = 1 - mCurrent` [S1, unofficial].
        *   Decrement `mSortPass`. If `mSortPass < 0`:
            *   Increment `mSortStage`.
            *   Set `mSortPass = mSortStage` [S1, unofficial].
        *   Bind source texture to unit 0 [S1, unofficial].
        *   Pass uniforms to sort shader:
            *   `uPass = float(1 << mSortPass)`
            *   `uStage = float(1 << mSortStage)` [S1, unofficial].
        *   Decrement `mSortStepsLeft`. If it reaches 0, reset counters (`mSortPass`, `mSortStage`, `mSortStepsLeft`) [S1, unofficial].

4.  **Render Slices (View Pass):**
    *   Loop `i` from 0 to `SLICES - 1` [S1, unofficial].
    *   Determine particle range:
        *   `startIndex = i * (mMaxParticleCount / SLICES)`
        *   `count = mMaxParticleCount / SLICES` [S1, unofficial].
    *   Set camera matrices to view space [S1, unofficial].
    *   Bind opacity texture to unit 1 [S1, unofficial].
    *   Set blend mode:
        *   If `mFlipped` is true: `GL_ONE_MINUS_DST_ALPHA`, `GL_ONE`
        *   Else: `GL_ONE`, `GL_ONE_MINUS_SRC_ALPHA` [S1, unofficial].
    *   Draw the current slice of particles using point sprites [S1, unofficial].

5.  **Opacity Pass (Light View):**
    *   Bind the opacity FBO [S1, unofficial].
    *   Set matrices to light camera (light view/projection) [S1, unofficial].
    *   Draw the same particle slice with blend mode `GL_ONE`, `GL_ONE_MINUS_SRC_ALPHA` to accumulate opacity into the opacity texture [S1, unofficial].

6.  **Final Composite:**
    *   Draw the floor and fullscreen quad [S1, unofficial].
    *   Bind the accumulated opacity texture (from step 5) to unit 2 [S1, unofficial].
    *   Use the light view projection matrix to sample the opacity texture for shadow calculation [S1, unofficial].
    *   Blend mode: `GL_ONE_MINUS_DST_ALPHA`, `GL_ONE` [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

# Standard GPU state operations (defined per implementation rules)
GL_ONE = 1
GL_ONE_MINUS_SRC_ALPHA = 771
GL_ONE_MINUS_DST_ALPHA = 774

def bind_texture(unit, tex):
    texture_unit[unit] = tex

def set_blend(src_factor, dst_factor):
    blend_src = src_factor
    blend_dst = dst_factor

def draw_particles(start_index, count):
    for idx in range(start_index, start_index + count):
        rasterize_particle(idx)

def bind_framebuffer(fbo):
    current_fbo = fbo

def set_camera_matrices(view_mat, proj_mat):
    active_view_matrix = view_mat
    active_proj_matrix = proj_mat

def draw_floor_and_quad():
    rasterize_floor()
    rasterize_fullscreen_quad()

def half_angle_slicing(view_dir, light_dir, dt, time):
    # 1. Calculate Half-Vector
    dot_val = math.dot(math.normalize(view_dir), math.normalize(light_dir))
    if dot_val > 0.0:
        half_vec = math.normalize(light_dir + view_dir)
        flipped = False
    else:
        half_vec = math.normalize(light_dir - view_dir)
        flipped = True

    # 2. Simulation
    bind_texture(0, sim_fbo[1 - current])
    bind_texture(1, spawn_data_tex)
    run_simulation_shader(1) # or BASE_LIFETIME / PRESIMULATION_DELTA_TIME on first frame
    current = 1 - current

    # 3. Odd-Even Merge Sort
    total_steps = (math.log2(max_particles) * (math.log2(max_particles) + 1)) / 2
    sort_pass = 0
    sort_stage = 0
    sort_steps_left = total_steps

    for _ in range(total_steps):
        current = 1 - current
        sort_pass -= 1
        if sort_pass < 0:
            sort_stage += 1
            sort_pass = sort_stage
        bind_texture(0, sim_fbo[current])
        pass_uniforms(u_pass=1 << sort_pass, u_stage=1 << sort_stage)
        run_sort_shader()
        sort_steps_left -= 1
        if sort_steps_left == 0:
            sort_pass = 0
            sort_stage = 0
            sort_steps_left = total_steps

    # 4. Render Slices & 5. Opacity Accumulation
    for i in range(slices):
        start_idx = i * (max_particles // slices)
        count = max_particles // slices

        # View Pass
        set_camera_matrices(view_matrix, proj_matrix)
        bind_texture(1, opacity_tex)
        if flipped:
            set_blend(GL_ONE_MINUS_DST_ALPHA, GL_ONE)
        else:
            set_blend(GL_ONE, GL_ONE_MINUS_SRC_ALPHA)
        draw_particles(start_idx, count)

        # Light View Pass (Opacity Accumulation)
        bind_framebuffer(opacity_fbo)
        set_camera_matrices(light_view_matrix, light_proj_matrix)
        set_blend(GL_ONE, GL_ONE_MINUS_SRC_ALPHA)
        draw_particles(start_idx, count)
        bind_framebuffer(default_fbo)

    # 6. Final Composite
    set_camera_matrices(view_matrix, proj_matrix)
    bind_texture(2, opacity_tex)
    set_blend(GL_ONE_MINUS_DST_ALPHA, GL_ONE)
    draw_floor_and_quad()
```

### Edge Cases and Pitfalls
*   **Precision:** The sorting relies on floating-point comparisons [S1, unofficial].
*   **Boundaries:** When sampling the opacity texture in the final composite, check for coordinates outside [0, 1] to avoid wrapping artifacts [S1, unofficial].
*   **Degenerate Input:** If particles are exactly on the sorting plane, the sort stability depends on the implementation of the odd-even merge sort shader [S1, unofficial].

## Subtopics

### Multiple Scattering Phase Function Lighting Approximation
This variant extends the half-angle slicing technique to handle the gradual angular spreading of light during multiple scattering events, which is essential for capturing fine angularly dependent effects with large forward-scattering particles [S2]. It models the angular dispersion of light experiencing its first few scattering events for efficient single-pass rendering on programmable graphics hardware [S2]. A key pitfall in this approximation is the presence of the light term $L_\lambda$ on both sides of the governing equation (Equation 7 in the source), requiring separation of intensity and distribution to solve iteratively or approximately [S2].

## Operating characteristics
*   **Time Complexity:** The sorting phase uses odd-even merge sort, which requires $O(\log^2 N)$ steps for $N$ particles [S1, unofficial]. The rendering phase loops over `SLICES` batches [S1, unofficial].
*   **Memory Complexity:** Requires $O(N)$ storage for particle data and $O(R^2)$ for the opacity texture, where $R$ is the opacity texture resolution [S1, unofficial].
*   **Performance:** The method is designed for GPU acceleration, leveraging parallel sorting and rasterization [S1, unofficial].
*   **Strengths:** Handles volumetric shadows efficiently; works well with particle systems [S1, unofficial].
*   **Limitations:** Sorting is view-dependent and must be recomputed if the camera or light moves significantly; limited to particle-based volumes [S1, unofficial].

## Background
The half-angle slicing technique was originally described by Kniss et al. [KPHE02] for rendering volumetric effects [S2]. The specific implementation using GPU-based odd-even merge sort and ping-pong FBOs for particle shadows is based on a webGL implementation by David Li, which in turn was ported from a NVIDIA article on smoke particles [S1, unofficial]. The technique is used in real-time volume rendering, particularly for atmospheric phenomena and particle systems where shadows are required [S1, unofficial][S2].

## Sources
[S1] GitHub - zk8085454/VolumeParticleShadows: Half angle slicing technique ..., https://github.com/zk8085454/VolumeParticleShadows/, fetched 2026-10-28 (unofficial)
[S2] Efficient Rendering of Atmospheric Phenomena, https://doi.org/10.2312/egwr/egsr04/375-386, fetched 2026-10-26
