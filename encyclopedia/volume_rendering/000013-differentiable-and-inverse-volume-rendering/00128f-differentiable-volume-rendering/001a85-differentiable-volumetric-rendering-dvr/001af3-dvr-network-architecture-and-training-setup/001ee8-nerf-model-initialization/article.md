# NeRF Model Initialization

Also called: NeRF MLP Setup, NeRF Network Initialization

## Overview
NeRF Model Initialization configures the multi-layer perceptrons (MLPs) and positional encoders required for differentiable volumetric rendering of a scene. It establishes the coordinate encoding scheme, the network topology for density and color prediction, and the sampling parameters for coarse and fine ray marching. It differs from sibling methods like Direct Voxel Grid Optimization (DVGO) by relying on implicit neural representations (MLPs) rather than explicit voxel grids for scene representation [S1, unofficial].

## How it works
**Inputs:**
- Scene data (poses, images) loaded via `_load_data` [S1, unofficial].
- Hyperparameters: `netdepth`, `netwidth`, `multires`, `multires_views`, `N_samples`, `N_importance`, `perturb`, `raw_noise_std`, `use_viewdirs` [S1, unofficial].

**Outputs:**
- `model`: Coarse MLP instance.
- `model_fine`: Fine MLP instance (optional).
- `render_kwargs_train`: Dictionary of rendering parameters for training.
- `render_kwargs_test`: Dictionary of rendering parameters for evaluation.

**Data Structures:**
- **Positional Encoder (`embed_fn`)**: Maps 3D coordinates $\mathbf{x} \in \mathbb{R}^3$ to a higher-dimensional space using sinusoidal embeddings up to frequency $2^{\text{multires}-1}$ [S1, unofficial].
- **Directional Encoder (`embeddirs_fn`)**: Maps 2D view directions $\mathbf{d} \in \mathbb{R}^2$ (spherical coordinates) to a higher-dimensional space using sinusoidal embeddings up to frequency $2^{\text{multires\_views}-1}$ [S1, unofficial].
- **Coarse MLP (`model`)**: A fully connected network with `netdepth` layers, `netwidth` hidden units, and skip connections from input to layers $4, 8, \dots$ [S1, unofficial]. Input dimension is $3 \times (2 \times \text{multires} + 1)$ for coordinates. If `use_viewdirs` is true, view directions are encoded and injected into hidden layers. Output dimension is 4 (RGB + volume density $\sigma$) [S1, unofficial].
- **Fine MLP (`model_fine`)**: Same architecture as `model` but with potentially different `netdepth_fine` and `netwidth_fine` [S1, unofficial].

**Steps:**
1. **Instantiate Positional Encoders**:
   - Create `embed_fn` for 3D coordinates using `get_embedder(multires, i_embed)` [S1, unofficial].
   - If `use_viewdirs` is true, create `embeddirs_fn` for 2D view directions using `get_embedder(multires_views, i_embed)` [S1, unofficial].

2. **Create Coarse MLP**:
   - Initialize `model` using `init_nerf_model` with:
     - Depth: `netdepth` (default 8) [S1, unofficial].
     - Width: `netwidth` (default 256) [S1, unofficial].
     - Input channels: Determined by `embed_fn` output size.
     - Output channels: 4 (RGB + $\sigma$) [S1, unofficial].
     - Skip connections: Added at layer indices $4, 8, \dots$ (every 4 layers) [S1, unofficial].
     - View direction input: If `use_viewdirs`, encode view directions and concatenate to hidden layer representations before the final output layer [S1, unofficial].

3. **Create Fine MLP (Optional)**:
   - If `N_importance > 0`, initialize `model_fine` with `netdepth_fine` and `netwidth_fine` [S1, unofficial].
   - Add `model_fine` parameters to the gradient variables list `grad_vars` [S1, unofficial].

4. **Define Network Query Function**:
   - Create `network_query_fn` that wraps `run_network`. This function takes ray origins, directions, and the encoders, applies the positional encoding, passes data through the MLP, and returns raw RGB and $\sigma$ values [S1, unofficial].

5. **Configure Training Render Kwargs**:
   - Create `render_kwargs_train` dictionary with:
     - `perturb`: `args.perturb` (default 1.0, jitter for sampling) [S1, unofficial].
     - `N_importance`: `args.N_importance` (default 0) [S1, unofficial].
     - `N_samples`: `args.N_samples` (default 64) [S1, unofficial].
     - `network_fn`: `network_query_fn` [S1, unofficial].
     - `network_fine`: `network_query_fn` for `model_fine` if `N_importance > 0` [S1, unofficial].
     - `use_viewdirs`: `args.use_viewdirs` [S1, unofficial].
     - `white_bkgd`: Boolean flag for background [S1, unofficial].
     - `raw_noise_std`: `args.raw_noise_std` (default 1e0, noise added to $\sigma$ for regularization) [S1, unofficial].

6. **Configure Test Render Kwargs**:
   - Create `render_kwargs_test` as a copy of `render_kwargs_train` with:
     - `perturb`: `False` [S1, unofficial].
     - `raw_noise_std`: `0.` [S1, unofficial].

7. **Load Checkpoints (Optional)**:
   - If a checkpoint directory `ckpts` exists, sort files containing 'model_' but not 'fine' or 'optimizer' [S1, unofficial].
   - Load weights into `model` using `model.set_weights(np.load(ft_weights, allow_pickle=True))` [S1, unofficial].
   - If `model_fine` exists, load corresponding weights using the filename pattern `'{}_fine_{}'.format(ft_weights[:-11], ft_weights[-10:])` [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
import torch
import torch.nn as nn

def get_embedder(multires, i_embed=0):
    """
    [G] Standard positional encoding implementation.
    Returns a function that maps 3D/2D coordinates to embedded space.
    """
    if i_embed == 0:
        return nn.Identity(), 3
    embed_kwargs = {
        'include_input': True,
        'input_dims': 3,
        'max_freq_log2': multires - 1,
        'num_freqs': multires,
        'log_sampling': True,
        'periodic_fns': [torch.sin, torch.cos],
    }
    embedder_obj = Embedder(**embed_kwargs)
    return embedder_obj, embedder_obj.out_dim

class Embedder(nn.Module):
    def __init__(self, **kwargs):
        super().__init__()
        self.kwargs = kwargs
        self.create_embedding_fn()

    def create_embedding_fn(self):
        embed_fns = []
        d = self.kwargs['input_dims']
        out_dim = 0
        if self.kwargs['include_input']:
            embed_fns.append(lambda x: x)
            out_dim += d
        max_freq = self.kwargs['max_freq_log2']
        N_freqs = self.kwargs['num_freqs']
        if self.kwargs['log_sampling']:
            freq_bands = 2. ** torch.linspace(0., max_freq, steps=N_freqs)
        else:
            freq_bands = torch.linspace(2.**0., 2.**max_freq, steps=N_freqs)
        for freq in freq_bands:
            for p_fn in self.kwargs['periodic_fns']:
                embed_fns.append(lambda x, p_fn=p_fn, freq=freq: p_fn(x * freq))
                out_dim += d
        self.embed_fns = embed_fns
        self.out_dim = out_dim

    def forward(self, inputs):
        return torch.cat([fn(inputs) for fn in self.embed_fns], -1)

def init_nerf_model(netdepth, netwidth, input_ch, output_ch, use_viewdirs, embed_fn, embeddirs_fn=None):
    """
    [G] Standard MLP initialization with skip connections.
    """
    class NeRF(nn.Module):
        def __init__(self):
            super().__init__()
            self.layers = nn.ModuleList()
            # Input layer
            self.layers.append(nn.Linear(input_ch, netwidth))
            # Hidden layers
            for i in range(netdepth - 1):
                if i % 4 == 0 and i > 0:
                    # Skip connection: concat input to hidden layer
                    self.layers.append(nn.Linear(input_ch + netwidth, netwidth))
                else:
                    self.layers.append(nn.Linear(netwidth, netwidth))
            
            self.use_viewdirs = use_viewdirs
            self.input_ch = input_ch
            self.netwidth = netwidth
            self.output_ch = output_ch
            
            if use_viewdirs:
                self.dir_layer = nn.Linear(netwidth, netwidth)
                self.output_layer = nn.Linear(netwidth + embeddirs_fn.out_dim, output_ch)
            else:
                self.output_layer = nn.Linear(netwidth, output_ch)

        def forward(self, x):
            h = x
            for i, layer in enumerate(self.layers):
                h = layer(h)
                if i > 0:
                    h = torch.relu(h)
                if i % 4 == 0 and i > 0:
                    h = torch.cat([h, x], -1)
            
            if self.use_viewdirs:
                dir_input = h
                h = self.dir_layer(h)
                h = torch.relu(h)
                # [G] Assuming view directions are encoded elsewhere and passed as a separate argument in run_network
                # Here we assume the full forward pass handles concatenation
                pass
            
            raw = self.output_layer(h)
            return raw

    model = NeRF()
    return model

def run_network(inputs, network_fn, embed_fn, embeddirs_fn=None, netchunk=1024*64):
    """
    [G] Process inputs in chunks to manage memory.
    """
    # [G] Placeholder for actual chunking logic found in original repo
    # inputs: [N_rays, N_samples, 3] for xyz, [N_rays, N_samples, 2] for dirs
    # Returns: [N_rays, N_samples, 4] raw RGB+sigma
    pass

def setup_neRF(args, basedir):
    # 1. Load Data
    poses, bds, imgs = _load_data(basedir, factor=args.factor)
    
    # 2. Instantiate Encoders
    embed_fn, input_ch = get_embedder(args.multires, args.i_embed)
    embeddirs_fn = None
    if args.use_viewdirs:
        embeddirs_fn, input_ch_dirs = get_embedder(args.multires_views, args.i_embed)
    
    # 3. Create Coarse MLP
    model = init_nerf_model(
        netdepth=args.netdepth,
        netwidth=args.netwidth,
        input_ch=input_ch,
        output_ch=4,
        use_viewdirs=args.use_viewdirs,
        embed_fn=embed_fn,
        embeddirs_fn=embeddirs_fn
    )
    
    grad_vars = list(model.parameters())
    
    # 4. Create Fine MLP (Optional)
    model_fine = None
    if args.N_importance > 0:
        model_fine = init_nerf_model(
            netdepth=args.netdepth_fine,
            netwidth=args.netwidth_fine,
            input_ch=input_ch,
            output_ch=4,
            use_viewdirs=args.use_viewdirs,
            embed_fn=embed_fn,
            embeddirs_fn=embeddirs_fn
        )
        grad_vars.extend(model_fine.parameters())
    
    # 5. Define Network Query Function
    def network_query_fn(inputs):
        # [G] Simplified wrapper; actual implementation handles chunking and encoding
        xyz = inputs[..., :3]
        dirs = inputs[..., 3:]
        
        xyz_embedded = embed_fn(xyz)
        
        if args.use_viewdirs:
            dirs_embedded = embeddirs_fn(dirs)
            # [G] Logic to inject dirs into hidden layers is handled in init_nerf_model or run_network
            raw = run_network(xyz_embedded, model, embed_fn, embeddirs_fn)
            raw_fine = None
            if model_fine:
                raw_fine = run_network(xyz_embedded, model_fine, embed_fn, embeddirs_fn)
        else:
            raw = run_network(xyz_embedded, model, embed_fn)
            raw_fine = None
            if model_fine:
                raw_fine = run_network(xyz_embedded, model_fine, embed_fn)
        
        return raw, raw_fine

    # 6. Configure Training Render Kwargs
    render_kwargs_train = {
        'perturb': args.perturb,
        'N_importance': args.N_importance,
        'N_samples': args.N_samples,
        'network_fn': network_query_fn,
        'network_fine': network_query_fn if args.N_importance > 0 else None,
        'use_viewdirs': args.use_viewdirs,
        'white_bkgd': args.white_bkgd,
        'raw_noise_std': args.raw_noise_std
    }
    
    # 7. Configure Test Render Kwargs
    render_kwargs_test = {
        **render_kwargs_train,
        'perturb': False,
        'raw_noise_std': 0.
    }
    
    # 8. Load Checkpoints (Optional)
    ckpts = sorted([f for f in os.listdir(args.ckpts) if 'model_' in f and 'fine' not in f and 'optimizer' not in f])
    if ckpts:
        ft_weights = ckpts[-1]
        # [G] Assuming model has a set_weights method compatible with numpy arrays
        model.set_weights(np.load(os.path.join(args.ckpts, ft_weights), allow_pickle=True))
        
        if model_fine:
            fine_weights_name = '{}_fine_{}'.format(ft_weights[:-11], ft_weights[-10:])
            fine_weights_path = os.path.join(args.ckpts, fine_weights_name)
            if os.path.exists(fine_weights_path):
                model_fine.set_weights(np.load(fine_weights_path, allow_pickle=True))
                
    return model, model_fine, render_kwargs_train, render_kwargs_test, grad_vars
```

**Parameters:**
- `netdepth`: Number of layers in the MLP. Default: 8 [S1, unofficial].
- `netwidth`: Number of channels (hidden units) per layer. Default: 256 [S1, unofficial].
- `multires`: Log2 of the maximum frequency for positional encoding of 3D coordinates. Default: 10 [S1, unofficial].
- `multires_views`: Log2 of the maximum frequency for positional encoding of 2D view directions. Default: 4 [S1, unofficial].
- `N_samples`: Number of coarse samples per ray. Default: 64 [S1, unofficial].
- `N_importance`: Number of additional fine samples per ray. Default: 0 [S1, unofficial].
- `perturb`: Jitter factor for sampling. Default: 1.0. Set to 0. for no jitter [S1, unofficial].
- `raw_noise_std`: Standard deviation of noise added to the regularized $\sigma$ output. Default: 1e0 [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Precision**: Positional encoding frequencies can lead to high-frequency oscillations if `multires` is too high for the scene scale. [G] Ensure coordinate normalization is applied before encoding.
- **Boundaries**: The `embeddirs_fn` expects 2D spherical coordinates. Ensure view directions are normalized and converted to spherical coordinates correctly [S1, unofficial].
- **Degenerate Input**: If `N_importance` is 0, `model_fine` and `network_fine` should be `None` to avoid unnecessary computation [S1, unofficial].

## Subtopics

### Direct Voxel Grid Optimization (DVGO)
- **What it is**: A method that uses explicit 3D voxel grids to store density and color, optimized via differentiable rendering.
- **What sets it apart**: Uses explicit memory structures (voxels) instead of implicit neural networks (MLPs), leading to different memory and compute characteristics.
- **When to choose**: When memory constraints are tight or when faster training convergence is needed for specific scene types compared to MLPs.
- **Link**: [Direct Voxel Grid Optimization (DVGO)](2184-direct-voxel-grid-optimization-dvgo.md)

## Operating characteristics
- **Time Complexity**: Training time is dominated by the forward pass through the MLPs for each sample on each ray. With `N_samples` = 64 and `netdepth` = 8, the per-ray computation is significant. Fine sampling doubles this if `N_importance > 0` [S1, unofficial].
- **Memory Complexity**: The MLP weights are constant size ($O(\text{netdepth} \times \text{netwidth}^2)$). The memory usage scales with the number of rays and samples during training. Checkpoint loading requires storing model weights in memory [S1, unofficial].
- **Measured Performance**: Not given in the sources.
- **Scaling**: Scales linearly with the number of rays and samples. The use of `netchunk` in `run_network` helps manage GPU memory during large batches [S1, unofficial].
- **Strengths**: High-quality novel view synthesis; implicit representation handles complex geometry without explicit meshing.
- **Limitations**: Slow training and inference compared to explicit methods; requires careful tuning of positional encoding frequencies.
- **When to use**: High-quality static scene rendering where training time is acceptable.
- **When not**: Real-time applications or scenes with extremely large scale where positional encoding frequencies become problematic.

## Background
- **Origin**: The NeRF (Neural Radiance Fields) method was introduced by Mildenhall et al. in 2020 [S1, unofficial].
- **History**: The initialization code provided is from the official PyTorch implementation repository [S1, unofficial].
- **Usage**: Widely used in computer vision for novel view synthesis, 3D reconstruction, and scene representation.

## Sources
[S1] bmild/nerf, https://github.com/bmild/nerf, fetched 2026-10-26 (unofficial)
