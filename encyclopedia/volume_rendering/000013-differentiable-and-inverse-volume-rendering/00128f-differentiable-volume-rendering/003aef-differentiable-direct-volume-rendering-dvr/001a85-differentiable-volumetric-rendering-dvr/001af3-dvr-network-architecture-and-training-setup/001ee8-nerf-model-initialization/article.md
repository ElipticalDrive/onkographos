# NeRF Model Initialization

Also called: NeRF MLP Setup, NeRF Network Initialization, NeRF Network Initialization & Query Function

## Overview
NeRF Model Initialization establishes the neural network architecture, positional encoding functions, and sampling parameters required for Differentiable Direct Volume Rendering (DVR). It solves the problem of configuring the multi-layer perceptron (MLP) and its associated query pipeline to map 3D spatial coordinates and view directions to volumetric density and color. It sets apart from siblings like [Direct Voxel Grid Optimization (DVGO)](2184-direct-voxel-grid-optimization-dvgo.md) by relying on a continuous MLP representation queried via hierarchical sampling rather than a discrete voxel grid, requiring explicit setup of positional encoders and coarse/fine network pairs for importance sampling.

## How it works
The initialization process configures the coarse and optional fine MLPs for NeRF, along with a query function that applies positional encoding to 3D coordinates and view directions before passing them to the network [S2, unofficial].

### Inputs and Outputs
*   **Inputs**: Training configuration arguments (hyperparameters), camera poses, and optionally checkpoint files for weight loading.
*   **Outputs**: Initialized `model` (coarse MLP), optional `model_fine` (fine MLP), `embed_fn` (positional encoder for coordinates), `embeddirs_fn` (positional encoder for view directions), `network_query_fn` (wrapper for network execution), and `render_kwargs` dictionaries for training and testing.

### Data Structures
*   **MLP Model**: A multi-layer perceptron.
    *   Input layer size: `input_ch` (for coordinates) + `input_ch_views` (if `use_viewdirs` is true).
    *   Hidden layers: `netdepth` layers of size `netwidth`.
    *   Skip connections: Input coordinates are concatenated to hidden layers at index `skips` (default `[4]`).
    *   Output layer: `output_ch` (default 4: RGB + sigma).
*   **Positional Encoder (`embed_fn`)**: Maps a 3D coordinate vector to a higher-dimensional vector using sinusoidal positional encoding. The dimension is determined by `multires` (log2 of max frequency).
*   **View Direction Encoder (`embeddirs_fn`)**: Maps a 2D view direction vector to a higher-dimensional vector. Dimension determined by `multires_views`.
*   **Render Kwargs**: A dictionary containing:
    *   `perturb`: Boolean/float for jittering samples.
    *   `N_importance`: Integer for number of fine samples.
    *   `N_samples`: Integer for number of coarse samples.
    *   `network_fn`: Reference to the coarse network query function.
    *   `network_fine`: Reference to the fine network query function (if `N_importance > 0`).
    *   `use_viewdirs`: Boolean.
    *   `white_bkgd`: Boolean.
    *   `raw_noise_std`: Float for noise added to sigma.

### Parameters
*   `multires`: Log2 of max frequency for positional encoding (3D location). Default: 10 [S1, unofficial].
*   `netdepth`: Number of layers in the network. Default: 8 [S1, S2, unofficial].
*   `multires_views`: Log2 of max frequency for positional encoding (2D direction). Default: 4 [S1, S2, unofficial].
*   `netwidth`: Number of channels per layer. Default: 256 [S1, S2, unofficial].
*   `N_samples`: Number of coarse samples per ray. Default: 64 [S1, unofficial].
*   `output_ch`: Number of output channels (RGB + sigma). Default: 4 [S1, S2, unofficial].
*   `N_importance`: Number of additional fine samples per ray. Default: 0 [S1, S2, unofficial].
*   `netchunk`: Number of points sent through the network in parallel (batch size for inference). Default: 1024*64 [S1, S2, unofficial].
*   `perturb`: Jitter factor for sampling. Default: 1.0. Set to 0. for no jitter [S1, unofficial].
*   `raw_noise_std`: Standard deviation of noise added to regularize sigma_a output. Default: 0. Recommended: 1e0 [S1, unofficial].
*   `netdepth_fine`: Depth of the fine network. Not given in the sources.
*   `netwidth_fine`: Width of the fine network. Not given in the sources.

### Steps
1.  **Instantiate Positional Encoders**:
    *   Create `embed_fn` for 3D coordinates using `get_embedder(multires, i_embed)` [S1, S2, unofficial].
    *   If `use_viewdirs` is true, create `embeddirs_fn` for 2D view directions using `get_embedder(multires_views, i_embed)` [S1, S2, unofficial].
2.  **Configure Network Dimensions**:
    *   Set `input_ch` based on `embed_fn` output size [S2, unofficial].
    *   Set `input_ch_views` based on `embeddirs_fn` output size (if applicable) [S2, unofficial].
    *   Set `output_ch = 4` [S2, unofficial].
    *   Set `skips = [4]` [S2, unofficial].
3.  **Initialize Coarse MLP**:
    *   Call `init_nerf_model` with parameters `D=netdepth`, `W=netwidth`, `input_ch`, `output_ch`, `skips`, `input_ch_views`, and `use_viewdirs` to create `model` [S1, S2, unofficial].
4.  **Initialize Fine MLP (Optional)**:
    *   If `N_importance > 0`, create `model_fine` using `init_nerf_model` with `D=netdepth_fine` and `W=netwidth_fine` [S1, unofficial].
    *   Add `model_fine` variables to the gradient variables list `grad_vars` [S1, unofficial].
5.  **Define Query Function**:
    *   Define `network_query_fn(inputs, viewdirs, network_fn)` which calls `run_network(inputs, viewdirs, network_fn, embed_fn, embeddirs_fn, netchunk)` [S1, S2, unofficial].
    *   `run_network` reshapes inputs, applies `embed_fn` to coordinates, applies `embeddirs_fn` to view directions (if `use_viewdirs`), concatenates them, and passes them through `network_fn` in chunks of size `netchunk` [S2, unofficial].
6.  **Configure Render Kwargs**:
    *   Create `render_kwargs_train` with `perturb=perturb`, `N_importance`, `N_samples`, `network_fn=network_query_fn`, `network_fine` (if applicable), `use_viewdirs`, `white_bkgd`, and `raw_noise_std` [S1, unofficial].
    *   Create `render_kwargs_test` as a copy of `render_kwargs_train` but with `perturb=False` and `raw_noise_std=0.` [S1, unofficial].
7.  **Load Checkpoints (Optional)**:
    *   If a list of checkpoint files `ckpts` is provided, sort them.
    *   Filter for files containing 'model_' but not 'fine' or 'optimizer' [S1, unofficial].
    *   Load weights into `model` using `model.set_weights(np.load(ft_weights, allow_pickle=True))` [S1, unofficial].
    *   If `model_fine` exists, load corresponding fine weights using the filename pattern `'{}_fine_{}'.format(ft_weights[:-11], ft_weights[-10:])` [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
import tensorflow as tf # Assuming TF as per source context, but logic is framework-agnostic

def get_embedder(multires, i_embed):
    """
    Returns a positional encoding function and the input channel count.
    For i_embed == 0 (full positional encoding):
    L = multires
    F = [sin(2^l * pi * x), cos(2^l * pi * x) for l in range(L)] + [x]
    Input channels = 3 * (2 * L) + 3
    """
    # Implementation details of get_embedder are assumed to be available
    # as per standard NeRF helpers [S1, S2, unofficial].
    pass

def init_nerf_model(D, W, input_ch, output_ch, skips, input_ch_views, use_viewdirs):
    """
    Initializes the MLP model.
    """
    # Implementation details of init_nerf_model are assumed to be available
    # as per standard NeRF helpers [S1, S2, unofficial].
    pass

def run_network(inputs, viewdirs, fn, embed_fn, embeddirs_fn, netchunk=1024*64):
    """
    Prepares inputs and applies network 'fn'.
    """
    inputs_flat = tf.reshape(inputs, [-1, inputs.shape[-1]])
    
    # Apply positional encoding to coordinates
    embedded = embed_fn(inputs_flat)
    
    if use_viewdirs:
        viewdirs_flat = tf.reshape(viewdirs, [-1, viewdirs.shape[-1]])
        embedded_views = embeddirs_fn(viewdirs_flat)
        # Concatenate coordinate encoding and view direction encoding
        embedded = tf.concat([embedded, embedded_views], axis=-1)
    
    # Chunked forward pass
    outputs_flat = []
    for i in range(0, embedded.shape[0], netchunk):
        batch = embedded[i:i+netchunk]
        out = fn(batch)
        outputs_flat.append(out)
    
    outputs = tf.reshape(tf.concat(outputs_flat, axis=0), list(inputs.shape[:-1]) + [outputs_flat[0].shape[-1]])
    return outputs

def network_query_fn(inputs, viewdirs, network_fn):
    """
    Wrapper for run_network.
    """
    return run_network(inputs, viewdirs, network_fn, embed_fn=embed_fn, embeddirs_fn=embeddirs_fn, netchunk=args.netchunk)

def initialize_nerf_model(args):
    """
    Main initialization routine.
    """
    # 1. Instantiate Positional Encoders
    embed_fn, input_ch = get_embedder(args.multires, args.i_embed)
    
    use_viewdirs = args.use_viewdirs
    embeddirs_fn = None
    input_ch_views = 0
    
    if use_viewdirs:
        embeddirs_fn, input_ch_views = get_embedder(args.multires_views, args.i_embed)
        
    output_ch = 4
    skips = [4]
    
    # 2. Initialize Coarse MLP
    model = init_nerf_model(
        D=args.netdepth, 
        W=args.netwidth, 
        input_ch=input_ch, 
        output_ch=output_ch, 
        skips=skips, 
        input_ch_views=input_ch_views, 
        use_viewdirs=use_viewdirs
    )
    
    # 3. Initialize Fine MLP (Optional)
    model_fine = None
    if args.N_importance > 0:
        model_fine = init_nerf_model(
            D=args.netdepth_fine, 
            W=args.netwidth_fine, 
            input_ch=input_ch, 
            output_ch=output_ch, 
            skips=skips, 
            input_ch_views=input_ch_views, 
            use_viewdirs=use_viewdirs
        )
        # Add fine model variables to grad_vars [S1, unofficial]
        # grad_vars.extend(model_fine.trainable_variables) [G]
        
    # 4. Define Query Function
    network_query_fn = lambda inputs, viewdirs: network_query_fn(inputs, viewdirs, model)
    network_query_fn_fine = lambda inputs, viewdirs: network_query_fn(inputs, viewdirs, model_fine) if model_fine else None
    
    # 5. Configure Render Kwargs
    render_kwargs_train = {
        'perturb': args.perturb,
        'N_importance': args.N_importance,
        'N_samples': args.N_samples,
        'network_fn': network_query_fn,
        'network_fine': network_query_fn_fine,
        'use_viewdirs': use_viewdirs,
        'white_bkgd': args.white_bkgd,
        'raw_noise_std': args.raw_noise_std
    }
    
    render_kwargs_test = {
        'perturb': False,
        'N_importance': args.N_importance,
        'N_samples': args.N_samples,
        'network_fn': network_query_fn,
        'network_fine': network_query_fn_fine,
        'use_viewdirs': use_viewdirs,
        'white_bkgd': args.white_bkgd,
        'raw_noise_std': 0.
    }
    
    # 6. Load Checkpoints (Optional)
    if args.ckpts:
        ckpts = sorted([f for f in args.ckpts if 'model_' in f and 'fine' not in f and 'optimizer' not in f])
        for ft_weights in ckpts:
            # Load coarse weights [S1, unofficial]
            # model.set_weights(np.load(ft_weights, allow_pickle=True)) [G]
            
            # Load fine weights if applicable [S1, unofficial]
            if model_fine:
                fine_weights_path = '{}_fine_{}'.format(ft_weights[:-11], ft_weights[-10:])
                # model_fine.set_weights(np.load(fine_weights_path, allow_pickle=True)) [G]
                
    return model, model_fine, render_kwargs_train, render_kwargs_test
```

### Edge Cases and Pitfalls
*   **Precision**: The `netchunk` parameter limits the number of points processed in parallel to manage GPU memory. If `netchunk` is too small, performance degrades due to overhead; if too large, it may cause OOM errors [S1, S2, unofficial].
*   **Boundaries**: The `perturb` parameter controls jittering. Setting it to 0 disables jitter, which may lead to aliasing artifacts if the sampling rate is insufficient [S1, unofficial].
*   **Degenerate Input**: If `N_importance` is 0, `model_fine` and `network_fine` are not created. The query function must handle `None` for the fine network gracefully [S1, unofficial].
*   **View Directions**: If `use_viewdirs` is false, `embeddirs_fn` is not instantiated, and view directions are not passed to the network. This simplifies the model but removes view-dependent appearance [S1, S2, unofficial].

## Subtopics

### Polygonal Mesh Sampling and Octahedral Initialization
This variant addresses slow and unstable convergence on tilted surfaces by replacing standard cube-based sampling with polygonal meshes. It initializes a 128×128×128 grid, selecting the center point of each grid along with its six neighboring grids (front, back, left, right, top, and bottom) to create polygonal meshes. To further improve convergence, it utilizes a regular octahedron with 20 faces (8 exterior, 12 interior) instead of standard cubes. The ray from camera origin to pixel is defined as $r(t) = o + td$, where $d = (\theta, \phi)$ is the viewing direction. The intersection of this ray with the polygonal meshes serves as the sampling point. Unlike standard NeRF, this method predicts opacity $\alpha$ directly instead of volume density, using the rendering equation $\hat{C}(r) = \sum_{i=1}^{N} T_i \alpha_i c_i$, where $T_i = \prod_{j=0}^{i-1} (1 - \alpha_j)$ [S3].

### NeRF-DS
NeRF-DS extends NeRF to capture dynamic, reflective objects. It employs a dual-network design similar to D-NeRF: a deformation field maps time-varying coordinates into a canonical space. It uses a specularity-aware radiance field that jointly predicts view-dependent appearance and reflection behavior, conditioning radiance on surface orientation and spatial location. An object mask is used to maintain temporal consistency under changing reflections [S4].

## Operating characteristics
*   **Time Complexity**: Initialization is $O(1)$ relative to scene size, depending only on network architecture parameters. Inference complexity is dominated by the number of samples per ray ($N_{samples} + N_{importance}$) and the number of rays.
*   **Memory Complexity**: Memory usage is proportional to the number of parameters in the MLPs ($D \times W \times (input\_ch + input\_ch\_views)$) and the storage for checkpoints. It does not scale with scene resolution or number of points, unlike voxel-based methods.
*   **Measured Performance**: Not given in the sources.
*   **Scaling**: Scales well with scene complexity as it is resolution-independent. Performance is limited by the MLP query speed and the number of samples required for convergence.
*   **Strengths**: Resolution-independent, compact representation, supports view-dependent effects.
*   **Limitations**: Slow inference speed compared to grid-based methods, requires many samples for high-quality results, sensitive to initialization and hyperparameters.
*   **When to use**: High-quality novel view synthesis where resolution independence is desired.
*   **When not to use**: Real-time applications requiring high frame rates, or scenes with extreme geometric complexity where sampling becomes inefficient.

## Background
The NeRF Model Initialization method originates from the original NeRF paper (Mildenhall et al., 2020) [S1, S2, unofficial]. It was designed to enable differentiable volume rendering using a continuous volumetric scene function represented by an MLP. The method has been widely adopted as the baseline for subsequent NeRF variants. Variants like NeRF-DS [S4] and polygonal mesh sampling [S3] have extended the initialization and sampling strategies to handle dynamic scenes and improve convergence on specific geometries.

## Sources
[S1] bmild/nerf, https://github.com/bmild/nerf, fetched 2026-10-26 (unofficial)
[S2] YoujiaZhang/NeMF, https://github.com/YoujiaZhang/NeMF, fetched 2026-10-27 (unofficial)
[S3] UE4-NeRF: Neural Radiance Field for Real-Time Rendering of Large-Scale Scene, https://doi.org/10.48550/arXiv.2310.13263, fetched 2026-10-29
[S4] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
