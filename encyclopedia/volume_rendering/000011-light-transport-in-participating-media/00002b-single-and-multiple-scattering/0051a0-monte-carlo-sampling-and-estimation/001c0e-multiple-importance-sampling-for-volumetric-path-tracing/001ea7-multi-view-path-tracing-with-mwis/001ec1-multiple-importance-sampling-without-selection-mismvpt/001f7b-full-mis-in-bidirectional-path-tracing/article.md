# Full MIS in Bidirectional Path Tracing

Also called: BPT with Full MIS, BPT: Full MIS Technique, Bidirectional Path Tracing (BPT) with Full MIS, Full Path PDF MIS

## Overview
Full MIS in Bidirectional Path Tracing is a variant of Bidirectional Path Tracing (BPT) that computes Multiple Importance Sampling (MIS) weights using exact, cheap-to-evaluate full path probability density functions (PDFs) derived from the null-scattering framework [S1, S2]. Unlike traditional BPT implementations that restrict MIS weights to directional real-scattering PDFs, this method incorporates medium sampling (attenuation) into the weight calculation, significantly improving accuracy for paths connecting through dense participating media [S1][S3, unofficial]. It builds on [Multiple Importance Sampling without Selection (mismvpt)](1848-multiple-importance-sampling-without-selection-mismvpt.md) by providing the specific path PDF formulation required for BPT vertex connections.

## How it works
The method extends standard BPT by constructing camera and light subpaths using delta tracking and connecting vertices using ratio tracking [S2]. The core innovation is the calculation of the full path PDF for every possible connection technique, allowing for correct MIS weighting [S1, S2].

### Inputs and Outputs
*   **Inputs:** Camera subpath vertices, light subpath vertices, medium parameters (extinction coefficient $\sigma_t$, scattering coefficient $\sigma_s$, phase function $p$), and the null-scattering majorant $\hat{\sigma}_t$ [S1, S2].
*   **Outputs:** A radiance estimate for each pixel, weighted by MIS coefficients derived from the full path PDFs [S2].

### Data Structures
*   **Subpaths:** Lists of vertices $V_c = \{\mathbf{x}_0, \dots, \mathbf{x}_k\}$ for the camera subpath and $V_l = \{\mathbf{y}_0, \dots, \mathbf{y}_m\}$ for the light subpath. Vertices are classified as either real-scattering events or null-scattering (free-flight) events [S1].
*   **Real-Scattering Vertices:** A subset of vertices where actual scattering occurs. Let $r$ be the total number of real-scattering vertices in a combined path [S1, S2].

### Algorithm Steps
1.  **Subpath Construction:** Generate the camera subpath $\mathbf{x}_0, \dots, \mathbf{x}_k$ and light subpath $\mathbf{y}_0, \dots, \mathbf{y}_m$ using delta tracking [S2]. Delta tracking samples the distance to the next interaction (real or null) using the majorant $\hat{\sigma}_t$ [S1].
2.  **Vertex Connection:** For each pair of real-scattering vertices $\mathbf{x}_i$ (from camera subpath) and $\mathbf{y}_j$ (from light subpath), attempt to connect them [S2].
3.  **Ratio Tracking for Connection:** To estimate the transmittance between $\mathbf{x}_i$ and $\mathbf{y}_j$, use ratio tracking. This involves sampling null-scattering subpaths between the vertices to estimate the extinction transmittance $T$ [S1]. This step is crucial for obtaining the exact PDF contribution of the connection segment [S1].
4.  **Determine Connection Technique:** A connection technique is defined by the number of real-scattering vertices on the light subpath ($s$) and the camera subpath ($t$) that are connected. For a path with total real-scattering length $r$, the indices must satisfy $s + t - 1 = r$ [S1, S2].
5.  **Calculate Full Path PDF:** Compute the full path PDF $p(\text{path})$ for the specific technique $(s, t)$ using Equation 44 from [S1]. This PDF accounts for:
    *   The probability of generating the specific prefix subpaths (camera and light).
    *   The probability of the ratio-tracking connection segment.
    *   The medium sampling probabilities (attenuation) along the entire path [S1, S2].
6.  **Compute MIS Weights:** Use the full path PDFs of all valid connection techniques for the given path to compute MIS weights (e.g., using the balance heuristic or power heuristic) [S2].
7.  **Estimate Radiance:** Combine the path contributions weighted by the MIS weights to produce the final radiance estimate [S2].

### Formula
The full path PDF for a BPT technique connecting $s$ real-scattering vertices on the light subpath and $t$ real-scattering vertices on the camera subpath (where $s + t - 1 = r$) is given by [S1, S2]:

$$
p(\text{path}) = p_{\text{cam}}(\mathbf{x}_0, \dots, \mathbf{x}_{k}) \cdot p_{\text{light}}(\mathbf{y}_0, \dots, \mathbf{y}_{m}) \cdot p_{\text{conn}}(\mathbf{x}_i, \mathbf{y}_j)
$$

Where $p_{\text{cam}}$ and $p_{\text{light}}$ are the PDFs of generating the respective subpaths via delta tracking, and $p_{\text{conn}}$ is the PDF of the connection segment estimated via ratio tracking [S1]. The exact form of Equation 44 in [S1] combines these terms, explicitly including the null-scattering probabilities and the transmittance estimates derived from ratio tracking.

### Parameters
*   **Majorant $\hat{\sigma}_t$:** The upper bound on the extinction coefficient used for delta tracking. Must be chosen to be tight enough for efficiency but valid for all media [S1].
*   **Spectral Handling:** The described implementation handles achromatic media only. Spectral MIS is not employed in the reference BPT implementation [S1, S2].

### Edge Cases and Pitfalls
*   **Dense Media:** Traditional directional MIS fails in scenes where the medium covers the light source because it ignores medium sampling, leading to poor weights for connections through dense regions [S3, unofficial]. Full MIS correctly prioritizes these connections [S3, unofficial].
*   **Ratio Tracking Variance:** While ratio tracking provides exact PDFs, it can introduce variance if the majorant is loose. However, it is cheaper than ray-marching for transmittance estimation [S1].
*   **Degenerate Paths:** Paths with $r=0$ (no real scattering) are handled by the standard BPT connection logic, but the full PDF formulation ensures consistency with the null-scattering framework [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def full_mis_bpt(camera_subpath, light_subpath, medium):
    """
    Perform Bidirectional Path Tracing with Full MIS using null-scattering.
    
    Args:
        camera_subpath: List of vertices from camera, including null-scattering steps.
        light_subpath: List of vertices from light, including null-scattering steps.
        medium: Medium parameters including extinction, scattering, phase function, and majorant.
        
    Returns:
        radiance_estimate: Weighted sum of path contributions.
    """
    radiance = 0.0
    
    # Identify real-scattering vertices in subpaths
    cam_real_vertices = [v for v in camera_subpath if v.is_real_scattering]
    light_real_vertices = [v for v in light_subpath if v.is_real_scattering]
    
    # Iterate over all possible connection pairs
    for i, x_i in enumerate(cam_real_vertices):
        for j, y_j in enumerate(light_real_vertices):
            
            # Determine technique indices s and t
            # s: number of real-scattering vertices on light subpath involved in connection
            # t: number of real-scattering vertices on camera subpath involved in connection
            # For a connection between x_i and y_j, the path consists of:
            # camera prefix (0 to i), connection (x_i to y_j), light prefix (j to end)
            # However, BPT typically connects specific vertices. 
            # Let s be the index in light_real_vertices (0-based) + 1
            # Let t be the index in cam_real_vertices (0-based) + 1
            s = j + 1
            t = i + 1
            r = s + t - 1  # Total real-scattering vertices
            
            # Calculate full path PDF for this technique
            # p_cam: PDF of generating camera subpath up to x_i
            p_cam = compute_subpath_pdf(camera_subpath, 0, i, medium)
            
            # p_light: PDF of generating light subpath from y_j to end
            p_light = compute_subpath_pdf(light_subpath, j, len(light_real_vertices)-1, medium)
            
            # p_conn: PDF of connecting x_i to y_j using ratio tracking
            # This includes the transmittance estimation via null-scattering
            p_conn = compute_connection_pdf(x_i, y_j, medium)
            
            full_pdf = p_cam * p_light * p_conn
            
            # Calculate path contribution (radiance transfer)
            # This involves geometry terms, phase functions, and albedo
            contribution = compute_path_contribution(x_i, y_j, camera_subpath, light_subpath, medium)
            
            # Store technique for MIS weighting
            # In a full implementation, we would collect all techniques for the same path
            # and compute weights. Here we assume a simplified balance heuristic or 
            # that we are computing the weight for this specific technique relative to others.
            # For the purpose of this article, we assume the full MIS weight is computed
            # by summing PDFs of all valid techniques for the same geometric path.
            
            # Note: In practice, one would gather all (s,t) pairs that form the same 
            # geometric path and compute weights w(s,t) = p(s,t) / sum(p(all techniques))
            # Then radiance += contribution * w(s,t)
            
            radiance += contribution * full_pdf # Placeholder for actual MIS weighting logic
            
    return radiance

def compute_subpath_pdf(subpath, start_idx, end_idx, medium):
    """
    Compute the PDF of generating a subpath segment using delta tracking.
    """
    pdf = 1.0
    for k in range(start_idx, end_idx + 1):
        vertex = subpath[k]
        # Probability of free-flight distance
        pdf *= medium.majorant * exp(-medium.majorant * vertex.distance)
        # Probability of scattering event type (real vs null)
        if vertex.is_real_scattering:
            pdf *= medium.extinction / medium.majorant
        else:
            pdf *= 1.0 # Null scattering probability is handled in the majorant sampling
    return pdf

def compute_connection_pdf(x_i, y_j, medium):
    """
    Compute the PDF of the connection segment using ratio tracking.
    """
    # Ratio tracking estimates transmittance T and its PDF
    # The PDF of the connection is proportional to the estimated transmittance
    # divided by the majorant and the number of samples used in ratio tracking.
    # This is an approximation that becomes exact as samples -> infinity.
    # For the purpose of this article, we assume the exact PDF is derived from
    # the null-scattering path integral formulation [S1].
    transmittance = estimate_transmittance_ratio_tracking(x_i, y_j, medium)
    distance = distance_between(x_i, y_j)
    # Simplified PDF for connection segment
    pdf = transmittance / (distance * medium.majorant) 
    return pdf
```

## Subtopics

### Directional MIS in BPT
Uses only real-scattering PDFs for weights. Simpler but inaccurate in dense media. Choose when media is sparse or homogeneous and performance is critical. [S1][S3, unofficial]

### Unidirectional Path Tracing with NEE
Uses Next-Event Estimation with unidirectional paths. Fails in scenes with enclosed lights or dense media due to geometric singularities. [S3, unofficial]

### Spectral Null-Scattering BPT
Extends the method to spectral rendering. Not implemented in the reference BPT code but theoretically possible by multiplying technique counts by color components. [S1]

## Operating characteristics
*   **Time Complexity:** Similar to standard BPT, but with overhead from ratio tracking for each connection. Ratio tracking is cheaper than ray-marching for transmittance estimation [S1].
*   **Memory Complexity:** Depends on subpath length. No significant additional memory compared to standard BPT [S1].
*   **Performance:** In scenes with dense media covering the light source, Full MIS achieves lower RMSE (0.049) compared to Directional MIS (0.043) and Unidirectional + NEE MIS (0.731), with LTUV of 57.2M lookups vs 44.9M for Dir. MIS and 12.7B for Uni.+NEE [S3, unofficial].
*   **Strengths:** Accurate weighting in dense media; exact PDFs; cheap transmittance estimation via ratio tracking [S1][S3, unofficial].
*   **Limitations:** More complex to implement than directional MIS; achromatic only in reference implementation [S1, S2].

## Background
The method originates from the null-scattering path integral formulation of light transport [S1, S2]. It was developed to address the limitations of traditional BPT in participating media, where directional MIS fails to account for medium sampling [S1]. The work was presented by Miller, Georgiev, and Jarosz in 2019 [S3, unofficial].

## Sources
[S1] PDF A null-scattering path integral formulation of light transport, https://cs.dartmouth.edu/~wjarosz/publications/miller19null.pdf, fetched 2026-10-29
[S2] PDF A null-scattering path integral formulation of light transport, https://blogs.autodesk.com/media-and-entertainment/wp-content/uploads/sites/162/s2019_null_path_formulation_rev2.pdf, fetched 2026-10-29
[S3] A Null-Scattering Path Integral Formulation of Light Transport, https://www.scribd.com/document/1067625811/A-Null-scattering-Path-Integral-Formulation-of-Light-Transport, fetched 2026-10-29 (unofficial)
