# FlowNet


## Overview
FlowNet is a deep learning framework for clustering and selecting representative streamlines or stream surfaces from complex flow fields [S1, unofficial]. It solves the problem of manual feature extraction being tedious and brittle by voxelizing flow structures into binary volumes and training a 3D CNN autoencoder to produce feature descriptors [S1, unofficial]. It sets itself apart from its siblings, [3D Shape Autoencoder](6689-3d-shape-autoencoder.md) and [Deep Implicit Volume Compression](6807-deep-implicit-volume-compression.md), by focusing on post-extracted flow structures (streamlines/surfaces) rather than raw volume data or implicit representations, using a specific pipeline of voxelization, autoencoding, t-SNE, and DBSCAN for interactive selection [S1, unofficial].

## How it works
**Inputs:** A set of flow field datasets containing pre-extracted flow structures (streamlines or stream surfaces).
**Outputs:** Clusters of representative flow structures for filtering and selection.

**Data Structures:**
1.  **Binary Volume:** A 3D voxel grid where cells are 1 if occupied by the flow structure, 0 otherwise.
2.  **Feature Descriptor:** A 1024-dimensional vector produced by the autoencoder for each structure.
3.  **2D Projection:** Coordinates in 2D space derived from t-SNE.

**Steps:**
1.  **Voxelization:** Each flow structure (streamline or stream surface) is voxelized into a binary volume [S1, unofficial].
2.  **Autoencoder Training:** A 3D Convolutional Neural Network (CNN) autoencoder is trained on these binary volumes [S1, unofficial].
    *   The encoder maps the binary volume to a latent space.
    *   The decoder reconstructs the volume from the latent space.
    *   The output of the encoder is a 1024-dimensional feature descriptor [S1, unofficial].
3.  **Dimensionality Reduction:** The 1024-dimensional descriptors are projected into 2D space using t-Distributed Stochastic Neighbor Embedding (t-SNE) [S1, unofficial].
4.  **Clustering:** DBSCAN (Density-Based Spatial Clustering of Applications with Noise) is applied to the 2D projections to identify groups of similar structures [S1, unofficial].
5.  **Selection:** An interactive interface exposes these clusters, allowing users to filter and select representative structures [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def flownet_pipeline(structures):
    # Step 1: Voxelize each structure into a binary volume
    binary_volumes = []
    for structure in structures:
        volume = voxelize(structure) # Returns 3D binary array
        binary_volumes.append(volume)
    
    # Step 2: Train 3D CNN Autoencoder to produce 1024-dim descriptors
    # Note: Training details (loss, architecture) are not specified in sources [S1, unofficial]
    autoencoder = Train3DCNNAutoencoder(binary_volumes)
    
    # Extract descriptors
    descriptors = []
    for volume in binary_volumes:
        descriptor = autoencoder.encoder(volume) # Returns 1024-dim vector
        descriptors.append(descriptor)
    
    # Step 3: Dimensionality reduction via t-SNE
    # Note: t-SNE parameters (perplexity, iterations) are not specified [S1, unofficial]
    projections = t_sne(descriptors, n_components=2)
    
    # Step 4: Clustering via DBSCAN
    # Note: DBSCAN parameters (eps, min_samples) are not specified [S1, unofficial]
    clusters = dbscan(projections)
    
    # Step 5: Return clusters for interactive selection
    return clusters
```

**Edge Cases and Pitfalls:**
*   Reconstruction fidelity varies across flow configurations [S1, unofficial].
*   The method relies on pre-extracted structures, unlike methods that compute features directly from the volume grid [S1, unofficial].

## Subtopics

### SurfNet
A later method by the same authors that replaces the voxel representation with a graph convolutional network (GCN) operating directly on surface meshes, learning node-level and surface-level embeddings with less training time [S1, unofficial]. See [SurfNet](link).

### Transfer Function Design
A related problem where features are computed directly from the volume grid, requiring a different feature space than FlowNet's structure-based approach [S1, unofficial]. See [Transfer Function Design](link).

## Operating characteristics
*   **Performance:** Across eleven flow field datasets, FlowNet achieved training F1 scores of 0.72–0.96 and test F1 scores of 0.57–0.86 [S1, unofficial].
*   **Strengths:** Learned descriptors capture structural similarity well enough for automated grouping [S1, unofficial].
*   **Limitations:** Reconstruction fidelity varies across flow configurations [S1, unofficial]. It requires pre-extracted structures, which can be tedious to generate manually [S1, unofficial].
*   **Comparison:** SurfNet, a successor, offers less training time [S1, unofficial].

## Background
FlowNet was introduced by Han et al. as the first deep learning framework for clustering and selecting representative streamlines or stream surfaces [S1, unofficial]. It addresses the tedium and brittleness of manual feature extraction based on handcrafted geometric features [S1, unofficial]. It was later extended by Han et al. with SurfNet, which uses graph convolutional networks [S1, unofficial].

## Sources
[S1] PDF Deep Learning Methods and Neural Representations for Scientific Volume ..., https://www.cs.uoregon.edu/Reports/AREA-202603-Mathai.pdf, fetched 2026-10-27 (unofficial)
