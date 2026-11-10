# Capacity-safe slab-based Marching Cubes

Also called: Capacity-safe slab-based isosurface extraction, Capacity-safe slab-based isosurface extraction pipeline, Capacity-safe tiled Marching Cubes, Two-pass tiled Marching Cubes, inclusive slab partitioning with exact allocation, slab-based two-pass GPU isosurface extraction

## Overview
This method is a GPU-accelerated isosurface extraction pipeline that partitions the z-axis into slabs and uses a two-pass counting/exact allocation strategy to prevent output-buffer overflow and eliminate boundary cracks in large scalar grids [S1]. It differs from standard Marching Cubes by resolving topological discontinuities via boundary-consistent inclusive slab indexing, and differs from [GPU Marching Cubes with Histogram Pyramid](2317-gpu-marching-cubes-with-histogram-pyramid.md) and [Primal Grid Marching Cubes](1211-primal-grid-marching-cubes.md) by specializing counting and compaction for tightly coupled extraction and rendering [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D scalar volume $V$ with dimensions $N_x \times N_y \times N_z$, and an isovalue $\tau$ [S1].
*   **Output:** A Vertex Buffer Object (VBO) containing position and normal arrays for the extracted isosurface mesh [S1].

**Data Structures:**
*   **Volume $V$:** Array of size $N_x N_y N_z$ [S1].
*   **Count Array:** Temporary storage for vertex counts per cell [S1].
*   **Offset Array:** Temporary storage for exclusive prefix-sum results [S1].
*   **VBO:** Interoperable buffer with capacity $N$ vertices. Each vertex stores a 3D position and a 3D normal [S1].

**Parameters:**
*   **Slab Thickness $S$:** Recommended range $[48, 96]$ for a $512^3$ grid [S1].
*   **Isovalue $\tau$:** Scalar threshold for extraction [S1].

**Algorithm Steps:**

1.  **Slab Partitioning:**
    Partition the z-axis into slabs $s = 0, \dots, S_{total}-1$. For each slab $s$, define the set of valid cell indices $k$ as the closed interval $\mathcal{K}_s = \{k \mid k_{begin}(s) \le k \le k_{end}(s)\}$ [S1].
    *   $k_{begin}(s) = s \times S$
    *   $k_{end}(s) = \min((s+1) \times S - 1, N_z - 2)$ [S1].
    *   If $N_z - 1$ is not divisible by $S$, the final slab is processed as a partial slab without padding [S1].
    *   This inclusive indexing ensures that the $k+1$ corner access required by Marching Cubes remains valid at slab boundaries, preventing cracks [S1].

2.  **Pass 1: Classification and Scan (Counting):**
    Initialize global vertex count $N \leftarrow 0$ [S1].
    For each slab $s$:
    a.  For each cell $(i, j, k)$ where $k \in \mathcal{K}_s$:
        i.  Compute case index $c = \text{caseIndex}(V, i, j, k, \tau)$ using the logic from [Marching Cubes Lookup and Configuration](2518-marching-cubes-lookup-and-configuration.md) [S1].
        ii. Retrieve vertex count $n_{ijk} = \text{numVerts}[c]$ [S1].
        iii. Store $n_{ijk}$ in `counts` array at linear index $\ell = \text{lin}(i, j, k) = (k N_y + j) N_x + i$ [S1].
    b.  Compute exclusive prefix-sum (scan) over `counts` for the slab to generate `offsets` array [S1].
    c.  Calculate slab vertex total $N_s = \text{offsets}[\text{last}] + \text{counts}[\text{last}]$ [S1].
    d.  Set slab base offset $\text{base}[s] = N$ [S1].
    e.  Update global count $N \leftarrow N + N_s$ [S1].

3.  **Allocation:**
    Allocate the interoperable VBO with exactly $N$ vertex capacity [S1].

4.  **Pass 2: Generation:**
    For each slab $s$:
    a.  For each cell $(i, j, k)$ where $k \in \mathcal{K}_s$:
        i.  Compute case index $c = \text{caseIndex}(V, i, j, k, \tau)$ [S1].
        ii. If $\text{numVerts}[c] \neq 0$:
            1.  Compute edge intersection vertices $\text{edgeVerts}[0..11]$ via linear interpolation of $V$ at $\tau$ [S1].
            2.  Calculate destination address $\text{dst} = \text{base}[s] + \text{offsets}[\text{lin}(i, j, k)]$ [S1].
            3.  Iterate through triangle edge list $\text{triTable}[c][t..t+2]$ for $t = 0$ to $\text{numVerts}[c]-1$ with step 3 [S1].
            4.  Let $(e_0, e_1, e_2) = \text{triTable}[c][t..t+2]$ [S1].
            5.  If $e_0 < 0$ or $e_1 < 0$ or $e_2 < 0$, break the loop (handle empty triangles) [S1].
            6.  Retrieve vertices $\mathbf{p}_0 = \text{edgeVerts}[e_0]$, $\mathbf{p}_1 = \text{edgeVerts}[e_1]$, $\mathbf{p}_2 = \text{edgeVerts}[e_2]$ [S1].
            7.  Compute face normal $\mathbf{n} = \frac{(\mathbf{p}_1 - \mathbf{p}_0) \times (\mathbf{p}_2 - \mathbf{p}_0)}{\|(\mathbf{p}_1 - \mathbf{p}_0) \times (\mathbf{p}_2 - \mathbf{p}_0)\|}$ to enforce counterclockwise winding [S1].
            8.  Write positions $\{\mathbf{p}_0, \mathbf{p}_1, \mathbf{p}_2\}$ to `pos[dst:dst+2]` and normals $\{\mathbf{n}, \mathbf{n}, \mathbf{n}\}$ to `nrm[dst:dst+2]` [S1].
            9.  Increment $\text{dst} \leftarrow \text{dst} + 3$ [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def capacity_safe_slab_marching_cubes(V, Nx, Ny, Nz, tau, S):
    # Standard MC lookup tables [G]
    num_verts_table = [...] # 256 entries, standard MC vertex counts
    tri_table = [...]       # 256 entries, standard MC edge indices

    def compute_case_index(V, i, j, k, tau):
        # Standard MC case index from sign configuration of 8 corners [G]
        idx = 0
        for bit in range(8):
            if V[i + (bit & 1), j + ((bit >> 1) & 1), k + ((bit >> 2) & 1)] >= tau:
                idx |= (1 << bit)
        return idx

    def compute_edge_intersections(V, i, j, k, tau):
        # Standard MC edge linear interpolation [G]
        verts = [None] * 12
        # Edge 0: (i,j,k) to (i+1,j,k)
        v0, v1 = V[i, j, k], V[i+1, j, k]
        if v0 != v1: verts[0] = (v0, v1, k) # Simplified placeholder for 3D coord
        # ... (full interpolation omitted for brevity, standard MC step)
        return verts

    # 1. Slab Partitioning and Pass 1: Counting
    global_N = 0
    slab_bases = []
    num_slabs = (Nz - 1 + S - 1) // S

    for s in range(num_slabs):
        k_start = s * S
        k_end = min((s + 1) * S - 1, Nz - 2)
        slab_cell_count = (k_end - k_start + 1) * (Ny - 1) * (Nx - 1)

        counts = [0] * slab_cell_count
        offsets = [0] * slab_cell_count

        # Process cells in this slab
        for k in range(k_start, k_end + 1):
            for j in range(Ny - 1):
                for i in range(Nx - 1):
                    c = compute_case_index(V, i, j, k, tau)
                    n_verts = num_verts_table[c]
                    lin_idx = (k * Ny + j) * Nx + i - (k_start * Ny * Nx) # Local index for slab array
                    counts[lin_idx] = n_verts

        # Exclusive prefix-sum (scan)
        if slab_cell_count > 0:
            offsets[0] = 0
            for idx in range(1, slab_cell_count):
                offsets[idx] = offsets[idx - 1] + counts[idx - 1]

        # Slab vertex total
        last_idx = slab_cell_count - 1
        Ns = offsets[last_idx] + counts[last_idx]

        # Base offset for this slab in the global VBO
        base_s = global_N
        slab_bases.append(base_s)
        global_N += Ns

    # 2. Allocation
    vbo_pos = [0] * (global_N * 3)
    vbo_nrm = [0] * (global_N * 3)

    # 3. Pass 2: Generation
    for s in range(num_slabs):
        k_start = s * S
        k_end = min((s + 1) * S - 1, Nz - 2)
        base_s = slab_bases[s]

        for k in range(k_start, k_end + 1):
            for j in range(Ny - 1):
                for i in range(Nx - 1):
                    c = compute_case_index(V, i, j, k, tau)
                    n_verts = num_verts_table[c]

                    if n_verts > 0:
                        edge_verts = compute_edge_intersections(V, i, j, k, tau)
                        lin_idx = (k * Ny + j) * Nx + i - (k_start * Ny * Nx)
                        dst = base_s + offsets[lin_idx]

                        tri_idx = 0
                        while tri_idx < n_verts:
                            e0 = tri_table[c][tri_idx]
                            e1 = tri_table[c][tri_idx + 1]
                            e2 = tri_table[c][tri_idx + 2]

                            if e0 < 0 or e1 < 0 or e2 < 0:
                                break

                            p0 = edge_verts[e0]
                            p1 = edge_verts[e1]
                            p2 = edge_verts[e2]

                            # Compute normal
                            v1 = p1 - p0
                            v2 = p2 - p0
                            n = normalize(cross(v1, v2))

                            # Write to VBO
                            vbo_pos[dst * 3 : (dst + 1) * 3] = p0
                            vbo_pos[(dst + 1) * 3 : (dst + 2) * 3] = p1
                            vbo_pos[(dst + 2) * 3 : (dst + 3) * 3] = p2

                            vbo_nrm[dst * 3 : (dst + 1) * 3] = n
                            vbo_nrm[(dst + 1) * 3 : (dst + 2) * 3] = n
                            vbo_nrm[(dst + 2) * 3 : (dst + 3) * 3] = n

                            dst += 3
                            tri_idx += 3

    return vbo_pos, vbo_nrm
```

## Operating characteristics
*   **Time Complexity:** $O(M)$ for both passes, where $M = (N_x-1)(N_y-1)(N_z-1)$ is the number of cells [S1].
*   **Memory Complexity:** Requires storage for the volume, temporary count/offset arrays, and the output VBO of size $N$ vertices [S1]. The method uses less than 11 GB GPU memory for a $512^3$ grid [S1].
*   **Performance:** On electromagnetic-potential fields with resolution $512^3$, the method generates approximately 42.81 million vertices in 44.7 ms and renders the resulting geometry at 28.6 FPS [S1].
*   **Strengths:** Guarantees exact buffer allocation (no over-provisioning), prevents buffer overruns, and eliminates boundary cracks through inclusive slab indexing [S1]. Supports zero-copy streaming to OpenGL via CUDA-OpenGL interoperability [S1].
*   **Limitations:** Less suitable for sharp-feature preservation compared to Dual Contouring or MC33 [S1]. Limited to volumes that fit within a single GPU's memory [S1].

## Background
The method was introduced in a paper presenting a GPU-accelerated visualization framework for interactive exploration of fusion-plasma simulation data [S1]. It addresses the challenges of large-scale scalar grids by combining inclusive slab partitioning with a two-pass counting and exact allocation strategy [S1]. The approach is designed for integration into engines like Unreal Engine for explicit-isosurface inspection and particle-trajectory visualization [S1].

## Sources
[S1] Visual tools for 3D plasma simulation: GPU-based isosurface extraction ..., https://pmc.ncbi.nlm.nih.gov/articles/PMC13674866/, fetched 2026-10-30
