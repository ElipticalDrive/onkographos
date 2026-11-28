# Appearance Feature Initialization and Training Schedule

Also called: Appearance feature initialization, Training schedule with appearance features

## Overview
This method defines a staged training protocol for 3D Gaussian Splatting that introduces appearance features after an initial period of geometry optimization, followed by simplification and quantization [S1]. It decouples appearance parameter optimization from the initial scale and rotation learning phases [S1]. Unlike standard Gaussian Splatting which optimizes all parameters simultaneously, this approach introduces static and view-dependent appearance features at iteration 15K, simplifies the primitive count at 20K, and applies Sub-Vector Quantization (SVQ) in the final 1K iterations [S1].

## How it works
**Inputs and Outputs:**
- Input: Initial 3D Gaussian primitives (position, covariance/scale, rotation, opacity, and initial SH coefficients) [S1].
- Output: Optimized Gaussian primitives with quantized appearance features [S1].

**Data Structures:**
The Gaussian primitive stores appearance features as a vector of dimension $D=3$ [S1].

**Parameters:**
- Total iterations: 30,000 [S1].
- Appearance introduction iteration: 15,000 [S1].
- Simplification iteration: 20,000 [S1].
- SVQ start iteration: 29,000 [S1].
- Local distinctiveness factor ($\lambda$): 2 [S1].
- Appearance feature dimension ($D$): 3 [S1].

**Steps:**

1. **Initial Training (Iterations 0–14,999):**
   Train scale and rotation parameters from the initial training iteration [S1]. Update spherical harmonics (SH) coefficients, specifically the DC term, which serves as the basis for later initialization [S1].

2. **Appearance Feature Introduction (Iteration 15,000):**
   Introduce appearance features [S1]. Initialize static appearance features using the spherical harmonics DC coefficients trained up to iteration 15,000 [S1]. Initialize view-dependent appearance features as zero vectors [S1].

3. **Simplification (Iteration 20,000):**
   Run the simplification process incorporating local distinctiveness (LD) scoring with factor $\lambda=2$ [S1].

4. **Sub-Vector Quantization (Iterations 29,000–30,000):**
   Apply SVQ to per-Gaussian features using fixed K-means indices and optimized codebooks [S1]. K-means clustering is performed once to assign indices, which remain fixed [S1]. Only the codebooks are optimized for the remaining 1,000 iterations [S1].
   - Scale: length 1, 26 codes for each sub-vector [S1].
   - Rotation: length 2, 29 codes for each sub-vector [S1].
   - Appearance features: length 2, 210 codes for each sub-vector [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def train_gaussian_avatar(total_iterations=30000, appearance_start=15000, simplification_iter=20000, svq_start=29000):
    # Initialize Gaussians with position, scale, rotation, opacity, SH coefficients
    gaussians = initialize_gaussians() # Sources do not specify initialization routine
    
    for iter in range(total_iterations):
        if iter < appearance_start:
            # Optimize scale and rotation only
            optimize_scale_rotation(gaussians) # Sources do not specify optimizer details
            # Update SH coefficients, tracking DC term for later initialization
            update_sh_dc(gaussians) # Sources do not specify SH update routine
            
        elif iter == appearance_start:
            # Initialize static appearance features from trained SH DC coefficients
            gaussians.static_appearance = gaussians.sh_dc_coefficients
            # Initialize view-dependent appearance features as zero vectors
            gaussians.view_dependent_appearance = [0.0] * len(gaussians.position)
            
        elif iter == simplification_iter:
            # Simplify primitives using Local Distinctiveness scoring with lambda=2
            # Sources do not specify the LD scoring formula or simplification routine
            gaussians = simplify_gaussians(gaussians, lambda_factor=2)
            
        elif iter >= svq_start:
            # Apply SVQ with fixed indices and optimized codebooks
            # Scale: length 1, 26 codes; Rotation: length 2, 29 codes; Appearance: length 2, 210 codes
            # K-means indices are fixed; only codebooks are optimized
            # Sources do not specify SVQ optimization routine
            optimize_svq_codebooks(gaussians, fixed_indices=True)
            
        # Standard rendering and loss computation for all iterations
        # Sources do not specify rendering or loss computation routines
        render_image(gaussians)
        compute_loss()
        update_gradients(gaussians)

def initialize_gaussians(): pass # Sources do not specify initialization routine
def optimize_scale_rotation(g): pass # Sources do not specify optimizer details
def update_sh_dc(g): pass # Sources do not specify SH update routine
def simplify_gaussians(g, lambda_factor): pass # Sources do not specify the LD scoring formula or simplification routine
def optimize_svq_codebooks(g, fixed_indices): pass # Sources do not specify SVQ optimization routine
def render_image(g): pass # Sources do not specify rendering routine
def compute_loss(): pass # Sources do not specify loss computation routine
def update_gradients(g): pass # Sources do not specify gradient update routine
```

**Edge Cases and Pitfalls:**
- Boundaries: The transition at iteration 15,000 requires that SH DC coefficients are fully trained up to that point to provide a meaningful initialization for static appearance features [S1].

## Subtopics

### [Appearance Embedding](14693-appearance-embedding.md)
Associates each training image with a unique embedding vector to model appearance changes such as lighting and shadows. Choose it when rendering requires per-view appearance variation beyond view-dependent SH terms.

## Operating characteristics
- **Time Complexity:** The training runs for 30,000 iterations [S1]. The simplification step at 20,000 iterations incorporates local distinctiveness scoring [S1]. The SVQ step in the last 1,000 iterations optimizes codebooks with fixed K-means indices [S1].
- **Memory Complexity:** The introduction of appearance features adds a vector of dimension $D=3$ per Gaussian [S1]. The SVQ step quantizes features using specific codebook sizes: 26 codes for scale, 29 for rotation, and 210 for appearance features [S1].
- **Performance:** Experiments were conducted on an NVIDIA RTX 4090 using the Mini-Splatting framework [S1].
- **Strengths:** Decoupling appearance from geometry optimization stabilizes training [S1]. Fixed K-means indices in SVQ reduce search space during codebook optimization [S1].
- **Limitations:** The fixed K-means indices in SVQ require a single upfront clustering pass [S1].

## Background
This method is part of the "Optimized Minimal 3D Gaussian Splatting" framework, implemented within the Mini-Splatting library [S1]. It builds on the standard 3D Gaussian Splatting rendering pipeline but introduces a specific training schedule to handle appearance features and quantization efficiently [S1].

## Sources
[S1] Optimized Minimal 3D Gaussian Splatting, https://doi.org/10.52202/085713-4534, fetched 2026-10-30
