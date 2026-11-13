# Hash Grid Neural Fields


## Overview
Hash Grid Neural Fields define a continuous scene function $F_\Theta: \mathbb{R}^d \rightarrow \mathbb{R}^k$ by mapping spatial coordinates to high-dimensional feature vectors using a cascade of learnable hash tables at different resolutions [G]. They solve the computational bottleneck of vanilla neural fields by replacing heavy MLP inference with fast hash table lookups and trilinear interpolation, enabling rapid training and rendering on consumer hardware [G]. Unlike frequency-based encodings, they learn spatially adaptive features directly in the grid, and unlike tree-based encodings, they avoid serial pointer-chasing to map efficiently to GPUs [G].

## Shared principles
The family builds on [Neural and primitive-based representations](10-neural-and-primitive-based-representations.md) and shares the mechanism of using multi-resolution hash tables to store spatial features, which are interpolated and concatenated before decoding [G]. All methods define a scene function $F_\Theta: \mathbb{R}^d \rightarrow \mathbb{R}^k$ where the input is a continuous coordinate vector (typically 3D spatial position $\mathbf{x}$, potentially augmented with viewing direction $\mathbf{d}$ or time $t$), and the output is a signal vector (e.g., density $\sigma$ and color $\mathbf{c}$) [G]. The network or primitives are trained by minimizing a loss between rendered observations and ground truth images [G].

## Subtopics

### [Neural Volumes with Multi-Resolution Hash Encoding](1497-neural-volumes-with-multi-resolution-hash-encoding.md)
A complete rendering pipeline for interactive volume rendering that compresses volumetric data into a neural representation. It solves the high inference cost of traditional neural fields by leveraging GPU tensor cores and a native CUDA neural network framework (TCNN) to achieve 10–60 fps rendering. Unlike [HashGrid Encoding](3866-hashgrid-encoding.md) which focuses on the encoding mechanism itself, InstantVNR provides a complete rendering pipeline including macro-cell acceleration for empty-space skipping and three distinct decoding paths (dense texture, sample streaming, and in-shader inference) to support real-time interaction.

### [Lagrangian Hashing](3242-lagrangian-hashing.md)
A hybrid representation combining the memory efficiency of Eulerian hierarchical hash grids (specifically InstantNGP) with the adaptive density control of Lagrangian point-based representations (such as 3D Gaussian Splatting). It solves the problem of high-frequency detail reconstruction by allowing feature points to move freely in space within hash buckets, rather than being fixed to uniform grid vertices. Unlike its sibling [HashGrid Encoding](3866-hashgrid-encoding.md), which uses static feature vectors per bucket, Lagrangian Hashing stores a mixture of Gaussians per bucket, enabling the representation to allocate its budget adaptively to regions requiring more detail.

### [HashGrid Encoding](3866-hashgrid-encoding.md)
Maps continuous spatial coordinates to high-dimensional feature vectors using a hierarchy of sparse hash tables at multiple resolutions. It solves the computational bottleneck of vanilla NeRF by replacing heavy MLP inference with fast hash lookups and trilinear interpolation. Accelerates training by 1000–10,000× and rendering by >3000× compared to vanilla NeRF, while maintaining comparable visual quality. Avoids serial pointer-chasing of tree-based encodings, mapping well to GPUs.

### [Occupancy Evaluation Function](6362-occupancy-evaluation-function.md)
Computes the importance weight (occupancy) at a given position in the occupancy grid, which is used by the sampling estimator to determine where to place samples. It transforms a raw neural field output (typically log-density) into a bounded occupancy probability using a sigmoid function. It differs from sibling methods by focusing specifically on the final scalar conversion step for sampling importance, rather than the feature extraction or hashing mechanism itself.

### [NeuS2 Network Architecture](10215-neus2-network-architecture.md)
Neural implicit surface reconstruction method combining multi-resolution hash encoding with separate networks for signed distance fields (SDF) and color. Decouples SDF and RGB prediction into distinct MLPs and employs a specific logistic density transformation for unbiased volume rendering. Differs from [Neural Volumes with Multi-Resolution Hash Encoding](1497-neural-volumes-with-multi-resolution-hash-encoding.md) by specifically decoupling the SDF and RGB prediction into distinct MLPs.

### IRIS Neural Architecture
- **Definition**: A neural architecture for IRIS that uses Ray-Coherent Aggregation (RCA) and a multi-resolution Hash Grid for feature initialization, followed by implicit geometry and color decoding MLPs [S1].
- **Feature Initialization**: Performed using a multi-resolution Hash Grid with 16 levels, spanning a resolution range from $N_{min} = 16$ to $N_{max} = 8192$ with a geometric growth factor. Each level stores 2 feature dimensions within a hash map of size $2^{21}$ [S1].
- **Implicit Geometry Decoder**: A shallow MLP with 2 hidden layers of 64 units and ReLU activations. It accepts a 32-dimensional aggregated feature input and produces a 16-dimensional output vector, where the first channel corresponds to the raw density $\sigma$, and the remaining 15 channels serve as the latent embedding for the subsequent color decoding head [S1].
- **RGB Network**: Shares an identical architecture with the geometry decoder (2 layers, 64 hidden units). It predicts the final 3-channel color value based on the concatenation of the geometric latent embedding and the view direction, encoded via Spherical Harmonics (SH) [S1].
- **Parameters**: The Ray-Coherent Aggregation (RCA) window size is consistently set to $N = 2$ [S1].

## Background
The family originated from the need to accelerate neural rendering on consumer hardware [G]. The core mechanism of multi-resolution hash encoding was popularized by [InstantNGP](https://nvlabs.github.io/instant-ngp/), which demonstrated that sparse hash tables could overcome the memory and computational bottlenecks of dense voxel grids and standard MLPs [G]. The approach has since been adapted into various forms, including Lagrangian variants for adaptive density and integrated into full rendering pipelines like InstantVNR for interactive volume rendering [G].

## Sources
[S1] IRIS: Intersection-aware Ray-based Implicit Editable Scenes, https://doi.org/10.48550/arXiv.2603.15368, fetched 2026-10-29

```python
import math

def sample_size_scaler(ss_base: float, t0: float, t1: float) -> float:
    """Calculates the actual step size for adaptive sampling."""
    if ss_base <= 0:
        return t1 - t0
    N = math.floor((t1 - t0) / ss_base) + 1
    return (t1 - t0) / N

def compute_intervals(org, dir, tMin, tMax):
    """Standard 3D DDA for an axis-aligned bounding box."""
    t_near = []
    t_far = []
    for i in range(3):
        if dir[i] == 0:
            if org[i] < tMin[i] or org[i] > tMax[i]:
                return []
            t_near.append(float('-inf'))
            t_far.append(float('inf'))
        else:
            t1 = (tMin[i] - org[i]) / dir[i]
            t2 = (tMax[i] - org[i]) / dir[i]
            t_near.append(min(t1, t2))
            t_far.append(max(t1, t2))
    t_entry = max(t_near)
    t_exit = min(t_far)
    if t_entry > t_exit or t_exit < 0:
        return []
    return [(t_entry, t_exit)]

def ray_march_adaptive(org, dir, tMin, tMax, ss_base, opacity_upper_bound, body_func):
    """Simulates the adaptive ray marching loop with macro-cell acceleration."""
    float_epsilon = 1e-6
    intervals = compute_intervals(org, dir, tMin, tMax)
    for t0, t1 in intervals:
        r = opacity_upper_bound(t0, t1)
        if abs(r) <= float_epsilon:
            continue
        ss = sample_size_scaler(ss_base, t0, t1)
        t_x = t0
        t_y = min(t1, t0 + ss)
        while t_y > t_x:
            body_func(t_x, t_y)
            t_x = t_y
            t_y = min(t_x + ss, t1)

def hash_grid_encode(inputs, offsets, embeddings, L, C, H, S, align_corners=True):
    """
    inputs: List of [B, D] float lists, values in [0, 1]
    offsets: List of [L+1] ints, cumulative hash table sizes
    embeddings: List of [sum(offsets), C] floats
    L: int, number of levels
    C: int, channels per level
    H: float, base resolution
    S: float, scaling factor exponent
    align_corners: bool, if True offset is 0.0, else 0.5
    """
    B, D = len(inputs), len(inputs[0])
    outputs = [[0.0] * C for _ in range(L)]
    dy_dx = [[[0.0] * C for _ in range(D)] for _ in range(B)]
    primes = [1, 2654435761, 805459861]

    for l in range(L):
        scale = math.exp2(l * S) * H - 1.0
        resolution = math.ceil(scale) + 1
        hashmap_size = offsets[l+1] - offsets[l]
        offset_val = 0.0 if align_corners else 0.5

        level_output = [0.0] * C
        level_grad_input = [[0.0] * C for _ in range(D)]

        for b in range(B):
            pos = [inputs[b][d] * scale + offset_val for d in range(D)]
            pos_grid = [int(math.floor(p)) for p in pos]
            frac = [pos[d] - pos_grid[d] for d in range(D)]

            for idx in range(1 << D):
                w = 1.0
                pos_grid_local = [0] * D
                for d in range(D):
                    if (idx >> d) & 1:
                        w *= frac[d]
                        pos_grid_local[d] = pos_grid[d] + 1
                    else:
                        w *= 1.0 - frac[d]
                        pos_grid_local[d] = pos_grid[d]

                hash_val = 0
                for d in range(D):
                    hash_val ^= (pos_grid_local[d] * primes[d])

                base_idx = (hash_val % hashmap_size) * C
                for c in range(C):
                    level_output[c] += w * embeddings[base_idx + c]

        outputs[l] = level_output
        # Gradient w.r.t embeddings would be accumulated here via atomicAdd in CUDA
        # For dy_dx, full implementation requires partial derivatives along each dim
        # This is simplified; full dy_dx requires partial derivatives along each dim
    return outputs, dy_dx

def lagrangian_hashing_forward(x, levels, hash_tables, decoder_mlp, N_min, b, B, sigma_scale, F_dim):
    """
    x: Query coordinate vector of shape (D,)
    levels: List of level indices [1, ..., L]
    hash_tables: List of feature tensors F_l for each level
    decoder_mlp: Neural network decoder
    N_min: Minimum resolution base
    b: Resolution scaling base
    B: Bucket size
    sigma_scale: Scaling factor for Gaussian standard deviation relative to grid resolution
    F_dim: Output feature dimension
    Returns: Output feature vector
    """
    concatenated_features = []
    
    for l in levels:
        N_l = N_min * (b ** l)
        corner_indices, alpha_weights = compute_grid_corners_and_weights(x, N_l)
        level_feature = [0.0] * F_dim
        
        for v, alpha_v in zip(corner_indices, alpha_weights):
            bucket_feature = get_bucket_feature(hash_tables[l], v, x, N_l, sigma_scale)
            level_feature = vector_add(level_feature, vector_scale(bucket_feature, alpha_v))
            
        concatenated_features.extend(level_feature)
        
    return decoder_mlp(concatenated_features)

def compute_grid_corners_and_weights(x, N_l):
    D = len(x)
    indices = []
    weights = []
    for mask in range(1 << D):
        corner = []
        w = 1.0
        for d in range(D):
            if (mask >> d) & 1:
                idx = math.floor(x[d] * N_l)
                frac = x[d] * N_l - idx
                corner.append(idx)
                w *= frac
            else:
                idx = math.floor(x[d] * N_l)
                frac = 1.0 - (x[d] * N_l - idx)
                corner.append(idx)
                w *= frac
        indices.append(tuple(corner))
        weights.append(w)
    return indices, weights

def get_bucket_feature(hash_table, v, x, N_l, sigma_scale):
    points = hash_table[v].points
    if points is None:
        return hash_table[v].static_feature
    sigma = sigma_scale * N_l
    weighted_sum = [0.0] * len(points[0].feature)
    total_weight = 0.0
    for p in points:
        dist_sq = sum((x[d] - p.center[d]) ** 2 for d in range(len(x)))
        weight = math.exp(-dist_sq / (2 * sigma ** 2))
        weighted_sum = vector_add(weighted_sum, vector_scale(p.feature, weight))
        total_weight += weight
    if total_weight > 0:
        return vector_scale(weighted_sum, 1.0 / total_weight)
    return [0.0] * len(points[0].feature)

def vector_add(a, b):
    return [a[i] + b[i] for i in range(len(a))]

def vector_scale(v, s):
    return [v[i] * s for i in range(len(v))]

def evaluate_occupancy(x_AABB, use_gt_ne, ne_field, aabb_scale, clamp, 
                       log_ne_min, log_ne_max, occ_eval_fn_ne_mid, 
                       occ_eval_fn_ne_width, predict_learned_fields):
    """Evaluates the occupancy at a given AABB coordinate."""
    if use_gt_ne and ne_field is not None:
        x_xyz = x_AABB * aabb_scale
        log10_ne = ne_field(x_xyz)
        if clamp:
            log10_ne = max(log_ne_min, min(log10_ne, log_ne_max))
        ne = 10.0 ** log10_ne  # Computed per source, but not used in sigmoid
    else:
        _, log10_ne, _, _ = predict_learned_fields(x_AABB)

    exponent = (log10_ne - occ_eval_fn_ne_mid) / occ_eval_fn_ne_width
    occ = 1.0 / (1.0 + math.exp(-exponent))
    return occ
