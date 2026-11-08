# MLPRender_Fea_WeakView_Density

Also called: Feature-based MLP with weak view dependence and Gaussian positional encoding

## Overview
A neural rendering module that predicts RGB color and volume density ($\sigma$) separately using a feature-based MLP with weak view dependence and Gaussian positional encoding for features and view directions. It sets itself apart from standard NeRF MLPs by decoupling the density prediction from the view-dependent color prediction, allowing the density to be derived from a specific feature channel while maintaining view-independence for the volumetric properties.

## How it works
**Inputs and Outputs:**
- **Inputs:** `features` tensor of shape $[B, \text{num\_rays}, \text{inChanel}]$ and `viewdirs` tensor of shape $[B, \text{num\_rays}, 3]$.
- **Outputs:** `sigma` (volume density) and `rgb` (color). If `viewdirs` is None, `rgb` is None.

**Data Structures and Parameters:**
- `inChanel`: Input feature dimension.
- `featureC`: Internal feature dimension (default 128).
- `viewpe`: Number of frequency bands for view direction positional encoding (default 6).
- `feape`: Number of frequency bands for feature positional encoding (default 6).
- `fea_pe_progress`, `view_pe_progress`: Progress variables for positional encoding (typically 0 to 1).
- Internal channel sizes:
  - `self.in_mlpC = (2 * \text{feape} + 1) * \text{inChanel}`
  - `self.mid_mlpC = (2 * \text{viewpe}) * 3`
- Layers: `layer1`, `layer2`, `layer3`. `layer2` outputs `featureC + 1` dimensions. `layer3` outputs 3 dimensions (RGB).
- Bias initialization: `layer3.bias` is initialized to 0.

**Steps:**
1. **Feature Positional Encoding:** If `feape > 0`, concatenate `features` with the positional encoding of `features`.
   $$ \text{indata} = [\text{features}, \text{positional\_encoding}(\text{features}, \text{feape}, \text{fea\_pe\_progress})] $$
   The resulting `indata` has shape $[B, \text{num\_rays}, \text{self.in\_mlpC}]$.

2. **View Direction Positional Encoding:** If `viewpe > 0`, initialize `middata` with the positional encoding of `viewdirs`.
   $$ \text{middata} = \text{positional\_encoding}(\text{viewdirs}, \text{viewpe}, \text{view\_pe\_progress}) $$
   The resulting `middata` has shape $[B, \text{num\_rays}, \text{self.mid\_mlpC}]$.

3. **First MLP Layer:** Concatenate `indata` to form `mlp_in` (which is just `indata` in this context) and pass through `layer1`.
   $$ x = \text{layer1}(\text{mlp\_in}) $$
   Apply ReLU activation:
   $$ x = \text{ReLU}(x) $$

4. **Second MLP Layer and Split:** Pass `x` through `layer2`, which outputs `featureC + 1` dimensions. Split the output into `sigma_features` (the last channel) and `x` (the remaining `featureC` channels).
   $$ \sigma_{\text{features}}, x = x[..., -1], x[..., :-1] $$
   Apply ReLU activation to `x`:
   $$ x = \text{ReLU}(x) $$

5. **Density Output:** The volume density $\sigma$ is derived from `sigma_features`. Typically, this involves an exponential function to ensure positivity, though the source notes explicitly state `sigma_features` is returned. In standard implementations, $\sigma = \exp(\sigma_{\text{features}})$ or similar. The source notes say "return ... sigma_features", implying the raw output or a specific activation is used. Given the context of "volume density", it is often $\sigma = \text{softplus}(\sigma_{\text{features}})$ or $\exp$. However, strictly following the source note "return ... sigma_features", we note that `sigma_features` is the direct output of the split. *Note: Standard NeRF uses $\sigma = \exp(\sigma_{\text{raw}})$. If the source does not specify, this is a gap.* [G] Assuming standard practice for density positivity if not explicitly defined as linear: $\sigma = \exp(\sigma_{\text{features}})$.

6. **Color Output:**
   - If `viewdirs` is not None:
     - Concatenate `x` with `middata` to form `mlp_mid`.
     - Pass `mlp_mid` through `layer3` to get RGB values.
     - Apply sigmoid activation to clamp RGB to $[0, 1]$.
     $$ \text{rgb} = \sigma(\text{layer3}(\text{mlp\_mid})) $$
   - If `viewdirs` is None:
     - Return `None` for RGB and `sigma_features` (or processed density) for density.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

def positional_encoding(x, num_freq_bands, progress=1.0):
    """
    Gaussian positional encoding as implied by the source description.
    Note: Standard NeRF uses sin/cos. The source explicitly mentions "Gaussian positional encoding".
    However, the variable names `viewpe` and `feape` and the structure `(2*pe+1)` strongly suggest
    the standard sin/cos positional encoding used in NeRF, where the factor 2 comes from sin and cos.
    "Gaussian" might be a misnomer in the source or refer to the envelope.
    Given the formula `(2*feape + 1)`, this matches the standard NeRF PE: [x, sin(2^0 pi x), cos(2^0 pi x), ...].
    We implement standard NeRF PE as it fits the dimensionality constraints exactly.
    """
    if num_freq_bands == 0:
        return x
    
    # Standard NeRF positional encoding
    # x shape: [..., D]
    # Output shape: [..., D * (2 * num_freq_bands + 1)]
    # The first D channels are the original x.
    # The next 2*num_freq_bands channels are sin/cos of scaled x.
    
    # Scale factors
    scales = torch.tensor([2**i for i in range(num_freq_bands)], dtype=x.dtype, device=x.device)
    
    # Reshape for broadcasting
    x = x.unsqueeze(-1) # [..., D, 1]
    scales = scales.view(1, 1, -1) # [1, 1, num_freq_bands]
    
    # Apply scaling
    x_scaled = x * scales * torch.pi # [..., D, num_freq_bands]
    
    # Compute sin and cos
    sin_vals = torch.sin(x_scaled)
    cos_vals = torch.cos(x_scaled)
    
    # Concatenate
    # Original x, then sin, then cos for each frequency
    # Shape: [..., D, 1 + 2*num_freq_bands]
    pe = torch.cat([x, sin_vals, cos_vals], dim=-1)
    
    # Reshape back to [..., D * (1 + 2*num_freq_bands)]
    pe = pe.view(*x.shape[:-1], -1)
    
    return pe

class MLPRender_Fea_WeakView_Density(nn.Module):
    def __init__(self, inChanel, featureC=128, viewpe=6, feape=6):
        super().__init__()
        self.inChanel = inChanel
        self.featureC = featureC
        self.viewpe = viewpe
        self.feape = feape
        
        self.in_mlpC = (2 * feape + 1) * inChanel
        self.mid_mlpC = (2 * viewpe) * 3
        
        # Layer 1: in_mlpC -> featureC
        self.layer1 = nn.Linear(self.in_mlpC, featureC)
        
        # Layer 2: featureC -> featureC + 1
        self.layer2 = nn.Linear(featureC, featureC + 1)
        
        # Layer 3: featureC + mid_mlpC -> 3 (RGB)
        self.layer3 = nn.Linear(featureC + self.mid_mlpC, 3)
        
        # Initialize bias of layer3 to 0
        torch.nn.init.constant_(self.layer3.bias, 0)

    def forward(self, features, viewdirs, fea_pe_progress=1.0, view_pe_progress=1.0):
        """
        features: [B, num_rays, inChanel]
        viewdirs: [B, num_rays, 3]
        """
        # Step 1: Feature Positional Encoding
        indata = features
        if self.feape > 0:
            pe_features = positional_encoding(features, self.feape, fea_pe_progress)
            indata = torch.cat([indata, pe_features], dim=-1)
        
        # Step 2: View Direction Positional Encoding
        middata = None
        if self.viewpe > 0:
            middata = positional_encoding(viewdirs, self.viewpe, view_pe_progress)
        
        # Step 3: First MLP Layer
        x = self.layer1(indata)
        x = F.relu(x, inplace=True)
        
        # Step 4: Second MLP Layer and Split
        x = self.layer2(x)
        sigma_features = x[..., -1]
        x = x[..., :-1]
        x = F.relu(x, inplace=True)
        
        # Step 5: Density Output
        # Note: Source says return sigma_features. Standard practice is exp or softplus.
        # We return sigma_features as per source note, but note that density is usually positive.
        sigma = sigma_features 
        
        # Step 6: Color Output
        if viewdirs is not None:
            # Concatenate x with middata
            mlp_mid = torch.cat([x, middata], dim=-1)
            rgb = self.layer3(mlp_mid)
            rgb = torch.sigmoid(rgb)
            return sigma, rgb
        else:
            return sigma, None
```

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of rays and features. The MLP layers are fully connected, so complexity is $O(B \cdot \text{num\_rays} \cdot (\text{in\_mlpC} \cdot \text{featureC} + \text{featureC}^2 + \text{featureC} \cdot \text{mid\_mlpC}))$.
- **Memory Complexity:** Linear with respect to the number of rays and features.
- **Strengths:** Decouples density and color, allowing for view-independent density which can be beneficial for certain reconstruction tasks. Weak view dependence allows for efficient rendering.
- **Limitations:** The separation of density and color may not capture complex view-dependent effects as well as fully view-dependent models.
- **When to use:** When view-independent density is desired, or when computational efficiency is prioritized over extreme view-dependent detail.

## Background
This method is part of the NeRF MLP Architecture family, specifically focusing on feature-based representations with weak view dependence. It is used in neural rendering to predict volumetric properties.

## Sources
[S1] Nemo1999/Joint-TensoRF, https://github.com/Nemo1999/Joint-TensoRF, fetched 2026-10-27 (unofficial)
