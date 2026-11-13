# Angle Feature Generation
## Overview
Angle Feature Generation creates features based on the angular relationship between rays and camera positions for multi-view stereo rendering [S1, unofficial]. It builds on [Ray Angle Computation](3156-ray-angle-computation.md) by converting angular differences into explicit feature vectors that encode viewing geometry [S1, unofficial]. Unlike Direction Feature Generation, which relies on coordinate transformation, this method explicitly computes the dot product between ray directions to capture angular similarity [S1, unofficial].
## How it works
**Inputs and Outputs:**
- Input: Reference ray direction, ray directions from source/training cameras, and camera positions for direction computation [S1, unofficial].
- Output: A feature vector containing the computed angle features (dot products) [S1, unofficial].

**Data Structures:**
- Feature vector: A 1D array or tensor where each element corresponds to a computed angular metric for a specific source view [S1, unofficial].
- Ray directions: 3D unit vectors representing the direction of the ray [S1, unofficial].

**Steps:**
1. **Compute Directions:** Determine the direction vectors from the camera centers to the ray points [S1, unofficial]. For a ray originating at camera center $\mathbf{C}$ and passing through point $\mathbf{P}$, the direction is $\mathbf{d} = \frac{\mathbf{P} - \mathbf{C}}{\|\mathbf{P} - \mathbf{C}\|}$ [S1, unofficial].
2. **Compute Dot Products:** Calculate the dot product between the reference ray direction $\mathbf{d}_{\text{ref}}$ and each source ray direction $\mathbf{d}_{\text{src}}$ [S1, unofficial]. The angle feature $f$ is given by:
   $$ f = \mathbf{d}_{\text{ref}} \cdot \mathbf{d}_{\text{src}} $$
   This value represents the cosine of the angle between the two rays [S1, unofficial].
3. **Aggregate Features:** Collect these dot products for all source views to form the final angle feature vector [S1, unofficial].

**Parameters:**
- Number of source views: Determines the length of the output feature vector [S1, unofficial].
- Normalization: Directions are normalized to unit length before dot product computation [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Zero-length rays:** If the point coincides with a camera center, the direction is undefined; guard against division by zero [G].
- **Precision:** Dot products are sensitive to floating-point precision; ensure directions are normalized [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch

def gen_angle_feature(ref_cam_pos, src_cam_pos, point):
    """
    Compute angle features based on dot products between ray directions.
    
    Args:
        ref_cam_pos: (3,) tensor, reference camera position in world space
        src_cam_pos: (N, 3) tensor, source camera positions in world space
        point: (3,) tensor, 3D point in world space
        
    Returns:
        features: (N,) tensor, angle features (dot products)
    """
    # Step 1: Compute reference ray direction
    ref_dir = point - ref_cam_pos
    ref_dir = ref_dir / (torch.norm(ref_dir) + 1e-8)
    
    features = []
    for i in range(len(src_cam_pos)):
        # Step 1: Compute source ray direction
        src_dir = point - src_cam_pos[i]
        src_dir = src_dir / (torch.norm(src_dir) + 1e-8)
        
        # Step 2: Compute dot product
        dot_product = torch.dot(ref_dir, src_dir)
        features.append(dot_product)
        
    return torch.stack(features)
```
## Subtopics

### Direction Feature Generation
Generates features based on the direction of rays in the reference camera coordinate system [S1, unofficial]. It sets itself apart by transforming ray directions using the world-to-camera rotation matrix rather than computing dot products between multiple rays [S1, unofficial]. Choose this method when explicit angular similarity metrics are less important than absolute directional encoding in the reference frame [S1, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
- Time: Not given in the sources.
- Memory: Not given in the sources.

**Measured Performance:**
- Not given in the sources.

**Scaling:**
- Not given in the sources.

**Strengths and Limitations:**
- Strengths: Not given in the sources.
- Limitations: Not given in the sources.

**When to Use:**
- Not given in the sources.
## Background
Originated in the context of multi-view stereo rendering [S1, unofficial]. It is used to enhance geometric understanding in neural rendering by providing explicit angular features [S1, unofficial].
## Sources
[S1] Xharlie/pointnerf, https://github.com/Xharlie/pointnerf, fetched 2026-10-26 (unofficial)
