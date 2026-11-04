# Neural Dual Contouring

Also called: NDC, SDC, UNDC, Self-Supervised Dual Contouring, Unoriented Neural Dual Contouring

## Overview
Neural Dual Contouring (NDC) is a data-driven mesh reconstruction method that replaces the hand-crafted Quadratic Error Function (QEF) minimization of classical Dual Contouring with a neural network that directly predicts vertex positions and edge crossings [S3]. It solves the problem of reconstructing high-fidelity meshes from implicit inputs (SDFs, point clouds, or voxels) without requiring explicit surface gradients or suffering from the ill-defined QEF scenarios that plague classical methods [S1, S3]. Unlike its sibling [Dual Contouring](778-dual-contouring.md), which relies on precise normals and QEF optimization, NDC produces meshes with better tessellation quality and can generate open surfaces with boundaries directly, avoiding the watertight constraint of methods like Marching Cubes [S3].

## How it works
**Inputs and Outputs:**
*   **Input:** A tensor representing the implicit field (e.g., SDF values $\Phi$ sampled at grid vertices $X$, binary voxel occupancy, or point cloud data) [S3].
*   **Output:** A mesh $\mathcal{M}$ consisting of vertices $\mathcal{V}$ and faces $\mathcal{F}$ [S3].

**Data Structures:**
*   **Grid $G$:** A regular 3D lattice of nodes $g_i$. For SDF inputs, the input is a floating-point tensor of shape $|X|$ [S3].
*   **Network Output:** The network predicts vertex positions $\mathcal{V}$ and sign/crossing flags $\mathcal{S}$ [S3].
*   **Faces $\mathcal{F}$:** Constructed based on the XOR of predicted signs on connected vertices, identical to classical Dual Contouring connectivity logic [S3].

**Algorithm Steps:**

1.  **Input Processing:**
    *   For SDF inputs, sample the function $\Phi$ at grid vertices $X$ into a tensor [S3].
    *   For Point Cloud inputs, process via a local point cloud encoder (PointNet++) and a regular grid encoder [S3].
    *   For Voxelized Occupancy inputs, use a similar architecture to SDF but with deeper convolutions [S3].

2.  **Sign Prediction (Connectivity):**
    *   Predict grid signs $\mathcal{S}$ using a neural network $f_{\mathcal{S}}$ taking input $I$ and lattice $G$ [S3].
    *   For SDF inputs, the sign $S$ at lattice locations is computed similarly to classical Dual Contouring, which is efficient [S3].
    *   Faces $\mathcal{F}$ are generated using the XOR of predicted signs on connected vertices: if $\text{sign}(f(g_i)) \neq \text{sign}(f(g_j))$, a quadrilateral face is constructed in the cell incident on edge $e_{ij}$ [S1, S3].

3.  **Vertex Prediction:**
    *   Predict vertex positions $\mathcal{V}$ using a neural network $f_{\mathcal{V}}$ taking input $I$ and lattice $G$ [S3].
    *   The network produces exactly one vertex per grid cell [S3].
    *   **Masking:** Apply a mask to select only grid cells that bear node SDF values of opposite signs for vertex prediction [S1].

4.  **Mesh Construction:**
    *   Construct quadrilateral faces in cells incident on every edge $e_{ij}$ connecting grid nodes of different signs [S1].
    *   Triangulate the resulting quadrilaterals by joining the top-left and bottom-right vertices as a convention [S1].

**Neural Network Architecture:**
*   **SDF Encoder ($V$):** A 3D CNN with 6 layers. The first 3 layers have kernel size $3\times3\times3$, and the last 3 layers have kernel size $1\times1\times3$, resulting in an overall receptive field of $7\times7\times7$ [S3].
    *   Hidden layers use 64 channels [S3].
    *   Leaky ReLU activation in all hidden layers; sigmoids at the output layer [S3].
*   **Voxel Input Variant:** Uses 7 layers of $3\times3\times3$ convolutions to enlarge the receptive field to $15\times15\times15$ [S3].
*   **Point Cloud Variant:** Uses a local point cloud encoder (PointNet++) and three $3\times3\times3$ and three $1\times1\times3$ convolution layers for grid processing [S3].
*   **UNDC Variant:** Uses the same architecture for $F$ as $V$ in the NDC model [S3].

**Training and Loss Functions:**
NDC is typically trained in a supervised manner using pseudo-ground-truth vertices computed by classical Dual Contouring [S3].

*   **Vertex Loss ($L_V$):** L2 reconstruction loss against pseudo ground-truth vertices $v_{gt}$ [S3].
    $$L_V(\theta) = \mathbb{E}_{(I, M_V, v_{gt}) \sim D} \sum_{m,n,k} \left[ M_V \odot |f_V(I, G; \theta) - v_{gt}|^2 \right]$$
    where $\odot$ is the Hadamard product on $G$, and $M_V$ is a binary mask [S3].

*   **Sign Loss ($L_S$):** Binary Cross Entropy (BCE) for sign/crossing predictions [S3].
    $$L_S(\theta) = \mathbb{E}_{(I, M_S, S_{gt}) \sim D} \sum_{m,n,k} \left[ M_S \odot \text{BCE}(f_S(I, G; \theta, S_{gt})) \right]$$
    where $M_S$ is a binary mask [S3].

*   **Data Augmentation:**
    *   Rotation by $\pi/2$ around Euclidean axes, mirroring, and global sign inversion [S3].
    *   Do not perform augmentation within a mini-batch; rely on SGD to aggregate data towards a zero-mean residual configuration [S3].
    *   Supervise predictions only in a narrowband around the input surface using binary masks $M_S$ and $M_V$ [S3].

**Pseudocode for Inference:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def neural_dual_contouring_inference(input_tensor, grid, network_V, network_S):
    """
    Reconstructs a mesh from an implicit input using Neural Dual Contouring.
    
    Args:
        input_tensor: Tensor of SDF values, voxel occupancy, or point cloud features.
        grid: Regular 3D grid structure defining node positions.
        network_V: Neural network predicting vertex positions.
        network_S: Neural network predicting signs/crossings.
        
    Returns:
        vertices: List of 3D coordinates.
        faces: List of vertex indices forming triangles.
    """
    # 1. Predict Signs for Connectivity
    # Predicts binary signs for each grid node or edge crossing
    predicted_signs = network_S(input_tensor, grid)
    
    # 2. Predict Vertex Positions
    # Outputs one vertex per grid cell
    predicted_vertices = network_V(input_tensor, grid)
    
    # 3. Construct Faces based on Sign Changes
    faces = []
    vertices = []
    
    # Iterate over all grid cells
    for cell in grid.cells:
        # Check edges of the cell for sign changes
        intersected_edges = []
        for edge in cell.edges:
            sign_i = predicted_signs[edge.node_i]
            sign_j = predicted_signs[edge.node_j]
            
            # If signs are opposite, the edge intersects the isosurface
            if sign_i != sign_j:
                intersected_edges.append(edge)
        
        # If the cell has intersected edges, construct a quad face
        if len(intersected_edges) > 0:
            # Get the predicted vertex for this cell
            cell_vertex = predicted_vertices[cell.id]
            
            # Add vertex to list
            vertices.append(cell_vertex)
            
            # Construct the quad face using the intersected edges
            # The specific topology depends on the number of intersected edges
            # Classical DC logic applies here, mapped to the predicted vertex
            face_indices = construct_quad_face(intersected_edges, len(vertices) - 1)
            faces.append(face_indices)
            
    # 4. Triangulate Quads
    # Convention: Join top-left and bottom-right vertices
    triangulated_faces = []
    for face in faces:
        # face is a list of 4 vertex indices
        triangulated_faces.append([face[0], face[1], face[2]])
        triangulated_faces.append([face[0], face[2], face[3]])
        
    return vertices, triangulated_faces
```

**Edge Cases and Pitfalls:**
*   **QEF Ill-Definedness:** Classical DC QEF minimization is ill-defined in certain scenarios and susceptible to SDF irregularities, leading to poor triangulation. NDC avoids this by predicting vertices directly [S1, S2].
*   **Self-Intersections:** NDC avoids self-intersections along sharp edges because it does not rely on QEF minimization [S2].
*   **No Gradients Required:** Unlike classical DC, NDC does not require surface gradients as input [S3].

## Subtopics

### [SDC Training & Regularization Setup](4563-sdc-training-regularization-setup.md)
Self-Supervised Dual Contouring (SDC) defines a self-supervised training pipeline for Neural Dual Contouring using Curriculum DeepSDF as the Deep Implicit Network (DIN). It solves the problem of reconstructing high-fidelity meshes from single or multiple images without ground-truth 3D labels by jointly training an encoder and an implicit field. It is distinguished from standard DIN training by the late-stage application of a mesh-based regularization term that aligns the predicted Signed Distance Field (SDF) with the geometry extracted via SDC, rather than relying solely on external SDF supervision.

### Mask Definitions (MS, MV, MF)
Binary masks ($M_S, M_V, M_F$) flag grid cells or edges for specific contouring operations based on input type (SDF/UDF grid, binary voxel, or point cloud) [S3].
*   **MV - NDC:** For a grid cell, if corner vertices have different signs in the ground truth SDF, set corresponding $M_V$ entry to 1; others 0 [S3].
*   **MV - UNDC:** For a grid cell, if any edge intersects the ground truth shape, set corresponding $M_V$ entry to 1; others 0 [S3].
*   **MS - NDC (SDF grid):** NDC directly uses input signs as $S$, so $M_S$ is not needed [S3].
*   **MF - UNDC (SDF/UDF):** For an edge in a grid cell, if both end vertices have signed distances < 1, set corresponding $M_F$ entry to 1; others 0 [S3].
*   **MS - NDC (Binary voxel):** For an occupied grid cell adjacent to an unoccupied cell in its 3x3 neighborhood, set corresponding entries for all 8 vertices to 1; others 0 [S3].
*   **MF - UNDC (Binary voxel):** For an edge in a grid cell, if all four adjacent cells are occupied, set corresponding $M_F$ entry to 1; others 0 [S3].
*   **MF - UNDC (Point cloud):** Set $M_F$ entries to 1 for edges stored in cells within 3 units (Manhattan distance) to the closest point cloud point; others 0 [S3].
*   **Parameters:** Grid cell size = 1 unit; Distance threshold for $M_F$ (UNDC, SDF/UDF) = 1 [S3].

## Operating characteristics
*   **Time Complexity:** Inference is a single forward pass through the CNN, avoiding test-time optimization [S2].
*   **Memory Complexity:** The SDF encoder uses 64 channels, resulting in low weight count (1MB) and computational efficiency (37 fps) [S3].
*   **Measured Performance:**
    *   On ABC dataset with $64^3$ SDF input, UNDC achieves CD $0.930 \times 10^{-5}$, F1 0.873, NC 0.974, ECD $0.328 \times 10^{-2}$, EF1 0.746, #V 5,584, #T 11,295, Inference time 0.051s [S3].
    *   On ABC dataset with $128^3$ SDF input, UNDC achieves CD $0.789 \times 10^{-5}$, F1 0.890, NC 0.983, ECD $0.149 \times 10^{-2}$, EF1 0.813, #V 22,578, #T 45,411, Inference time 0.410s [S3].
    *   SDC pre-processing time is $2.7 \times 10^{-3}$ seconds per shape for a $64^3$ grid, compared to 9.0 seconds for NDC and UNDC [S1].
*   **Scaling:** Performance scales with grid resolution; higher resolutions ($128^3$) yield better metrics but increase inference time and vertex count [S3].
*   **Strengths:** Produces meshes with better tessellation quality; can produce open surfaces with boundaries; does not require surface gradients; avoids QEF minimization ill-defined scenarios [S3].
*   **Limitations:** Requires training data (pseudo-ground-truth from classical DC); receptive fields are limited to ensure generalizability [S3].
*   **When to use:** When high-fidelity mesh reconstruction from implicit inputs is needed without manual gradient computation or QEF tuning.
*   **When not to use:** When real-time performance is critical and training overhead is prohibitive, or when exact watertightness is required without post-processing (though NDC can produce watertight meshes, it is not guaranteed by design like MC) [S3].

## Background
Neural Dual Contouring (NDC) was introduced as a data-driven mesh reconstruction method based on dual contouring that uses a neural network to predict vertex locations and edge crossings, eliminating the need for hand-crafted functions and surface gradients [S3]. It was trained on the ABC CAD dataset [S3] and tested on ABC, Thingi10K, FAUST, MGN, and Matterport3D datasets [S3]. The method was developed to address the limitations of classical Dual Contouring, which relies on Quadratic Error Function (QEF) minimization requiring precise SDF values and normals [S1]. Self-Supervised Dual Contouring (SDC) was later proposed as a variant that uses self-supervised losses relying only on the input SDF, avoiding dependence on vertex positions obtained via QEF minimization [S1].

## Sources
[S1] Self-Supervised Dual Contouring, https://doi.org/10.1109/cvpr52733.2024.00448, fetched 2026-10-26
[S2] Self-Supervised Dual Contouring, https://inria.hal.science/hal-04590906v1, fetched 2026-10-26
[S3] Neural Dual Contouring, https://arxiv.org/abs/2202.01999v3, fetched 2026-10-26
