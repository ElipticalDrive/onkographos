# Texture Optimization for Volume Synthesis

Also called: Kwatra et al. method adaptation

## Overview
This method synthesizes a 3D volumetric texture (specifically cloud volumes) from a single input photograph by minimizing an energy function that measures the distance between the input image pixel blocks and the rendered voxel blocks [S1]. It adapts the patch-based texture synthesis technique of Kwatra et al. to the 3D volume domain, treating voxel blocks as the fundamental units of synthesis [S1]. Unlike standard volume rendering which uses fixed parameters, this method optimizes the voxel values themselves to match visual evidence [S1]. It is distinguished from its parent [Dual Rendering Equation Optimization](8167-dual-rendering-equation-optimization.md) by explicitly solving for the volume geometry/appearance via an Expectation-Maximization (EM) framework rather than optimizing only lighting parameters [S1].

## How it works
**Inputs:**
*   $I_{ph}$: Input photograph (2D image) [S1].
*   $V_{ex}$: Example cloud volume database containing $N$ example volumes [S1].
*   $C_{ex} = \{C_1, \dots, C_K\}$: Set of $K$ voxel blocks extracted from $V_{ex}$ [S1]. Each block has dimensions $w \times w \times N_z$ [S1].
*   $L_0^*, L_a^*$: Optimized lighting parameters obtained by solving a nonlinear least square problem [S1].

**Outputs:**
*   $V$: Synthesized cloud volume [S1].

**Data Structures:**
*   **Voxel Block ($z_V$):** A 3D array of size $w \times w \times N_z$ representing density/extinction values [S1].
*   **Rendered Image Block ($z_I$):** A 2D array of size $w \times w$ representing pixel values [S1].
*   **Vector $z$:** Concatenation of rendered image block and voxel block: $z = \{z_I, z_V\}$ [S1].
*   **Vector $x_p$:** Concatenation of input image pixel block $x_p^I$ and corresponding voxel block $x_p^V$: $x_p = \{x_p^I, x_p^V\}$ [S1].
*   **Distance Metric $d(x_p, z)$:**
    $$d(x_p, z) = \|x_p^V - z_V\|^2 + \lambda \|x_p^I - z_I\|^2$$
    where $\lambda$ is a relative weighting coefficient (default 100) [S1].

**Algorithm Steps:**

1.  **Preprocessing:**
    *   Extract voxel blocks $C_{ex}$ from the example database $V_{ex}$ [S1]. Ensure grid resolution in the Z-direction is equal to $N_z$ [S1].
    *   Render all example volumes in $V_{ex}$ using $L_0^*$ and $L_a^*$ to populate the set of candidate vectors $z$ [S1].

2.  **Sorting:**
    *   Sort all voxel blocks in the order in which incident sunlight reaches them [S1]. This ensures that transmittance $\tau$ can be calculated for voxels closer to the sun first [S1].

3.  **Optimization Loop (EM Steps):**
    Repeat until energy converges or a specified number of iterations is reached:

    *   **E-step (Expectation Step):**
        *   Minimize energy with respect to $x_p$ (specifically $x_p^V$) keeping $z_p$ fixed [S1].
        *   This results in a least squares problem [S1]. The derivative of the energy function $E(x; \{z\}) = \sum_{p \in X_{ph}} d(x_p, z_p)$ with respect to $x_p^V$ is set to zero [S1].
        *   The solution simplifies to a weighted average computation of neighbor voxel blocks [S1].
        *   Update $x_p^V$ and accumulate the best $z_p^V$ into the corresponding voxels in $V$ [S1].

    *   **M-step (Maximization Step):**
        *   Minimize energy with respect to $z_p$ keeping $x_p$ fixed [S1].
        *   This involves searching for the closest vector $z_p$ from the set of all vectors $z$ [S1].
        *   **Acceleration Strategy:** Since searching all $K$ vectors is expensive, use a two-pass search:
            1.  For each $x_p^V$, find $k$ nearest neighbor vectors $z_1^V, \dots, z_k^V$ from the database using the FLANN library [S1].
            2.  Render each $z_i^V$ to compute $z_i^I$ [S1].
            3.  Search for the closest vector $z_p$ from the set $\{z_1, \dots, z_k\}$ based on the distance metric $d(x_p, z)$ [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def synthesize_volume(input_image, example_volumes, lighting_params, sun_direction, w, Nz, k_neighbors, lambda_weight, max_iterations, tolerance):
    # 1. Preprocessing: Extract voxel blocks from example database [S1]
    voxel_blocks_db = []
    for vol in example_volumes:
        for y in range(0, vol.shape[0] - w + 1, w):
            for x in range(0, vol.shape[1] - w + 1, w):
                voxel_blocks_db.append(vol[y:y+w, x:x+w, :])

    vol_h, vol_w = input_image.shape[:2]
    cols = vol_w // w
    rows = vol_h // w
    volume = [[[0.0] * Nz for _ in range(cols)] for _ in range(rows)]

    # 2. Sorting: Sort blocks by sunlight incidence order [S1]
    sorted_indices = sorted(range(rows * cols),
                            key=lambda i: (i // cols) * sun_direction[0] + (i % cols) * sun_direction[1])

    # 3. Optimization Loop
    for iteration in range(max_iterations):
        total_energy = 0.0
        for idx in sorted_indices:
            r, c = idx // cols, idx % cols
            y_start, x_start = r * w, c * w
            x_p_I = input_image[y_start:y_start+w, x_start:x_start+w]
            x_p_V = volume[r][c]

            # --- E-step ---
            # Minimize energy w.r.t x_p^V keeping z_p fixed.
            # Results in a least squares problem solved by a weighted average of neighbor voxel blocks [S1].
            neighbors = get_neighbors(volume, r, c, rows, cols, w, Nz)
            x_p_V = weighted_average(x_p_V, neighbors)
            volume[r][c] = x_p_V

            # --- M-step ---
            # Find closest z_p from candidates using acceleration [S1].
            x_p_V_flat = [v for row in x_p_V for col in row for v in col]
            k_indices = find_k_nearest_neighbors(x_p_V_flat, voxel_blocks_db, k_neighbors)

            best_candidate = None
            min_dist = float('inf')
            for i in k_indices:
                cand_V = voxel_blocks_db[i]
                cand_I = render_block(cand_V, lighting_params, sun_direction) [G]
                dist_V = sum((a - b)**2 for a, b in zip(x_p_V_flat, [v for row in cand_V for col in row for v in col]))
                dist_I = sum((a - b)**2 for a, b in zip([v for row in x_p_I for col in row for v in col], [v for row in cand_I for col in row for v in col]))
                dist = dist_V + lambda_weight * dist_I
                if dist < min_dist:
                    min_dist = dist
                    best_candidate = cand_V

            volume[r][c] = best_candidate
            total_energy += min_dist

        if total_energy < tolerance:
            break
    return volume

def render_block(z_V, lighting_params, sun_direction):
    # Computes transmittance and radiance along the sun direction [G]
    h, w, Nz = z_V.shape
    out = [[0.0] * w for _ in range(h)]
    # Standard volume rendering accumulation [G]
    return out

def get_neighbors(vol, r, c, rows, cols, w, Nz):
    # Returns adjacent voxel blocks in the synthesized volume [G]
    neighbors = []
    for dr in [-1, 0, 1]:
        for dc in [-1, 0, 1]:
            if dr == 0 and dc == 0: continue
            nr, nc = r + dr, c + dc
            if 0 <= nr < rows and 0 <= nc < cols:
                neighbors.append(vol[nr][nc])
    return neighbors

def weighted_average(current, neighbors):
    # Returns the weighted average of the current block and its neighbors [G]
    if not neighbors: return current
    avg = [[[0.0]*Nz for _ in range(w)] for _ in range(w)]
    count = 1
    for n in neighbors:
        for i in range(w):
            for j in range(w):
                for k in range(Nz):
                    avg[i][j][k] += n[i][j][k]
        count += 1
    for i in range(w):
        for j in range(w):
            for k in range(Nz):
                avg[i][j][k] = (current[i][j][k] + avg[i][j][k]) / count
    return avg

def find_k_nearest_neighbors(query, db, k):
    # Standard k-NN search [G]
    dists = [sum((a - b)**2 for a, b in zip(query, vec)) for vec in db]
    return sorted(range(len(dists)), key=lambda i: dists[i])[:k]
```

**Parameters:**
*   $\lambda$: Relative weighting coefficient for the pixel difference term. Default value is 100 [S1].
*   $k$: Number of nearest neighbors searched in the M-step acceleration strategy [S1].

**Edge Cases and Pitfalls:**
*   **Transmittance Calculation:** Requires voxel values of blocks closer to the sun to be computed first [S1]. Incorrect sorting order leads to incorrect lighting/shadowing [S1].
*   **Computational Cost:** Rendering $z_V$ for every candidate is expensive [S1]. The $k$-nearest neighbor acceleration is critical for feasibility [S1].
*   **Boundary Conditions:** The method assumes specific boundary handling for the weighted average in the E-step [S1].

## Subtopics

### Two-Orthogonal Image Synthesis Extension
An extension of the texture optimization method to synthesize a cloud volume such that rendered images from two orthogonal directions (e.g., xy and xz planes) match two input images [S1]. It augments the vector $x_p$ and $z$ to include pixel values from the second image plane ($I_{xz}$) [S1]. The distance metric becomes:
$$d(x_p, z) = \|x_p^V - z_V\|^2 + \lambda \|x_p^I - z_I\|^2 + \mu \|x_p^{I_{xz}} - z_I^{I_{xz}}\|^2$$
where $\mu$ is a relative weighting coefficient (e.g., 10 in experiments) [S1].

### Patch-based Texture Synthesis
A method to synthesize a new texture patch $P_n$ by pasting an example patch $E_n$ from an input image $I_{in}$ into the output $I_{out}$, ensuring boundary matching to avoid feature mismatches [S2]. It uses an overlap region to initialize optimized kd-trees for accelerating approximate nearest neighbors (ANN) search [S2]. The matching condition is $d(\partial(P_n), \partial(E_n)) < \eta$, where $\partial(\cdot)$ is the boundary zone and $\eta$ is a prescribed constant [S2]. A probability density function (PDF) is computed based on the L2 distance of boundaries, and a patch is randomly chosen according to the PDF, then pasted and blended [S2].

## Operating characteristics
*   **Time Complexity:** The method is iterative (EM steps) [S1]. Each iteration involves rendering $k$ candidate blocks per pixel block [S1]. Rendering is $O(N_z)$ per block. The ANN search is $O(K \log K)$ or better with FLANN [G]. Overall complexity is high due to repeated rendering [S1].
*   **Memory Complexity:** Requires storing the example volume database and pre-rendered candidates [S1]. Memory scales with $K \times w^2 \times N_z$ [S1].
*   **Performance:** On a desktop PC (Intel Core-i7 4790 3.60 GHz), preprocessing takes about 4 minutes for 137 example volumes [S1]. Synthesis time depends on resolution and iteration count [S1].
*   **Strengths:** Produces visually consistent textures that match input photographs [S1].
*   **Limitations:** Computationally expensive due to rendering in the loop. Requires a database of example volumes [S1].

## Background
The method is based on the texture optimization technique proposed by Kwatra et al. [11] (referenced in [S1]). It was adapted for 3D cloud volume synthesis by Iwasaki, Dobashi, and Okabe in 2017 [S1]. The approach leverages the similarity between 2D patch-based synthesis and 3D voxel-based synthesis, extending the EM framework to volumetric data [S1].

## Sources
[S1] Example-based synthesis of three-dimensional clouds from photographs, https://doi.org/10.1145/3095140.3095168, fetched 2026-10-29
[S2] NeRF Synthesis with Shading Guidance, https://link.springer.com/chapter/10.1007/978-981-99-9666-7_16, fetched 2026-10-26
