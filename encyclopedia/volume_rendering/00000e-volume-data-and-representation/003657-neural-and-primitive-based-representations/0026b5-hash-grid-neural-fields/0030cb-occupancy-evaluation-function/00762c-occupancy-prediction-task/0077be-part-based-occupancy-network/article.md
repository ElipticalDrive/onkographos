# Part-based Occupancy Network

Also called: NASA, HALO-VAE

## Overview
Part-based Occupancy Networks represent 3D shapes, particularly human bodies, by decomposing them into multiple bone-hinged parts, each with its own local occupancy field [S1]. Unlike monolithic implicit fields, this method learns rigid bone transformations and local non-rigid deformations separately, allowing for precise articulation and deformation [S1]. It distinguishes itself from dynamic extensions like [O-Flow](23428-occupancy-prediction-task.md) by maintaining a static canonical representation per part rather than learning continuous deformation gradients over time [S1].

## How it works
The method constructs a composite occupancy field by combining outputs from individual part-based networks. It handles both static articulated bodies (e.g., humans) and interacting objects (e.g., hand-object grasps) [S1].

**Inputs and Outputs:**
*   **Inputs:** Pose parameters (e.g., joint angles), bone codes (latent representations for each bone), and query points in 3D space. For hand-object interactions, inputs also include point cloud representations of the hand and object models [S1].
*   **Outputs:** A scalar occupancy value (or signed distance) indicating whether a query point is inside or outside the shape, and potentially the specific bone responsible for that occupancy [S1].

**Data Structures:**
*   **Bone Codes:** A latent vector associated with each bone part, learned during training [S1].
*   **Rigid Network:** A neural network that takes pose parameters and bone codes to predict rigid transformation matrices (rotation and translation) for each bone [S1].
*   **Non-Rigid Deformation Network:** A neural network equipped with a linear subspace projection operator for each bone. It takes the pose and bone code to predict local non-rigid deformation corrections [S1].
*   **Occupancy Field:** A function $f_b(\mathbf{p})$ for each bone $b$, mapping a query point $\mathbf{p}$ to an occupancy value [S1].

**Steps:**
1.  **Rigid Transformation:** For each bone $b$, the rigid network predicts a transformation $T_b$ based on the global pose parameters and the bone's latent code. This transforms the query point $\mathbf{p}$ from world space to the bone's local canonical space $\mathbf{p}'_b = T_b^{-1} \mathbf{p}$ [S1].
2.  **Non-Rigid Deformation:** The pose is projected into a low-dimensional subspace using a learned linear projection operator specific to bone $b$. This subspace vector serves as local deformation information. The non-rigid deformation network uses this information and the bone code to predict a deformation offset $\Delta \mathbf{p}_b$ for the query point in the canonical space [S1].
3.  **Local Occupancy Evaluation:** The deformed point $\mathbf{p}''_b = \mathbf{p}'_b + \Delta \mathbf{p}_b$ is passed to the bone-specific occupancy network to predict the local occupancy value $o_b = f_b(\mathbf{p}''_b)$ [S1].
4.  **Global Combination:** The final occupancy value $O(\mathbf{p})$ for the query point is determined by applying a maximum operator across all bone occupancy values:
    $$ O(\mathbf{p}) = \max_{b \in \text{Bones}} o_b $$
    This assumes the union of the parts defines the overall shape [S1].

**For Hand-Object Interaction (HALO-VAE variant):**
1.  Encode the hand point cloud and object point cloud into separate latent codes using occupancy network encoders [S1].
2.  Predict the distance from query points to both the hand and the object [S1].
3.  Predict hand decomposition (bone assignments) [S1].
4.  Apply physical constraints to prevent penetration between the decoded hand and object geometries [S1].

**Parameters:**
*   **Pose Parameters:** Typically SMPL or MANO parameters for human bodies [S1].
*   **Latent Dimension:** Not given in the sources.
*   **Subspace Dimension:** Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Penetration:** In hand-object scenarios, without explicit physical constraints, the decoded hand and object may interpenetrate [S1].
*   **Boundary Handling:** The maximum operator assumes a union; care must be taken if intersection or subtraction is required [G].
*   **Degenerate Input:** If pose parameters are invalid or missing, rigid transformations may be undefined [G].

## Subtopics

### O-Flow
O-Flow is an implicit neural representation (INR) method for dynamic human modeling that extends O-Net [S1]. It generates the body shape for the first frame using O-Net, then uses a velocity network to learn a deformation gradient field for subsequent frames [S1]. It maps points from material space to deformation space via an ordinary differential equation to achieve continuous motion representation [S1]. It is chosen when continuous dynamic motion is required, but it struggles with large-scale action changes because the fixed mesh is formed only in the first frame [S1].

## Operating characteristics
*   **Time Complexity:** Depends on the number of bones and the depth of the neural networks. Each query point requires evaluation through multiple bone-specific networks.
*   **Memory Complexity:** Scales with the number of bones and the size of the latent codes and network weights.
*   **Strengths:** Allows for detailed local deformation and rigid articulation separately; handles complex shapes like hands and objects effectively [S1].
*   **Limitations:** Requires careful handling of part boundaries and penetration in interactive scenarios; dynamic variants like O-Flow are limited by fixed mesh topology [S1].
*   **When to Use:** Suitable for articulated human bodies and hand-object interactions where part-based deformation is beneficial [S1].
*   **When Not to Use:** Not ideal for non-articulated, monolithic shapes where a single implicit field suffices [G].

## Background
Part-based Occupancy Networks, specifically NASA and HALO-VAE, were developed to address the limitations of monolithic implicit fields in representing articulated human bodies and interacting objects [S1]. NASA treats the human body as an object hinged by multiple bones, learning rigid and non-rigid transformations separately [S1]. HALO-VAE extends this to hand-object grasping, using point cloud encoders and physical constraints [S1]. These methods build on earlier occupancy network approaches like O-Net and O-Flow, which focused on static or dynamic global shapes [S1].

## Sources
[S1] Human 3D Avatar Modeling with Implicit Neural Representation: A Brief Survey, https://doi.org/10.1109/icsps58776.2022.00148, fetched 2026-10-26
