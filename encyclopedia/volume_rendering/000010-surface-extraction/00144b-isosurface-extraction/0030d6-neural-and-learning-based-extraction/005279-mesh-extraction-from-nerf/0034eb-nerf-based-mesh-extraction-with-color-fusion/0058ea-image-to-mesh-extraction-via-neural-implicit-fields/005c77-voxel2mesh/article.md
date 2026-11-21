# Voxel2Mesh

Also called: 3D Mesh Model Generation from Volumetric Data

## Overview
Voxel2Mesh is a CNN-based architecture that generates 3D surface meshes directly from 3D image volumes without post-processing, avoiding artifacts and enabling end-to-end training [S1, unofficial]. It sets itself apart from standard voxel rendering or segmentation methods by outputting explicit surface representations (vertices and faces) rather than labeled voxel grids, allowing for differentiable mesh operations and better accuracy in biomedical segmentation tasks [S1, unofficial].

## How it works
**Inputs and Outputs:**
The input is a volumetric tensor $x$ of shape $(B, C_{in}, D, H, W)$ for 3D data or $(B, C_{in}, H, W)$ for 2D data, where $B$ is batch size, $C_{in}$ is input channels, and $D, H, W$ are spatial dimensions [S1, unofficial]. The output is a list of meshes, one per semantic class (excluding background). Each mesh consists of vertices $V$ (shape $[B, 3, N]$), faces $F$ (shape $[B, 3, K]$), and latent features $L$ (shape $[B, C_{feat}, N]$) [S1, unofficial].

**Data Structures:**
*   **Encoder:** A U-Net encoder with $S+1$ layers (where $S$ is `config.steps`).
*   **Decoder:** Iterative refinement stages.
*   **Mesh State:** At each stage $i$, the state includes current vertices, faces, and latent features.
*   **Skip Connections:** Feature maps from the encoder are upsampled and concatenated with decoder features.

**Parameters:**
*   `config.ndims`: 2 or 3, determining convolution dimensionality [S1, unofficial].
*   `config.first_layer_channels`: Base channel count for the U-Net [S1, unofficial].
*   `config.steps`: Number of downsampling/upsampling steps [S1, unofficial].
*   `config.num_classes`: Number of semantic classes. The loop runs for $k \in \{1, \dots, \text{num\_classes}-1\}$ [S1, unofficial].
*   `unpool`: List specifying mesh subdivisions per stage. Default: `[0, 1, 1, 1, 1]` [S1, unofficial].

**Algorithm Steps:**

1.  **Initialization:**
    Load a pre-loaded icosahedron sphere with 162 vertices. Normalize vertices to unit length:
    $$ V_{init} = \frac{V_{sphere}}{\sqrt{\sum_{j} V_{sphere,j}^2}} $$
    Initialize prediction list $pred[k]$ for each class $k$ with $V_{init}$, initial faces (icosahedron topology), and reference sphere vertices [S1, unofficial].

2.  **Encoding:**
    Encode the input volume $x$ using the U-Net encoder.
    *   Layer 0: Input channels $C_{in}$, Output channels $C_{first} = \text{config.first\_layer\_channels}$.
    *   Layer $i$ ($1 \le i \le S$): Input channels $C_{first} \cdot 2^{i-1}$, Output channels $C_{first} \cdot 2^i$.
    *   Apply max pooling before each layer except the first.
    *   Store encoder outputs as $down\_outputs$ [S1, unofficial].

3.  **Iterative Decoding and Refinement:**
    For each class $k$ from $1$ to $\text{num\_classes}-1$:
    For each decoder stage $i$ from $0$ to $S$:
    
    a. **Feature Retrieval:**
       Retrieve current vertices $V$, faces $F$, and latent features $L$ from $pred[k][i]$.
       Retrieve skip connection feature map $x_{skip}$ from $down\_outputs$ in reverse order ($down\_outputs[::-1]$).
       
    b. **Skip Connection Processing:**
       If $i > 0$:
       1. Upsample $x_{skip}$ using `grid_upconv_layer` (ConvTranspose).
       2. Crop and merge with the corresponding encoder output $down\_output$.
       3. Pass through `grid_unet_layer`.
       If $i == 0$: $x_{skip}$ is simply the encoder output.
       
    c. **Mesh Unpooling (Subdivision):**
       If `do_unpool[0] == 1` (or based on `unpool` list configuration), apply `uniform_unpool` to $V$, $F$, and $L$ to increase resolution by subdividing faces and averaging vertex positions/features [S1, unofficial].
       
    d. **Graph Construction:**
       Compute adjacency matrix $A$ and degree matrix $D$ for the current mesh topology $(V, F)$ using `adjacency_matrix(V, F)` [S1, unofficial].
       
    e. **Feature Aggregation:**
       1. Compute skip features: Pass cropped $x_{skip}$ (first `skip_amount` channels) and current vertices $V$ through `skip_connection` module:
          $$ F_{skip} = \text{skip\_connection}(x_{skip}[:, :skip\_amount], V) $$
       2. Concatenate features along dimension 2:
          $$ F_{input} = \begin{cases} \text{cat}(L, F_{skip}, V) & \text{if } L \text{ is not None} \\ \text{cat}(F_{skip}, V) & \text{otherwise} \end{cases} $$
          
    f. **Graph Convolution:**
       Process $F_{input}$ through `graph_unet_layer` using $A$, $D$, $V$, and $F$ to update latent features $L_{new}$ [S1, unofficial].
       
    g. **Vertex Displacement:**
       Predict vertex displacements $\Delta V$ using `feature2vertex` layer on $L_{new}$ and mesh topology [S1, unofficial].
       
    h. **Vertex Update:**
       Update vertices:
       $$ V_{new} = V + \Delta V $$
       
    i. **Storage:**
       Store $V_{new}$, $F$, $L_{new}$, and reference $V_{sphere}$ in $pred[k][i+1]$ for the next iteration [S1, unofficial].

4.  **Inference:**
    Extract final predicted vertices and faces for each class. Rasterize the mesh to voxels if volume comparison is needed [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

# Assume helper functions are defined:
# - grid_upconv_layer, grid_unet_layer
# - uniform_unpool
# - adjacency_matrix
# - skip_connection
# - graph_unet_layer
# - feature2vertex

def voxel2mesh_forward(x, config, down_outputs):
    """
    x: Input volume tensor [B, C_in, D, H, W] or [B, C_in, H, W]
    config: Configuration object with ndims, first_layer_channels, steps, num_classes
    down_outputs: List of encoder feature maps from U-Net
    """
    # 1. Initialize Sphere
    # Load icosahedron vertices (162 vertices)
    sphere_vertices = load_icosahedron_vertices() # Shape [162, 3]
    # Normalize to unit length
    sphere_vertices = sphere_vertices / torch.sqrt(torch.sum(sphere_vertices**2, dim=1)[:, None])
    sphere_vertices = sphere_vertices[None] # Add batch dim [1, 162, 3]
    
    # Initialize faces (icosahedron topology)
    sphere_faces = load_icosahedron_faces() # Shape [162, 3] (indices)
    sphere_faces = sphere_faces[None] # Add batch dim [1, 162, 3]
    
    # Reference vertices for loss/normalization
    ref_vertices = sphere_vertices.clone()
    
    # Initialize predictions for each class (excluding background 0)
    # pred[k] will store states for each stage i
    num_classes = config.num_classes
    pred = {}
    for k in range(1, num_classes):
        pred[k] = []
        # Initial state: vertices, faces, latent_features (None initially), ref_vertices
        # Latent features shape: [B, C_feat, N]
        # For simplicity, assume initial latent features are zeros or learned embeddings
        B = x.shape[0]
        N_init = sphere_vertices.shape[1]
        C_feat = config.first_layer_channels # Or specific latent dim
        latent_init = torch.zeros(B, C_feat, N_init, device=x.device)
        
        pred[k].append({
            'vertices': sphere_vertices.clone(),
            'faces': sphere_faces.clone(),
            'latent_features': latent_init,
            'ref_vertices': ref_vertices.clone()
        })

    # 2. Decoder Loop
    for i in range(config.steps + 1): # Stages 0 to steps
        # Get skip connection
        if i == 0:
            x_skip = down_outputs[-1] # Last encoder output
        else:
            # Retrieve from reversed down_outputs
            # down_outputs[::-1] means index 0 is last layer
            x_skip = down_outputs[::-1][i-1]
            
            # Upsample and merge
            x_skip = grid_upconv_layer(x_skip)
            # Crop and merge with corresponding encoder output
            # Assuming down_outputs order matches encoder depth
            # Encoder outputs: [L0, L1, ..., LS]
            # Decoder stage i (i>0) corresponds to encoder output at index S-i
            # Note: Logic depends on exact indexing of down_outputs provided by encoder
            # Based on notes: "corresponding encoder output down_output"
            # If down_outputs is [L0...LS], and we iterate i=0..S
            # i=0 uses LS. i=1 uses LS-1.
            # So index is S - i
            idx_enc = config.steps - i
            x_enc = down_outputs[idx_enc]
            x_skip = crop_and_merge(x_skip, x_enc)
            x_skip = grid_unet_layer(x_skip)

        # Process each class
        for k in range(1, num_classes):
            state = pred[k][-1]
            V = state['vertices']
            F = state['faces']
            L = state['latent_features']
            
            # Unpooling
            # Check if unpooling is enabled for this stage
            # Note: 'unpool' list typically has length equal to number of stages
            # Assuming unpool[i] controls stage i
            if config.unpool[i] > 0:
                V, F, L = uniform_unpool(V, F, L)
            
            # Graph Construction
            A, D = adjacency_matrix(V, F)
            
            # Skip Features
            # Crop x_skip to skip_amount channels
            skip_amount = config.first_layer_channels # Or specific param
            x_skip_cropped = x_skip[:, :skip_amount]
            F_skip = skip_connection(x_skip_cropped, V)
            
            # Concatenate
            if L is not None and L.shape[1] > 0:
                F_input = torch.cat([L, F_skip, V], dim=2)
            else:
                F_input = torch.cat([F_skip, V], dim=2)
                
            # Graph Convolution
            L_new = graph_unet_layer(F_input, A, D, V, F)
            
            # Vertex Displacement
            delta_V = feature2vertex(L_new, V, F)
            
            # Update Vertices
            V_new = V + delta_V
            
            # Store for next stage
            pred[k].append({
                'vertices': V_new,
                'faces': F, # Faces topology usually fixed or updated by unpool
                'latent_features': L_new,
                'ref_vertices': state['ref_vertices']
            })

    # 3. Output
    # Extract final vertices and faces for each class
    meshes = []
    for k in range(1, num_classes):
        final_state = pred[k][-1]
        meshes.append({
            'vertices': final_state['vertices'],
            'faces': final_state['faces'],
            'class_id': k
        })
    return meshes
```

## Operating characteristics
*   **Time Complexity:** Depends on the number of classes, decoder steps, and mesh resolution (number of vertices $N$). Graph convolutions are typically $O(E)$ where $E$ is edges, roughly proportional to $N$.
*   **Memory Complexity:** High. GPU memory exhaustion is a known issue during evaluation due to high vertex counts, especially with default unpooling settings [S1, unofficial].
*   **Performance:** Evaluated on Electron Microscopy, MRI brain images, and CT liver scans. Outperforms state-of-the-art segmentation methods (CNN baseline + post-processing) in accuracy [S1, unofficial].
*   **Strengths:** End-to-end differentiable training, no post-processing artifacts, explicit surface representation.
*   **Limitations:** High GPU memory usage; sensitive to vertex winding order (faces may be clockwise or anti-clockwise inconsistently); potential slice order issues in input stacks [S1, unofficial].
*   **When to use:** When explicit 3D meshes are required from volumetric data and end-to-end training is desired.
*   **When not to use:** When GPU memory is limited (reduce unpooling values) or when consistent face orientation is critical without post-processing correction [S1, unofficial].

## Background
Voxel2Mesh was introduced by Udaranga Wickramasinghe, Edoardo Remelli, Graham Knott, and Pascal Fua in the MICCAI 2020 proceedings [S1, unofficial]. It addresses the limitation of CNN-based volumetric methods that label individual voxels, which require post-processing to obtain 3D surface representations, introducing artifacts and preventing end-to-end training [S1, unofficial].

## Sources
[S1] cvlab-epfl/voxel2mesh, https://github.com/cvlab-epfl/voxel2mesh, fetched 2026-10-27 (unofficial)
