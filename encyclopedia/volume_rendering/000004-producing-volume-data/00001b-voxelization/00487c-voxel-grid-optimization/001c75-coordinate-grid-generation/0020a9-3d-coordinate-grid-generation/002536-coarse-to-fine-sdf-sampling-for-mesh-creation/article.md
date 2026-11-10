# Coarse-to-Fine SDF Sampling for Mesh Creation

Also called: Speedup Mesh Creation via Coarse-to-Fine SDF Sampling

## Overview
This method accelerates mesh generation from implicit surfaces by reducing neural network inference calls for Signed Distance Function (SDF) evaluation. It computes SDFs on a coarse grid, classifies regions as inside, outside, or near the surface, and only evaluates the network at full resolution for points near the zero-crossing boundary. This approach builds on [3D Coordinate Grid Generation](2333-3d-coordinate-grid-generation.md) to provide sample coordinates. [S1, unofficial]

## How it works
The method takes a trained decoder network, a latent vector representing the shape, and a target resolution $N$ as inputs, and outputs a PLY mesh file representing the isosurface. [S1, unofficial]

### Data Structures
- **Samples Array**: A 2D array of shape $(M, 4)$, where $M$ is the number of samples. Columns 0–2 store $(x, y, z)$ coordinates, and column 3 stores the SDF value. [S1, unofficial]
- **Classification Indices**: Integer arrays `pos_samples`, `neg_samples`, and `valid_samples` indexing into the samples array. [S1, unofficial]

### Parameters
- $N$: Target resolution (e.g., 1024). [S1, unofficial]
- `voxel_origin`: Minimum corner of the voxel space, set to $[-1, -1, -1]$. [S1, unofficial]
- `max_batch`: Maximum number of points processed per neural network inference pass. [S1, unofficial]
- `threshold`: Classification tolerance, set to `voxel_size_half`. [S1, unofficial]

### Steps
1. **Initialize Coarse Grid Parameters**: Set `voxel_origin` to $[-1, -1, -1]$. Calculate coarse resolution $N_{half} = N/2$. Calculate coarse voxel size as $\text{voxel\_size}_{half} = \frac{2.0}{N_{half} - 1}$. [S1, unofficial]
2. **Generate Coarse Samples**: Generate sample coordinates for the half-resolution grid using `get_samples(N_{half}, voxel\_origin, voxel\_size_{half})`. [S1, unofficial]
3. **Infer Coarse SDF Values**: Evaluate the decoder network on the coarse samples in batches of `max_batch` to obtain $\text{sdf\_values}_{half}$. [S1, unofficial]
4. **Upsample to Full Resolution**: Interpolate $\text{sdf\_values}_{half}$ to full resolution $N$ using cubic upsampling: $\text{sdf\_values}_{upsampled} = \text{upsample\_cubic}(\text{sdf\_values}_{half}, N_{half}, N)$. [S1, unofficial]
5. **Classify Regions**: Compare $\text{sdf\_values}_{upsampled}$ against `voxel_size_half`. Points with values $> \text{voxel\_size}_{half}$ are `pos_samples`, values $< -\text{voxel\_size}_{half}$ are `neg_samples`, and values within $[-\text{voxel\_size}_{half}, \text{voxel\_size}_{half}]$ are `valid_samples`. [S1, unofficial]
6. **Generate Full Resolution Coordinates**: Calculate full voxel size as $\text{voxel\_size} = \frac{2.0}{N - 1}$. Generate sample coordinates for the full resolution grid using `get_samples(N, voxel\_origin, voxel\_size)`. [S1, unofficial]
7. **Assign Pre-computed SDF Values**: Initialize the SDF column (index 3) of the full-resolution samples array to $0.0$. Set `samples[pos_samples, 3]` to $0.1$ and `samples[neg_samples, 3]` to $-0.1$. Leave `samples[valid_samples, 3]` unset for inference. [S1, unofficial]
8. **Infer Fine SDF Values**: Evaluate the decoder network only on `valid_samples` coordinates. Assign the returned SDF values to `samples[valid_samples, 3]`. [S1, unofficial]
9. **Extract and Reshape**: Extract the third column of the samples array and reshape it into an $N \times N \times N$ tensor. [S1, unofficial]
10. **Mesh Generation**: Convert the SDF tensor to a PLY mesh using `convert_sdf_samples_to_ply(SDF\_tensor, voxel\_origin, voxel\_size, output\_filename)`. [S1, unofficial]

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def create_mesh_speedup(decoder, latent_vec, N, max_batch, output_filename):
    # Step 1: Initialize coarse grid parameters
    N_half = N // 2
    voxel_origin = [-1.0, -1.0, -1.0]
    voxel_size_half = 2.0 / (N_half - 1)
    voxel_size = 2.0 / (N - 1)

    # Step 2: Generate coarse sample coordinates
    samples_half = generate_grid(N_half, voxel_origin, voxel_size_half)

    # Step 3: Infer coarse SDF values
    sdf_values_half = batch_infer(decoder, latent_vec, samples_half, max_batch)

    # Step 4: Upsample to full resolution
    sdf_values_upsampled = cubic_upsample(sdf_values_half, N_half, N)

    # Step 5: Classify regions
    pos_samples, neg_samples, valid_samples = classify_valid(sdf_values_upsampled, voxel_size_half)

    # Step 6: Generate full resolution coordinates
    samples_full = generate_grid(N, voxel_origin, voxel_size)

    # Step 7: Assign pre-computed SDF values
    samples_full[:, 3] = 0.0
    samples_full[pos_samples, 3] = 0.1
    samples_full[neg_samples, 3] = -0.1

    # Step 8: Infer fine SDF values for valid samples
    if len(valid_samples) > 0:
        valid_coords = samples_full[valid_samples, :3]
        sdf_values_valid = batch_infer(decoder, latent_vec, valid_coords, max_batch)
        samples_full[valid_samples, 3] = sdf_values_valid

    # Step 9: Extract and reshape
    sdf_tensor = samples_full[:, 3].reshape(N, N, N)

    # Step 10: Convert to mesh
    convert_to_ply(sdf_tensor, voxel_origin, voxel_size, output_filename)

def generate_grid(res, origin, size):
    # [G] Standard 3D grid generation
    coords = np.stack(np.meshgrid(
        np.linspace(origin[0], origin[0] + (res - 1) * size, res),
        np.linspace(origin[1], origin[1] + (res - 1) * size, res),
        np.linspace(origin[2], origin[2] + (res - 1) * size, res),
        indexing='ij'
    ), axis=-1).reshape(-1, 3)
    return np.hstack([coords, np.zeros((len(coords), 1))])

def batch_infer(decoder, latent, coords, max_batch):
    # [G] Standard batched inference
    values = []
    for i in range(0, len(coords), max_batch):
        values.append(decoder(coords[i:i+max_batch], latent))
    return np.concatenate(values)

def cubic_upsample(grid, old_res, new_res):
    # [G] Standard trilinear/cubic interpolation
    from scipy.ndimage import zoom
    return zoom(grid, new_res / old_res, order=3)

def classify_valid(sdf_grid, threshold):
    # [G] Standard threshold classification
    flat = sdf_grid.flatten()
    pos = np.where(flat > threshold)[0]
    neg = np.where(flat < -threshold)[0]
    valid = np.where(np.abs(flat) <= threshold)[0]
    return pos, neg, valid

def convert_to_ply(sdf_tensor, origin, size, filename):
    # [G] Standard marching cubes to PLY export
    from skimage import measure
    verts, faces, _, _ = measure.marching_cubes(sdf_tensor, level=0.0)
    with open(filename, 'w') as f:
        f.write("ply\nformat ascii 1.0\nelement vertex {}\nproperty float x\nproperty float y\nproperty float z\nend_header\n".format(len(verts)))
        for v in verts:
            f.write(f"{v[0]} {v[1]} {v[2]}\n")
```

### Edge Cases and Pitfalls
- **Precision**: Cubic upsampling may blur sharp features; threshold selection directly impacts surface fidelity. [S1, unofficial]
- **Boundaries**: `voxel_origin` and `voxel_size` must align with the decoder's training domain to avoid extrapolation artifacts. [S1, unofficial]
- **Degenerate Input**: If `valid_samples` is empty, the method skips fine inference and uses the upsampled grid directly. [G]

## Operating characteristics
- **Time Complexity**: Reduces neural network inferences from $N^3$ to approximately $(N/2)^3 + N_{valid}$, where $N_{valid}$ is the number of points near the surface. [S1, unofficial]
- **Memory Complexity**: Requires storing the coarse SDF grid ($N_{half}^3$), the full-resolution coordinate grid ($N^3$), and the final SDF tensor ($N^3$). [S1, unofficial]
- **Strengths**: Significantly lowers computational cost of SDF evaluation, which is typically the bottleneck in implicit surface reconstruction. [S1, unofficial]
- **Limitations**: Accuracy depends on upsampling quality and threshold tuning; an overly large threshold increases inference cost, while an overly small threshold risks aliasing or incomplete surfaces. [S1, unofficial]
- **When to use**: Generating high-resolution meshes from neural implicit representations where inference is expensive. [S1, unofficial]
- **When not to use**: Surfaces with extreme geometric complexity or thin features requiring dense sampling everywhere. [S1, unofficial]

## Background
This method is described in the unofficial repository [B1ueber2y/DIST-Renderer](https://github.com/B1ueber2y/DIST-Renderer) [S1, unofficial]. It is integrated into a color training and mesh creation pipeline for 3D shapes, optimizing the mesh generation step by deferring expensive network evaluations to the surface boundary. [S1, unofficial]

## Sources
[S1] B1ueber2y/DIST-Renderer, https://github.com/B1ueber2y/DIST-Renderer, fetched 2026-10-26 (unofficial)
