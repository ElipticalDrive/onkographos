# Gaussian Decoding from Scene Tokens

Also called: Scene Token Fusion and Gaussian Decoding, Scene Token Gaussian Decoding

## Overview
Gaussian Decoding from Scene Tokens is a method that maps learned scene tokens into a set of freely placed 3D Gaussian primitives, allowing each token to represent a surface region with variable spatial extent and complexity [S1, unofficial]. It differs from standard Gaussian Splatting by decoupling Gaussian center prediction from fixed ray intersections, enabling the decoder to place Gaussians anywhere in 3D space rather than anchoring them to specific rays [S1, unofficial]. This approach builds on [3D Gaussian Splatting with Transformer-based Parameterization](6072-3d-gaussian-splatting-with-transformer-based-parameterizatio.md) by introducing a fusion stage that integrates geometry and appearance information before decoding.

## How it works
**Inputs and Outputs:**
*   **Inputs:** Scene tokens $\stok_k$ (derived from fused geometry and visual features), color embeddings, and camera parameters.
*   **Outputs:** A set of $G$ 3D Gaussians per scene token, each defined by 23 parameters.

**Data Structures:**
*   **Scene Token ($\stok_k$):** A vector representation for the $k$-th scene region.
*   **Geometry Tokens:** Derived from backbone features, serving as keys in cross-attention.
*   **Visual Tokens:** Layer-specific features serving as values in cross-attention.
*   **Color Embedding:** Appearance information used in the final residual cross-attention.
*   **Gaussian Parameters:** Each Gaussian $j$ in token $k$ has:
    *   $\gmean_{kj} \in \mathbb{R}^3$: Center.
    *   $\gscale_{kj} \in \mathbb{R}^3$: Scale.
    *   $\grot_{kj} \in \mathbb{R}^4$: Rotation (unit quaternion).
    *   $\gopac_{kj} \in \mathbb{R}$: Opacity.
    *   $\gcolor_{kj}$: Color represented by degree-1 spherical harmonics.

**Numbered Steps:**
1.  **Token Initialization:** Scene tokens start from a shared template with a forward bias of $z=0.5$ in the reference frame [S1, unofficial].
2.  **Fusion:** Perform fusion using three blocks. Each block consists of:
    *   **Cross-Attention:** Queries are scene tokens; keys are geometry tokens; values are visual tokens from one backbone layer (deepest first) [S1, unofficial].
    *   **Self-Attention:** Among scene tokens [S1, unofficial].
    *   *Parameters:* 24 attention heads, MLP ratio 4, GELU activation, residual gates initialized to 0.01 [S1, unofficial].
3.  **Appearance Injection:** Apply a final residual cross-attention with the color embedding as values to add appearance information to the scene tokens [S1, unofficial].
4.  **Decoding:** Decode each scene token into $G$ Gaussians using a two-layer MLP [S1, unofficial].
5.  **Parameter Constraints:** Apply squared-hinge penalties to keep scales in $[10^{-6}, 0.1]$, opacities above 0.01, and positions within a radius of 10 [S1, unofficial].

**Formula:**
The set of Gaussians $\mathcal{G}_k$ for token $k$ is decoded as:
$$
\mathcal {G}_k = \left \{ \left ( \gmean _{kj}, \gscale _{kj}, \grot _{kj}, \gopac _{kj}, \gcolor _{kj} \right ) \right \}_{j=1}^{\ngs } = D(\stok _k)
$$
where $D$ is the MLP decoder, $\ngs = G$ is the number of Gaussians per token, and the full scene is assembled from all $K$ tokens [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def gaussian_decode_from_scene_tokens(scene_tokens, geometry_tokens, visual_tokens_layers, color_embedding):
    """
    Decodes scene tokens into 3D Gaussians.
    
    Args:
        scene_tokens: Tensor of shape (K, D_token)
        geometry_tokens: Tensor of shape (N_geo, D_geo)
        visual_tokens_layers: List of Tensors, one per layer, deepest first. 
                              Each shape (N_vis, D_vis)
        color_embedding: Tensor of shape (N_color, D_color)
        
    Returns:
        gaussians: List of K lists, each containing G Gaussian parameter sets.
    """
    K, D_token = scene_tokens.shape
    G = 32  # Number of Gaussians per token
    
    # Step 1: Fusion Blocks
    current_tokens = scene_tokens
    for i in range(3):
        # Cross-Attention: Queries=current_tokens, Keys=geometry_tokens, Values=visual_tokens_layers[i]
        # Using 24 heads, MLP ratio 4, GELU, residual gates init 0.01
        cross_attn_out = cross_attention(
            query=current_tokens,
            key=geometry_tokens,
            value=visual_tokens_layers[i],
            num_heads=24,
            mlp_ratio=4,
            activation='gelu',
            residual_gate_init=0.01
        )
        current_tokens = current_tokens + cross_attn_out  # Residual connection
        
        # Self-Attention among scene tokens
        self_attn_out = self_attention(
            query=current_tokens,
            key=current_tokens,
            value=current_tokens,
            num_heads=24,
            mlp_ratio=4,
            activation='gelu',
            residual_gate_init=0.01
        )
        current_tokens = current_tokens + self_attn_out  # Residual connection
        
    # Step 2: Appearance Injection
    # Final residual cross-attention with color embedding as values
    appearance_attn_out = cross_attention(
        query=current_tokens,
        key=geometry_tokens, # Or geometry tokens as per standard practice, source says "residual cross-attention with color embedding as values"
        value=color_embedding,
        num_heads=24,
        mlp_ratio=4,
        activation='gelu',
        residual_gate_init=0.01
    )
    current_tokens = current_tokens + appearance_attn_out
    
    # Step 3: Decode to Gaussians
    gaussians = []
    # Two-layer MLP decoder D
    decoder = TwoLayerMLP(input_dim=D_token, output_dim=G * 23)
    
    for k in range(K):
        token = current_tokens[k]
        params = decoder(token) # Shape (G * 23,)
        token_gaussians = []
        for j in range(G):
            # Extract 23 parameters
            offset = j * 23
            g_mean = params[offset : offset+3]
            g_scale = params[offset+3 : offset+6]
            g_rot = params[offset+6 : offset+10]
            g_opac = params[offset+10]
            g_color = params[offset+11 : offset+23] # Degree-1 SH
            
            # Apply constraints [G]
            # Squared-hinge penalties are applied during training loss, 
            # but for inference, we assume valid parameters.
            # If implementing training, apply penalties to gradients.
            
            token_gaussians.append({
                'mean': g_mean,
                'scale': g_scale,
                'rot': g_rot,
                'opacity': g_opac,
                'color': g_color
            })
        gaussians.append(token_gaussians)
        
    return gaussians
```

**Parameters:**
*   $G=32$: Number of Gaussians per scene token [S1, unofficial].
*   Degree-1 spherical harmonics: Used for color representation [S1, unofficial].
*   Cross-attention: 24 heads, MLP ratio 4, GELU activation, residual gates initialized to 0.01 [S1, unofficial].
*   Reference frame: All geometry is expressed in the coordinate frame of the first input camera [S1, unofficial].
*   Scene scale: Normalized so that the median distance of input depth points to the origin is one [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Free Placement:** Nothing constrains decoded Gaussians to stay near their source patch; the decoder can place them anywhere in 3D [S1, unofficial]. This requires auxiliary losses (e.g., Cone loss) to ground Gaussians in their source patch [S1, unofficial].
*   **Precision:** Scales are constrained to $[10^{-6}, 0.1]$ and opacities above 0.01 to prevent numerical instability [S1, unofficial].
*   **Boundaries:** Positions are constrained within a radius of 10 [S1, unofficial].

## Subtopics

### [Depth and Gaussian Decoding](18077-depth-and-gaussian-decoding.md)
A variant that predicts depth from features to map pixel-aligned predictions to 3D space, decoupling Gaussian center prediction from fixed ray intersections. It differs by inferring geometry directly from latent features without relying on encoder-side depth maps. Choose this when explicit 3D geometry reconstruction from feature streams is required without external depth inputs.

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** On DL3DV at 6 views, $G=32$ gains 0.34 to 0.36 dB PSNR over $G=8$, while $G=64$ adds at most 0.08 dB at twice the Gaussians [S1, unofficial].
*   **Scaling:** The number of Gaussians per token can be varied ($G=8, 16, 32, 64$), with diminishing returns for higher $G$ [S1, unofficial].
*   **Strengths:** Allows tokens to spread Gaussians over observed regions with variable complexity; free placement offers flexibility.
*   **Limitations:** Requires auxiliary losses (Cone loss) to prevent Gaussians from drifting away from their source patch; no guarantee of association between token and observation [S1, unofficial].
*   **When to Use:** When flexible, region-based Gaussian representation is needed and computational budget allows for transformer-based fusion.
*   **When Not to Use:** When strict ray-anchored Gaussian placement is required or when auxiliary losses cannot be applied.

## Background
Originated in ZipSplat (Lin et al., 2026), which builds on DA3-Giant. The method uses features from three backbone layers merged into geometry tokens, fused into scene tokens, and decoded into Gaussians [S1, unofficial]. It is trained on RealEstate10K and DL3DV datasets [S1, unofficial].

## Sources
[S1] ZipSplat: Fewer Gaussians, Better Splats, https://doi.org/10.48550/arXiv.2606.05102, fetched 2026-10-29 (unofficial)
