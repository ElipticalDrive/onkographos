# Gaussian Pruning (Monocular)
Also called: Reliable Observation Filtering

## Overview
Gaussian Pruning (Monocular) is a preprocessing step for 3D Gaussian Splatting initialization in monocular SLAM that filters out unreliable initial Gaussian centers based on multi-view consistency checks [S1, S2, unofficial]. It prevents incorrect initial geometry from degrading camera tracking performance by ensuring only Gaussians with valid, consistent observations in the training views are retained [S1, unofficial]. Unlike sibling methods that focus on replication or allocation density, this method focuses on quality control of the initial point cloud [S1, unofficial].

## How it works
**Inputs:**
- Initialized set of $N$ 3D Gaussians with centers $c_0^i$ for $i \in \{1, \dots, N\}$ [S2, unofficial].
- Camera calibration parameters (intrinsic matrix $K$, extrinsic matrices $T_{v}$) for $V$ training views [S2, unofficial].
- Depth buffer (z-buffer) data for each training view, derived from the initialization point cloud [S2, unofficial].
- Foreground masks for each training view [S2, unofficial].

**Outputs:**
- A subset of Gaussians $\mathcal{G}_{valid} \subseteq \{1, \dots, N\}$ that pass all filtering criteria [S2, unofficial].

**Data Structures:**
- Boolean indicator $r_{i,v}$ for each Gaussian $i$ and view $v$, where $r_{i,v}=1$ if the observation is reliable, 0 otherwise [S2, unofficial].
- Set $\mathcal{R} = \{ i \mid \exists v, r_{i,v}=1 \}$ (or similar aggregation logic, though the source defines the sample set based on the indicator) [S2, unofficial].

**Steps:**
1. **Projection:** For each Gaussian $i$ and each training view $v$, project the initialized center $c_0^i$ into the image plane of view $v$ using the camera calibration parameters [S2, unofficial].
   - Compute homogeneous coordinates: $p_{h} = K \cdot T_{v} \cdot c_0^i$ (assuming $c_0^i$ is in homogeneous coordinates or appropriately transformed) [S2, unofficial].
   - Apply perspective division to obtain 2D image coordinates $(u_{i,v}, v_{i,v})$ [S2, unofficial].
   - Formula: $(u_{i,v}, v_{i,v}) = \text{persp\_div}(p_{h})$ [S2, unofficial].

2. **Visibility Check:** Determine the projected visibility of Gaussian $i$ in view $v$ by performing z-buffer rasterization of the initialization point cloud [S2, unofficial]. Let $vis_{i,v}$ be a boolean indicating if the Gaussian is visible (not occluded by other initialized points) in view $v$ [S2, unofficial].

3. **Filtering Criteria:** For each Gaussian $i$ and view $v$, evaluate four binary criteria [S2, unofficial]:
   - **Range ($rng_{i,v}$):** Is $(u_{i,v}, v_{i,v})$ inside the valid image domain (e.g., $0 \le u < W, 0 \le v < H$)? [S2, unofficial].
   - **Foreground ($fg_{i,v}$):** Does the pixel $(u_{i,v}, v_{i,v})$ belong to the foreground mask of view $v$? [S2, unofficial].
   - **Visibility ($vis_{i,v}$):** Is the Gaussian visible according to the z-buffer rasterization? [S2, unofficial].
   - **Depth Consistency ($dep_{i,v}$):** Is the projected depth $z_{proj}$ consistent with the z-buffer depth $z_{buf}$ at $(u_{i,v}, v_{i,v})$? [S2, unofficial].
     - Condition: $\frac{|z_{proj} - z_{buf}|}{z_{buf}} \le \tau$, where $\tau = 0.05$ (5%) [S2, unofficial].

4. **Reliable Observation Indicator:** Compute the indicator $r_{i,v}$ for Gaussian $i$ in view $v$ [S2, unofficial]:
   $$ r_{i,v} = rng_{i,v} \land fg_{i,v} \land vis_{i,v} \land dep_{i,v} $$
   where $\land$ denotes logical AND [S2, unofficial].

5. **Pruning:** A Gaussian $i$ is considered to have a reliable observation if $r_{i,v}=1$ for at least one view $v$ (or potentially all views, depending on specific implementation details not fully explicit, but the source implies inclusion in a "reliable sample set" based on the indicator) [S2, unofficial]. The source defines the reliable sample set as $\mathcal{R} = \{ (i, v) \mid r_{i,v}=1 \}$ [S2, unofficial]. Gaussians not contributing to any reliable observation (or failing a global threshold if applied) are pruned [S1, unofficial].

**Parameters:**
- Depth consistency threshold $\tau = 5\%$ [S2, unofficial].

**Pitfalls:**
- If the filtering logic depends on the current rendering state (e.g., changing Gaussian centers or parameters during training), it reintroduces interference and instability [S2, unofficial].
- **Solution:** Perform filtering in advance using only the initialized centers and training data, and keep the filtering results fixed throughout the training process [S2, unofficial].

## Subtopics

### Reliable Observation Filtering
This is the core mechanism of Gaussian Pruning (Monocular). It refers specifically to the sequential application of the four binary criteria (range, foreground, visibility, depth consistency) to determine the reliability of each Gaussian's observation in each view [S2, unofficial]. It sets itself apart by explicitly addressing the issue of unreliable observations due to occlusion, mask-boundary quantization, exposure differences, and background mixing, which can dominate gradients in texture-degraded scenes [S2, unofficial]. Choose this when initializing Gaussians for monocular SLAM to ensure robust camera tracking by preventing incorrect geometry from random initialization [S1, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N \cdot V)$, where $N$ is the number of Gaussians and $V$ is the number of training views, due to projection and z-buffer checks for each Gaussian-view pair [S2, unofficial].
- **Memory Complexity:** $O(N \cdot V)$ to store the reliability indicators $r_{i,v}$ [S2, unofficial].
- **Measured Performance:** On the TUM RGB-D dataset (Monocular input), Gaussian pruning reduces the average camera tracking error (ATE RMSE) from 46.6 cm (without pruning) to 3.96 cm across sequences f1/desk, f2/xyz, and f3/office [S1, unofficial].
- **Strengths:** Significantly enhances camera tracking performance in monocular SLAM by preventing incorrect initial geometry [S1, unofficial]. Stabilizes training by fixing the reliable observation set [S2, unofficial].
- **Limitations:** Depends on the quality of the initial point cloud and camera calibration. May prune valid Gaussians if the initial geometry is poor or if depth consistency threshold is too strict [S2, unofficial].
- **When to use:** Monocular SLAM settings with 3D Gaussian Splatting initialization [S1, unofficial].
- **When not to use:** RGB-D inputs where depth information is more reliable and direct, or when computational resources for pre-filtering are extremely limited (though the cost is generally low) [S1, S2, unofficial].

## Background
Gaussian Pruning (Monocular) was introduced in the context of Gaussian Splatting SLAM to address the challenges of initializing 3D Gaussians from monocular input [S1, unofficial]. The method was proposed to mitigate the issue of randomly initialized Gaussians persisting in 3D space and leading to incorrect initial geometry for other views, which degrades camera tracking [S1, unofficial]. The specific filtering criteria and the concept of "reliable observation filtering" were detailed in research on visibility-aware texture consensus for 3D Gaussian reconstruction in texture-degraded scenes [S2, unofficial].

## Sources
[S1] Gaussian Splatting SLAM, https://arxiv.org/abs/2312.06741, fetched 2026-10-27 (unofficial)
[S2] Visibility-Aware Texture Consensus and Local Structural Constraints for 3D Gaussian Reconstruction in Texture- Degraded Scenes, https://doi.org/10.21203/rs.3.rs-10727268/v1, fetched 2026-10-26 (unofficial)
