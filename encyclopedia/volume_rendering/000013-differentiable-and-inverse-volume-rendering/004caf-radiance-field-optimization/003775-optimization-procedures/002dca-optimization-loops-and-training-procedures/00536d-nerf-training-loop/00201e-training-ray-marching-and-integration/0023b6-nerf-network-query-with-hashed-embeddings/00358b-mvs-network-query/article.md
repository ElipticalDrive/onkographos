# MVS Network Query


## Overview
MVS Network Query prepares inputs for a neural network by embedding points and view directions, then applies the network in batches to generate features for multi-view stereo (MVS) rendering [S1, unofficial]. It differs from standard NeRF network queries by explicitly handling multi-view geometric consistency through the generation of angle and point features derived from camera poses and depth candidates, rather than relying solely on spatial coordinates and view directions for a single viewpoint [S1, unofficial].

## How it works
The method consists of two main phases: input preparation for the neural network and the MVS rendering pipeline that utilizes these outputs.

### Inputs and Outputs
**Inputs:**
- Points: 3D spatial coordinates $(x, y, z)$.
- View directions: Unit vectors $(d_x, d_y, d_z)$.
- Optional embedding functions: `embed_fn` for points, `embeddirs_fn` for view directions.
- Optional alpha features: `alpha_feat` concatenated with points.
- Reference pose: `pose_ref` for angle feature generation.
- Depth candidates: Discrete depth values for sampling.
- Input features: `input_feat` for channel concatenation.

**Outputs:**
- Raw network outputs: Tensors containing color, density, and auxiliary features.
- Final rendered images: Computed via warping and aggregation of features from multiple views.

### Data Structures
- **Points**: Array of shape $(N, 3)$, where $N$ is the number of sample points.
- **View Directions**: Array of shape $(N, 3)$, normalized.
- **Angle Features**: Array of shape $(N, F_a)$, derived from the cosine of the angle between ray directions and reference camera axes.
- **Point Features**: Array of shape $(N, F_p)$, derived from depth candidates and geometric constraints.
- **Network Input**: Concatenated tensor of shape $(N, 3 + F_a + F_p + \dots)$.
- **Network Output**: Flat tensor of shape $(N, C)$, where $C$ is the number of output channels (typically $\ge 4$ for RGB + alpha, plus auxiliary features).

### Steps

#### 1. Input Preparation
1. **Embed Points**: If `embed_fn` is provided, apply it to the points. Otherwise, use raw coordinates [S1, unofficial].
2. **Concatenate Alpha Features**: If `alpha_feat` is provided, concatenate it with the points/embedded points along the feature dimension [S1, unofficial].
3. **Process View Directions**:
   - Expand dimensions of `viewdirs` if necessary to match batch dimensions [S1, unofficial].
   - If `embeddirs_fn` is provided, apply it to `viewdirs` [S1, unofficial].
4. **Concatenate Inputs**: Concatenate the processed points and view directions along the feature dimension [S1, unofficial].
5. **Determine Output Requirements**: Check if only alpha (density) is needed based on the presence of `viewdirs`. If `viewdirs` is absent, the network may output only density [S1, unofficial].
6. **Batched Network Application**: Apply the neural network function in batches using a `batchify` mechanism to manage memory constraints. The batch size is determined by available GPU memory and input dimensions [S1, unofficial].
7. **Reshape Outputs**: Reshape the flat outputs from the network to match the input point dimensions, resulting in a tensor of shape $(N, C)$ [S1, unofficial].

#### 2. MVS Rendering Pipeline
1. **Compute Cosine Angle**: Calculate the norm of ray directions as $\cos\_angle = \| \mathbf{d} \|$ (typically 1.0 for normalized vectors, but used for angular encoding) [S1, unofficial].
2. **Generate Angle Features**:
   - If `pose_ref` (reference camera pose) is available, compute features based on the relative orientation between the current ray and the reference camera [S1, unofficial].
   - Otherwise, use normalized ray directions as angle features [S1, unofficial].
3. **Generate Point Features**: Use a function `gen_pts_feats` to generate features based on depth candidates and geometric consistency across views [S1, unofficial].
4. **Query Network**: Pass the concatenated angle and point features (and optionally view directions) through the neural network to obtain raw outputs [S1, unofficial].
5. **Feature Concatenation**: If the raw output has more than 4 channels, concatenate the first 8 channels of `input_feat` with the raw output channels starting from index 4. This integrates multi-view geometric features with the radiance field predictions [S1, unofficial].
6. **Depth to Distance Conversion**: Convert depth candidates to distances using the camera intrinsics and ray directions [S1, unofficial].
7. **Final Rendering**: Convert raw outputs to final rendering results by aggregating color and alpha values along the ray, weighted by density and visibility [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def run_network_mvs(points, viewdirs, embed_fn=None, embeddirs_fn=None, 
                    alpha_feat=None, pose_ref=None, input_feat=None, 
                    depth_candidates=None, network=None, batch_size=100000):
    """
    Prepares inputs for MVS network and queries it in batches.
    """
    # Step 1: Embed Points
    if embed_fn is not None:
        pts_embedded = embed_fn(points)
    else:
        pts_embedded = points
    
    # Step 2: Concatenate Alpha Features
    if alpha_feat is not None:
        pts_combined = torch.cat([pts_embedded, alpha_feat], dim=-1)
    else:
        pts_combined = pts_embedded
    
    # Step 3: Process View Directions
    if viewdirs is not None:
        # Expand dimensions if necessary
        if viewdirs.dim() < pts_combined.dim():
            viewdirs = viewdirs.unsqueeze(1)
        
        if embeddirs_fn is not None:
            dirs_embedded = embeddirs_fn(viewdirs)
        else:
            dirs_embedded = viewdirs
    else:
        dirs_embedded = None
    
    # Step 4: Concatenate Points and View Directions
    if dirs_embedded is not None:
        inputs = torch.cat([pts_combined, dirs_embedded], dim=-1)
    else:
        inputs = pts_combined
    
    # Step 5: Determine if only alpha is needed
    # This is a heuristic based on the presence of viewdirs
    only_alpha = (viewdirs is None)
    
    # Step 6: Apply Network in Batches
    outputs = []
    num_points = inputs.shape[0]
    for i in range(0, num_points, batch_size):
        batch_input = inputs[i:i+batch_size]
        batch_output = network(batch_input)
        outputs.append(batch_output)
    
    raw_outputs = torch.cat(outputs, dim=0)
    
    # Step 7: Reshape Outputs
    # Assuming raw_outputs is flat (N, C), reshape to match input dimensions
    # If network outputs multiple channels, this step ensures correct shape
    final_outputs = raw_outputs.reshape(num_points, -1)
    
    return final_outputs

def mvs_rendering_pipeline(points, viewdirs, pose_ref, input_feat, 
                           depth_candidates, network, batch_size=100000):
    """
    Performs MVS rendering by generating features, querying network, and converting outputs.
    """
    # Step 1: Compute Cosine Angle
    cos_angle = torch.norm(viewdirs, dim=-1, keepdim=True)
    
    # Step 2: Generate Angle Features
    if pose_ref is not None:
        # Compute relative orientation features
        angle_feats = compute_angle_features(viewdirs, pose_ref)
    else:
        angle_feats = viewdirs
    
    # Step 3: Generate Point Features
    point_feats = gen_pts_feats(points, depth_candidates)
    
    # Step 4: Query Network
    # Combine angle and point features
    mvs_inputs = torch.cat([angle_feats, point_feats], dim=-1)
    raw = run_network_mvs(mvs_inputs, viewdirs=None, network=network, batch_size=batch_size)
    
    # Step 5: Feature Concatenation
    if raw.shape[1] > 4 and input_feat is not None:
        # Concatenate first 8 channels of input_feat with raw channels 4 onwards
        input_feat_expanded = input_feat.unsqueeze(0).expand(raw.shape[0], -1, -1)
        raw_features = raw[:, 4:]
        input_feat_concat = input_feat_expanded[:, :, :8]
        raw = torch.cat([raw[:, :4], torch.cat([input_feat_concat, raw_features], dim=-1)], dim=-1)
    
    # Step 6: Convert Depth Candidates to Distances
    distances = depth_to_distance(depth_candidates, viewdirs)
    
    # Step 7: Convert Raw Outputs to Final Rendering Results
    final_images = render_from_raw(raw, distances, viewdirs)
    
    return final_images
```

### Parameters
- `batch_size`: Number of points processed per network forward pass. Typical value: 100,000 [S1, unofficial].
- `embed_fn`: Optional function for point embedding. Not given in the sources.
- `embeddirs_fn`: Optional function for view direction embedding. Not given in the sources.
- `pose_ref`: Reference camera pose for angle feature generation. Not given in the sources.
- `input_feat`: Input features for channel concatenation. Not given in the sources.
- `depth_candidates`: Discrete depth values for sampling. Not given in the sources.

### Edge Cases and Pitfalls
- **Precision**: Depth conversion and angle feature computation require high precision to avoid artifacts in multi-view consistency [S1, unofficial].
- **Boundaries**: Ensure that points are within the valid viewing frustum of all reference cameras to avoid invalid features [S1, unofficial].
- **Degenerate Input**: If `viewdirs` is None, the network may output only density, which should be handled appropriately in the rendering pipeline [S1, unofficial].

## Subtopics

### MVS Rendering
MVS Rendering is a variant that performs multi-view stereo rendering by generating features, querying the network, and converting raw outputs to final images [S1, unofficial]. It differs from standard NeRF rendering by explicitly incorporating multi-view geometric consistency through angle and point features derived from camera poses and depth candidates [S1, unofficial]. Choose this variant when multi-view consistency is critical for accurate reconstruction, such as in structured scenes with known camera poses [S1, unofficial].

## Operating characteristics
- **Time Complexity**: Linear with respect to the number of sample points $N$, $O(N)$, due to batched network queries [S1, unofficial].
- **Memory Complexity**: Linear with respect to the number of sample points $N$, $O(N)$, for storing inputs and outputs [S1, unofficial].
- **Measured Performance**: Not given in the sources.
- **Scaling**: Scales well with batch size, limited by GPU memory [S1, unofficial].
- **Strengths**: Explicitly handles multi-view consistency, leading to more accurate reconstructions in structured scenes [S1, unofficial].
- **Limitations**: Requires known camera poses and depth candidates, limiting applicability to unstructured scenes [S1, unofficial].
- **When to Use**: Suitable for scenes with known camera poses and multi-view data, such as indoor environments or object scans [S1, unofficial].
- **When Not to Use**: Not suitable for scenes with unknown camera poses or single-view data [S1, unofficial].

## Background
MVS Network Query originates from the PointNeRF project, which integrates multi-view stereo information into neural radiance fields for improved reconstruction accuracy [S1, unofficial]. The method was developed to address the limitations of standard NeRF in handling multi-view consistency, leveraging geometric features from multiple camera views to enhance the quality of the rendered images [S1, unofficial].

## Sources
[S1] Xharlie/pointnerf, https://github.com/Xharlie/pointnerf, fetched 2026-10-26 (unofficial)
