# Procedural Terrain JSON Schema

Also called: Procedural Terrain Storage Format (JSON)

## Overview
The Procedural Terrain JSON Schema defines a data structure for storing the minimal set of parameters required to reproduce a designed procedural terrain in external applications [S1, unofficial]. It serves as an export format that decouples terrain design from rendering, allowing the same terrain to be synthesized using different base noise implementations or rendering engines [S1, unofficial]. Unlike [Noise Layer Evaluation](7076-noise-layer-evaluation.md), which describes the runtime evaluation of a single layer, this schema defines the persistent storage format for a collection of layers and their global bounds [S1, unofficial].

## How it works
The schema represents a terrain as a root object containing an array of noise layers and global extrema. Each layer defines a base noise frequency, a redistribution function (RD) for non-linear mapping, amplitude scaling, and optional masking against other layers [S1, unofficial].

### Inputs and Outputs
- **Input**: A JSON document conforming to the schema.
- **Output**: A set of parameters used to reconstruct the height field $H(i,j)$ by summing processed layers.

### Data Structures
The root object contains:
- `noiseLayers`: An array of layer objects.
- `maxValue`: The expected maximum value of the final terrain function [S1, unofficial].
- `minValue`: The expected minimum value of the final terrain function [S1, unofficial].

Each layer object contains:
- `baseNoise`: An object with a `frequency` field (number). This scaling is applied directly to the base noise (e.g., SimplexNoise) before any other processing [S1, unofficial].
- `rd`: An object containing `points`, an array of objects with `x` and `y` fields (numbers). These are equalized points used to construct the redistribution function [S1, unofficial].
- `preAmpShift`: A number added to the noise output before scaling [S1, unofficial].
- `ampScale`: A number multiplying the translated noise value [S1, unofficial].
- `maskLayerIndex`: An integer index pointing to another layer in `noiseLayers` used for masking [S1, unofficial].
- `maskMinValue`: A number representing the minimum value of the masked range [S1, unofficial].
- `maskMaxValue`: A number representing the maximum value of the masked range [S1, unofficial].

### Processing Steps
To evaluate the terrain at coordinates $(i,j)$:

1. **Initialize Sum**: Let $H(i,j) = 0$.
2. **Iterate Layers**: For each layer $k$ in `noiseLayers`:
    a. **Base Noise**: Compute $n_k = \text{BaseNoise}(i \cdot \text{baseNoise.frequency}, j \cdot \text{baseNoise.frequency})$. The base noise implementation must match the one used during design [S1, unofficial].
    b. **Redistribution**: Map $n_k$ through the redistribution function defined by `rd.points`. Let the result be $r_k$. The redistribution function is constructed from the equalized points [S1, unofficial].
    c. **Amplitude Scaling**: Compute the processed value $X_k$ using the formula:
       $$X_k = (r_k + \text{preAmpShift}) \times \text{ampScale}$$
    d. **Masking**: If `maskLayerIndex` is valid (i.e., $0 \le \text{maskLayerIndex} < \text{len(noiseLayers)}$):
       i. Retrieve the processed value $X_m$ from the layer at `maskLayerIndex`.
       ii. If $X_m < \text{maskMinValue}$ or $X_m > \text{maskMaxValue}$, discard $X_k$ (set $X_k = 0$ or skip addition) [S1, unofficial].
    e. **Accumulate**: Add $X_k$ to $H(i,j)$.
3. **Result**: The final height is $H(i,j)$.

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def evaluate_terrain(json_data, i, j, base_noise_func):
    """
    Evaluates the procedural terrain defined by json_data at coordinates (i, j).
    
    Args:
        json_data: Dict conforming to the schema.
        i, j: Integer coordinates.
        base_noise_func: Function that takes (x, y) and returns a float.
        
    Returns:
        Float height value.
    """
    noise_layers = json_data['noiseLayers']
    total_height = 0.0
    
    # Pre-evaluate all layers to handle masking dependencies
    # Note: Masking refers to other layers. The order of summation 
    # in the schema description implies sequential processing, 
    # but masking requires access to the *processed* value of another layer.
    # We must store processed values of all layers first.
    
    processed_values = []
    
    for layer in noise_layers:
        # 1. Base Noise
        freq = layer['baseNoise']['frequency']
        noise_val = base_noise_func(i * freq, j * freq)
        
        # 2. Redistribution
        # Construct RD from points. Assuming linear interpolation between points.
        rd_points = layer['rd']['points']
        # [G] Implementation detail: Interpolate noise_val through the 
        # redistribution function defined by rd_points. 
        # rd_points are sorted by x. Map noise_val (x) to y.
        rd_val = interpolate_redistribution(rd_points, noise_val)
        
        # 3. Amplitude Scaling
        pre_amp_shift = layer['preAmpShift']
        amp_scale = layer['ampScale']
        x_val = (rd_val + pre_amp_shift) * amp_scale
        
        # 4. Masking
        mask_index = layer['maskLayerIndex']
        if mask_index is not None and 0 <= mask_index < len(noise_layers):
            # Check if the masked layer's processed value is within range
            # Note: We need the processed value of the masked layer.
            # If the masked layer appears later in the array, we might not have it yet.
            # However, masking is typically against a layer that has already been 
            # processed or is independent. The schema implies the mask layer's 
            # *value* is used. We assume the mask layer's processed value is 
            # available. If not, this requires a two-pass approach or 
            # dependency resolution. For simplicity, we assume the mask layer 
            # is either already processed or the mask check is against a 
            # static property. 
            # [G] Standard interpretation: Masking is against the final processed 
            # value of the referenced layer. If the referenced layer is later 
            # in the array, the order of summation matters. The schema does not 
            # specify order independence. We assume the mask layer is processed 
            # before or the mask check is against the layer's *own* processed 
            # value if it were evaluated. 
            # To be safe, we store processed values in a list.
            masked_val = processed_values[mask_index]
            mask_min = layer['maskMinValue']
            mask_max = layer['maskMaxValue']
            
            if masked_val < mask_min or masked_val > mask_max:
                # Discard current layer's contribution
                processed_values.append(0.0)
                continue
        
        processed_values.append(x_val)
        total_height += x_val
        
    return total_height

def interpolate_redistribution(points, x):
    """
    Interpolates y for a given x using the provided points.
    Points are sorted by x.
    """
    if not points:
        return x # Identity if no RD
    
    # Find the segment
    for k in range(len(points) - 1):
        p1 = points[k]
        p2 = points[k+1]
        if p1['x'] <= x <= p2['x']:
            # Linear interpolation
            if p2['x'] == p1['x']:
                return p1['y']
            t = (x - p1['x']) / (p2['x'] - p1['x'])
            return p1['y'] + t * (p2['y'] - p1['y'])
    
    # Extrapolate or clamp? Schema doesn't specify.
    # [G] Clamp to the range of the points.
    if x < points[0]['x']:
        return points[0]['y']
    else:
        return points[-1]['y']
```

### Parameters and Edge Cases
- **Base Noise Consistency**: The application rendering the terrain must use the same base noise implementation (e.g., Perlin's reference Simplex) as the design tool [S1, unofficial].
- **Masking Order**: If a layer masks another layer that appears later in the array, the evaluation order must respect this dependency. The schema does not explicitly define evaluation order, but masking implies access to the processed value of the target layer.
- **Redistribution Construction**: The `rd.points` are "equalized points." The exact method of constructing the function (e.g., linear interpolation, spline) is not specified in the schema but is implied to be implemented by the application [S1, unofficial].

## Operating characteristics
- **Time Complexity**: $O(L \cdot N)$, where $L$ is the number of layers and $N$ is the number of pixels/voxels. Each layer requires one noise evaluation, one redistribution lookup, and optional masking check.
- **Memory Complexity**: $O(L)$ for storing the schema parameters. The terrain height field requires $O(N)$ memory.
- **Strengths**: Decouples design from implementation; allows for non-linear height mapping via redistribution; supports complex masking logic.
- **Limitations**: Requires the same base noise implementation in the consumer application; masking dependencies may complicate parallel evaluation if not ordered correctly.

## Background
The schema was introduced in a thesis on terrain synthesis using noise [S1, unofficial]. It was designed to allow procedural terrains to be exported from a design tool and imported into applications for rendering. The approach uses noise layers, redistribution functions, and masking to create varied terrain features.

## Sources
[S1] Terrain synthesis using noise, https://trepo.tuni.fi/handle/10024/101043, fetched 2026-10-26 (unofficial)
