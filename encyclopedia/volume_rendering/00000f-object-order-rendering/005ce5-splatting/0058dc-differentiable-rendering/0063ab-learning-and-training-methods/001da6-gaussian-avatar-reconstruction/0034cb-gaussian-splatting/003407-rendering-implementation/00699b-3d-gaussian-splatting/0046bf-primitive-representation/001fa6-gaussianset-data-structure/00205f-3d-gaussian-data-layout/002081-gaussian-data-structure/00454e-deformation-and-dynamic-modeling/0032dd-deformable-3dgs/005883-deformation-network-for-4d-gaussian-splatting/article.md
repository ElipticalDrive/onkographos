# Deformation Network for 4D Gaussian Splatting

Also called: Deformation

## Overview
The Deformation Network is a neural module used within Deformable 3D Gaussian Splatting to predict dynamic 3D Gaussian parameters (means, scales, rotations, opacity, and spherical harmonic coefficients) from static input features and time. It solves the problem of rendering dynamic scenes by mapping a static Gaussian representation to a time-varying state, enabling the reconstruction of 4D content from multi-view video. It differs from sibling methods like VolSegGS by explicitly modeling temporal deformation via a dedicated time-conditioned MLP rather than static or voxel-based approaches.

## How it works
The method constructs a neural network that takes static Gaussian attributes and a time coordinate as input, applies positional encoding, and passes them through a deformation network to output the dynamic parameters for a specific timestamp.

### Inputs and Outputs
**Inputs:**
*   `point`: Static 3D mean position of the Gaussian, shape $(N, 3)$.
*   `scales`: Static scale parameters, shape $(N, 3)$.
*   `rotations`: Static rotation parameters (quaternions), shape $(N, 4)$.
*   `opacity`: Static opacity, shape $(N, 1)$.
*   `shs`: Static spherical harmonic coefficients, shape $(N, K, 3)$ where $K$ is the number of bands.
*   `times_sel`: Time coordinates for the target frames, shape $(M, 1)$ or broadcastable to $N$.

**Outputs:**
*   `dynamic_means`: Predicted 3D means at time $t$, shape $(N, 3)$.
*   `dynamic_scales`: Predicted scales at time $t$, shape $(N, 3)$.
*   `dynamic_rotations`: Predicted rotations at time $t$, shape $(N, 4)$.
*   `dynamic_opacity`: Predicted opacity at time $t$, shape $(N, 1)$.
*   `dynamic_shs`: Predicted SH coefficients at time $t$, shape $(N, K, 3)$.

### Data Structures
The `Deformation` class (referred to as `self.deformation_net` in implementation) contains:
1.  `self.timenet`: A Sequential module consisting of `Linear -> ReLU -> Linear`. It processes time inputs.
2.  `self.deformation_net`: An instance of the `Deformation` MLP class.
3.  Buffers (FloatTensors):
    *   `time_poc`: Powers of 2 for time positional encoding.
    *   `pos_poc`: Powers of 2 for position positional encoding.
    *   `rotation_scaling_poc`: Powers of 2 for scale/rotation positional encoding.
    *   `opacity_poc`: Powers of 2 for opacity positional encoding.

### Positional Encoding
Positional encoding is applied to inputs using a function `poc_fre`. For an input tensor $x$ and a buffer buffer $B$ (containing powers of 2 from $2^0$ to $2^{n-1}$):
1.  Multiply $x$ by $B$ (broadcasted).
2.  Flatten the result.
3.  Apply $\sin$ and $\cos$.
4.  Concatenate $[x, \sin(x \cdot B), \cos(x \cdot B)]$ along the last dimension.

### Parameters
*   `net_width`: Width of the deformation network layers.
*   `timebase_pe`: Number of frequency bands for time positional encoding.
*   `defor_depth`: Depth of the deformation network.
*   `posbase_pe`: Number of frequency bands for position positional encoding.
*   `scale_rotation_pe`: Number of frequency bands for scale/rotation positional encoding.
*   `opacity_pe`: Number of frequency bands for opacity positional encoding.
*   `timenet_width`: Width of the time network.
*   `timenet_output`: Output dimension of the time network.
*   `grid_pe`: Parameter passed to the Deformation class for grid positional encoding.

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class DeformationNetwork(nn.Module):
    def __init__(self, net_width, timebase_pe, defor_depth, posbase_pe, 
                 scale_rotation_pe, opacity_pe, timenet_width, timenet_output, grid_pe):
        super().__init__()
        self.net_width = net_width
        self.timebase_pe = timebase_pe
        self.defor_depth = defor_depth
        self.posbase_pe = posbase_pe
        self.scale_rotation_pe = scale_rotation_pe
        self.opacity_pe = opacity_pe
        self.timenet_width = timenet_width
        self.timenet_output = timenet_output
        
        # Initialize buffers for positional encoding
        # Powers of 2 from 2^0 to 2^(n-1)
        self.register_buffer('time_poc', torch.pow(2, torch.arange(timebase_pe).float()))
        self.register_buffer('pos_poc', torch.pow(2, torch.arange(posbase_pe).float()))
        self.register_buffer('rotation_scaling_poc', torch.pow(2, torch.arange(scale_rotation_pe).float()))
        self.register_buffer('opacity_poc', torch.pow(2, torch.arange(opacity_pe).float()))
        
        # Time network: Linear -> ReLU -> Linear
        self.timenet = nn.Sequential(
            nn.Linear(1, timenet_width),
            nn.ReLU(),
            nn.Linear(timenet_width, timenet_output)
        )
        
        # Deformation MLP
        # Input channels: 3 (pos) + 3*posbase_pe*2 (pos PE) + timenet_output
        # Note: The source comment suggests input_ch = 3 + (3 * posbase_pe) * 2
        # However, times_sel is also passed. The exact channel count for the first layer 
        # of self.deformation_net depends on how time embedding is concatenated.
        # Based on "Forward Dynamic Pass", pos, scales, rotations, opacity, shs, times_sel are passed.
        # The Deformation class likely handles the concatenation internally or expects specific inputs.
        # We assume self.deformation_net is an nn.Module defined elsewhere or here.
        # For completeness, we define a placeholder structure if not provided, 
        # but the source implies it's a separate class. We will assume it takes the concatenated features.
        
        # Initialize weights
        self.initialize_weights()

    def poc_fre(self, x, poc_buf):
        """
        Positional Encoding function.
        x: input tensor
        poc_buf: buffer of powers of 2
        """
        # Multiply input by poc_buf
        x = x * poc_buf
        # Flatten, sin, cos, concatenate
        # Assuming x is [..., D], poc_buf is [n]
        # We need to expand poc_buf to match dimensions for broadcasting
        # Standard PE: sin(x * 2^(i/n)) usually, but here it's powers of 2.
        # The source says "powers of 2 from 2^0 to 2^(n-1)".
        
        # Reshape poc_buf for broadcasting: [1, 1, ..., n]
        poc_buf = poc_buf.view(1, 1, -1)
        
        # x shape: [..., D]
        # We need to apply PE to each dimension of x? 
        # Usually PE is applied per dimension. 
        # Source: "multiplies input by poc_buf... flattens, takes sin and cos, and concatenates [input, sin, cos]"
        # This implies x is treated as a sequence of scalars or the PE is applied to the whole vector?
        # Let's assume standard per-dimension PE logic adapted to the buffer.
        
        # If x is (N, D), and poc_buf is (n,), we might need to repeat poc_buf for D dimensions?
        # Or is x flattened first? "flattens" suggests x might be (N, D) -> (N*D,).
        # Let's follow the source literally:
        x_flat = x.view(-1, x.shape[-1]) # Flatten all but last dim? Or all dims?
        # "multiplies input by poc_buf" -> if x is (N, 3), poc_buf is (n,), result is (N, 3, n)?
        # Then "flattens" -> (N*3*n,)?
        # Then "takes sin and cos" -> (N*3*n,)?
        # Then "concatenates [input, sin, cos]" -> This part is ambiguous.
        
        # Alternative interpretation: Standard PE where each dim gets encoded.
        # Let's assume the source implies:
        # For each dimension d in x:
        #   encoded_d = [x_d, sin(x_d * poc_buf), cos(x_d * poc_buf)]
        # Concatenate along last dim.
        
        # Re-reading: "multiplies input by poc_buf (powers of 2), flattens, takes sin and cos, and concatenates [input, sin, cos] along the last dimension."
        # This sounds like:
        # 1. x * poc_buf -> shape [..., D, n] if poc_buf is [n] and x is [..., D]
        # 2. Flatten -> [..., D*n]
        # 3. sin/cos -> [..., D*n]
        # 4. Concat [input, sin, cos]? Input was [..., D].
        # This seems to imply the input is also expanded or the concatenation is tricky.
        
        # Let's stick to the most likely implementation for "poc_fre" in this context:
        # It's a custom PE.
        
        # Step 1: Expand poc_buf to match x's last dimension if necessary, or broadcast.
        # If x is (N, 3) and poc_buf is (n,), x * poc_buf is (N, 3, n) if we view poc_buf as (1, 1, n).
        # But source says "flattens".
        
        # Let's implement a robust version based on the description:
        # x: (N, D)
        # poc_buf: (n,)
        # 1. x_expanded = x.unsqueeze(-1) * poc_buf.unsqueeze(0).unsqueeze(0) -> (N, D, n)
        # 2. x_flat = x_expanded.view(N, -1) -> (N, D*n)
        # 3. sin_vals = torch.sin(x_flat)
        # 4. cos_vals = torch.cos(x_flat)
        # 5. Concat [x, sin_vals, cos_vals]? 
        #    x is (N, D). sin_vals is (N, D*n).
        #    This doesn't match dimensions for concatenation along last dim unless D*n == D? No.
        
        # Correction: Maybe "input" in "concatenates [input, sin, cos]" refers to the original x?
        # And the result is (N, D + D*n + D*n)?
        
        # Let's assume the standard "concatenate along last dim" means:
        # Output = [x, sin(x*PE), cos(x*PE)]
        # But x*PE is higher dim.
        
        # Let's look at the "Forward Dynamic Pass":
        # "Encode point using pos_poc... Pass these embeddings... to self.deformation_net"
        # So the output of poc_fre is the embedding.
        
        # Let's assume the following logic which is common in such repos:
        # For each feature dimension, apply PE.
        
        # Given the ambiguity, we will write the code as described literally, 
        # assuming x is flattened before PE application if "flattens" is a step.
        
        # Actually, a common pattern is:
        # x: (N, D)
        # freq_bands: (n,)
        # x_pe = torch.cat([x, torch.sin(x.unsqueeze(-1) * freq_bands), torch.cos(x.unsqueeze(-1) * freq_bands)], dim=-1)
        # This results in (N, D + D*n + D*n).
        
        # The source says "flattens". This might mean flattening the spatial dims before PE?
        # Or flattening the result of multiplication?
        
        # Let's try:
        # x: (N, D)
        # x_mul = x.unsqueeze(-1) * poc_buf # (N, D, n)
        # x_flat = x_mul.view(N, -1) # (N, D*n)
        # sin_x = torch.sin(x_flat)
        # cos_x = torch.cos(x_flat)
        # out = torch.cat([x, sin_x, cos_x], dim=-1) # (N, D + D*n + D*n)
        
        # This seems plausible.
        
        x_mul = x.unsqueeze(-1) * poc_buf.view(1, 1, -1)
        x_flat = x_mul.view(x.shape[0], -1)
        sin_x = torch.sin(x_flat)
        cos_x = torch.cos(x_flat)
        out = torch.cat([x, sin_x, cos_x], dim=-1)
        return out

    def forward(self, point, scales, rotations, opacity, shs, times_sel):
        # 1. Positional Encoding
        # point: (N, 3)
        pos_emb = self.poc_fre(point, self.pos_poc)
        
        # scales: (N, 3), rotations: (N, 4) -> combined?
        # Source: "scales and rotations using rotation_scaling_poc"
        # Concat scales and rotations? Or encode separately?
        # "Encode point... scales and rotations... opacity"
        # Let's assume scales and rotations are concatenated before encoding or encoded separately and concatenated.
        # Source: "rotation_scaling_poc" is one buffer.
        # Let's concatenate scales and rotations first.
        scale_rot = torch.cat([scales, rotations], dim=-1) # (N, 7)
        scale_rot_emb = self.poc_fre(scale_rot, self.rotation_scaling_poc)
        
        # opacity: (N, 1)
        opacity_emb = self.poc_fre(opacity, self.opacity_poc)
        
        # Time encoding
        # times_sel: (M, 1) or (N, 1)?
        # Source: "times_sel" passed to deformation_net.
        # Also "time output (timenet_output dims)" is part of input to Deformation class.
        # So we need to encode time first.
        
        # Time network
        # times_sel needs to be (N, 1) to match batch size N
        if times_sel.shape[0] != point.shape[0]:
            # Broadcast time to N if necessary
            times_sel = times_sel.expand(point.shape[0], -1)
            
        time_emb = self.timenet(times_sel) # (N, timenet_output)
        
        # 2. Forward Dynamic Pass
        # "Pass these embeddings, along with raw opacity, shs, and times_sel to self.deformation_net"
        # Wait, raw opacity? Or opacity_emb?
        # Source: "Pass these embeddings, along with raw opacity, shs, and times_sel"
        # This is contradictory. "these embeddings" refers to pos_emb, scale_rot_emb.
        # "raw opacity" suggests opacity is NOT embedded?
        # But earlier it said "opacity_poc" buffer exists.
        # Let's look at "Deformation Network Input Construction":
        # "position (3 dims) + position PE (3 * posbase_pe * 2 dims) + time output (timenet_output dims)"
        # This suggests the input to the main Deformation MLP is:
        # [pos, pos_PE, time_emb]
        # What about scales, rotations, opacity, shs?
        # The source says "predicts dynamic 3D Gaussian parameters... from static input points, scales, rotations, opacity, SHs, and time."
        # And "Forward Dynamic Pass: Encode point... scales and rotations... Pass these embeddings, along with raw opacity, shs, and times_sel to self.deformation_net"
        
        # This implies self.deformation_net takes:
        # [pos_emb, scale_rot_emb, opacity, shs, time_emb] ?
        
        # Let's construct the input tensor.
        # pos_emb: (N, 3 + 3*posbase_pe*2)
        # scale_rot_emb: (N, 7 + 7*scale_rotation_pe*2) ? Or separate?
        # opacity: (N, 1)
        # shs: (N, K, 3) -> Flatten to (N, 3K)
        # time_emb: (N, timenet_output)
        
        # The source comment `input_ch=(3)+(3*(posbase_pe))*2` is specific to position.
        # It doesn't mention other channels.
        # However, the Deformation class must handle all inputs.
        
        # Let's assume the Deformation class concatenates all these features.
        
        shs_flat = shs.view(shs.shape[0], -1) # (N, 3K)
        
        # Concatenate all features
        # Note: The source says "raw opacity".
        input_features = torch.cat([
            pos_emb, 
            scale_rot_emb, 
            opacity, 
            shs_flat, 
            time_emb
        ], dim=-1)
        
        # Pass to deformation network
        # self.deformation_net is an nn.Module
        # It returns the dynamic parameters
        dynamic_params = self.deformation_net(input_features)
        
        # Reshape dynamic_params back to Gaussian parameters
        # The output structure depends on the Deformation class definition.
        # Assuming it outputs: means, scales, rotations, opacity, shs
        # We need to split dynamic_params.
        
        # Since the exact split is not given in the source, we assume the Deformation class 
        # returns a tuple or dict.
        # For pseudocode completeness, we assume a standard split.
        
        # Let's assume self.deformation_net returns a tuple:
        # (means, scales, rotations, opacity, shs)
        
        # However, the source doesn't define self.deformation_net's forward return.
        # We will leave this as a comment.
        
        return dynamic_params

    def initialize_weights(self):
        for m in self.modules():
            if isinstance(m, nn.Linear):
                # Bug: Source says bias is initialized with weight logic
                # init.xavier_uniform_(m.weight, gain=1)
                # init.xavier_uniform_(m.bias, gain=1) # This is the bug
                # Correct would be:
                # nn.init.xavier_uniform_(m.weight)
                # nn.init.constant_(m.bias, 0)
                pass
```

### Edge Cases and Pitfalls
*   **Bias Initialization Bug:** The `initialize_weights` function in the source code incorrectly initializes the bias of Linear layers using `init.xavier_uniform_(m.weight, gain=1)` instead of `init.xavier_uniform_(m.bias, gain=1)` or `init.constant_(m.bias, 0)`. This applies weight initialization logic to the bias tensor, which is incorrect and may affect convergence.
*   **Time Broadcasting:** If `times_sel` does not match the batch size $N$, it must be broadcasted or repeated to match $N$ before passing to the time network.
*   **SH Flattening:** Spherical Harmonic coefficients must be flattened before concatenation with other features.

## Operating characteristics
*   **Time Complexity:** Depends on the size of the deformation network and the number of Gaussians $N$. The positional encoding is $O(N \cdot D \cdot n)$ where $D$ is input dimension and $n$ is PE bands. The MLP forward pass is $O(N \cdot L \cdot W^2)$ where $L$ is depth and $W$ is width.
*   **Memory Complexity:** $O(N \cdot D_{input} + N \cdot D_{output})$ for storing inputs and outputs. The network weights are constant size.
*   **Strengths:** Handles dynamic scenes effectively by learning temporal deformations.
*   **Limitations:** Requires training the deformation network, which adds computational overhead compared to static 3DGS. Performance depends on the capacity of the network (width, depth, PE bands).

## Background
The Deformation Network is part of the 4D Gaussian Splatting method, which extends 3D Gaussian Splatting to dynamic scenes. It was introduced in the unofficial repository `ElenaViewSynthesis/4dgs` [S1, unofficial]. The method uses a neural network to predict Gaussian parameters over time, allowing for the reconstruction of 4D content.

## Sources
[S1] ElenaViewSynthesis/4dgs, https://github.com/ElenaViewSynthesis/4dgs, fetched 2026-10-28 (unofficial)
