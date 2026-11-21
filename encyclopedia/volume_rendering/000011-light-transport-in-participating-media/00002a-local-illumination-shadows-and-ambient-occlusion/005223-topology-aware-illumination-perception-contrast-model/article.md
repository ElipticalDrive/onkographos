# Topology-aware illumination perception contrast model

Also called: Perceptual illumination, Distance illumination, Saliency illumination, Topology-aware lighting

## Overview
The Topology-aware illumination perception contrast model is a volume rendering framework that utilizes topological attributes derived from a contour tree to automatically generate distinct illuminations for subregions, allowing users to perceive structural differences (such as importance and topological distance) via lighting cues [S1]. It specifically addresses the problem of inner structures being obscured by outer structures in semi-transparent volumes by applying less illuminance to outside structures and higher illuminance to inner structures [S1]. Unlike standard lighting models that treat all geometry equally or rely solely on spatial distance, this method maximizes visual perception differences between neighboring objects by optimizing luminance contrast based on topological saliency and persistence [S1].

## How it works
The method operates in two phases: an Initialization Phase and an Optimization Phase. It requires a pre-computed contour tree of the volume data, which is first simplified to reduce noise [S1].

### Inputs and Outputs
*   **Inputs:**
    *   Simplified contour tree of the volume data.
    *   Initial root luminance $L_0$ (user-specified).
    *   Initial illumination contrast ratio $\gamma_0$ (user-specified, range $[0, 1]$) [S1].
    *   Topological attributes for each branch: persistence ($p$), volume ($v$), hypervolume ($hv$), and saddle value ($s$) [S1].
    *   User-modulated weights: $w$ (for saliency/distance models) and $V$ (lighting speed in medium) [S1].
*   **Outputs:**
    *   Illumination coefficients ($k_a, k_d, k_s$) for each branch in the contour tree.
    *   Luminance distribution across the contour tree.

### Data Structures
The core data structure is the **Contour Tree**, where each node represents a branch (a connected component of a level set). Each branch node stores:
*   **Topological Attributes:**
    *   $p_i$: Persistence of branch $i$.
    *   $v_i$: Volume of branch $i$.
    *   $hv_i$: Hypervolume of branch $i$.
    *   $s_i$: Saddle value of branch $i$.
    *   $depth_i$: Topological depth (distance from root).
    *   $n_{ci}$: Number of children of branch $i$.
    *   $n_{si}$: Number of siblings (branches sharing the same parent).
*   **Illumination State:**
    *   $L_i$: Luminance of branch $i$.
    *   $k_{d,i}$: Diffuse coefficient for branch $i$.

### Formulas and Definitions

**1. Base Illumination Model (Blinn-Phong)**
The color $C$ of a voxel is computed as [S1]:
$$ C = (k_a + k_d (N \cdot L)) C_{tf} + k_s (N \cdot H)^n $$
Where:
*   $k_a, k_d, k_s$: Ambient, diffuse, and specular coefficients.
*   $N$: Normalized gradient vector of the voxel.
*   $L$: Normalized lighting direction.
*   $H$: Normalized half-way direction.
*   $n$: Shininess exponent.
*   $C_{tf}$: Color defined by the transfer function.

**2. Topology-Aware Attenuation Factor ($f_{att}$)**
Light intensity weakens over distance. The attenuation factor is defined using Stokes' law adapted for topology [S1]:
$$ f_{att} = e^{-\alpha z} $$
Where:
*   $z$: Modeled as the persistence $p$ of the branch [S1].
*   $\alpha$: Attenuation coefficient derived from Stokes' law [S1]:
    $$ \alpha = \frac{2 \eta_{visc} \eta_{freq}^2}{3 \rho V^3} $$
    *   $\eta_{visc}$: Dynamic viscosity (0.001002 P·s for water at 20°C) [S1].
    *   $\rho$: Density (998.2071 kg/m³ for water at 20°C) [S1].
    *   $\eta_{freq}$: Modeled as the number of siblings $n_{si}$ [S1].
    *   $V$: Lighting speed in the medium (user-controlled) [S1].

The modified Blinn-Phong equation becomes [S1]:
$$ C = (k_a + k_d f_{att} (N \cdot L)) C_{tf} + k_s f_{att} (N \cdot H)^n $$

**3. Just-Noticeable-Difference (JND)**
The minimal luminance change $\Delta L_i$ required for an observer to perceive a difference, based on luminance $L_i$ [S1]:
$$ \log(\Delta L_i) = \begin{cases} -2.86 & \text{if } \log(L_i) < -3.94 \\ (0.405 \log(L_i) + 1.6)^{2.18} - 2.86 & \text{if } -3.94 \le \log(L_i) < -1.44 \\ \log(L_i) - 0.395 & \text{if } -1.44 \le \log(L_i) < -0.0184 \\ (0.294 \log(L_i) + 0.65)^{2.7} - 0.72 & \text{if } -0.0184 \le \log(L_i) < 1.9 \\ \log(L_i) - 1.255 & \text{if } \log(L_i) \ge 1.9 \end{cases} $$

**4. Illumination Contrast Ratio ($\gamma$)**
Defined between parent branch $i-1$ and child branch $j$ [S1]:
$$ \gamma = \frac{L_{child}}{L_{parent}} $$
Range $[0, 1]$. $\gamma=0$ implies the child is black; $\gamma=1$ implies equal luminance. The model aims for $\gamma < 1$ so children (inner structures) have higher luminance than parents (outer structures) [S1].

**5. Saliency Illumination ($k_d$ function)**
For "Saliency Illumination," $k_d$ is a function of topological saliency $Sri$ [S1]:
$$ k_d = w S_{ri} $$
Where $w$ is a user weight. $Sri$ is calculated using a Gaussian weighting function $\omega_{ij}$ based on topological depth difference $d(c_i, c_j)$ and maximum depth $r$ [S1]:
$$ \omega_{ij} = e^{-\frac{d(c_i, c_j)^2}{r^2}} $$

**6. Distance Illumination ($k_d$ function)**
For "Distance Illumination," $k_d$ is a function of topological distance (depth $z$) using a Sigmoid function [S1]:
$$ k_d = \frac{w}{2} \frac{1}{1 + e^{-z}} $$
This assigns higher $k_d$ to deeper (inner) branches [S1].

**7. Optimization Constraints**
The optimization must satisfy [S1]:
1.  $\Delta L_i \ge \Delta L_{JND, i}$
2.  $\gamma_i \ge \gamma_0$
3.  $L_i \ge L_{i,0}$ (where $L_{i,0}$ is the initial luminance from the Initialization Phase)

### Algorithm Steps

#### Phase 1: Initialization
1.  **Simplify Contour Tree:** Reduce noise in the topological structure [S1].
2.  **Set Root Luminance:** User specifies $L_0$ for the root branch [S1].
3.  **Recursive Initialization:**
    *   For each branch $i$ (starting from root):
        *   Calculate JND difference $\Delta L_{JND}$ between parent and children using the JND formula [S1].
        *   Set initial luminance for children $L_c = L_{parent} + \Delta L_{JND}$ [S1].
        *   Recursively apply to all branches from root to leaves [S1].

#### Phase 2: Optimization
The goal is to maximize luminance differences while satisfying constraints. The process iterates through depth levels $i \ge 1$.

**Step 2.1: Parent-to-Children Distribution**
For each depth level $i$:
1.  Calculate new base luminance $L'_{i-1}$ for the parent level based on contrast ratio $\gamma_{i-1}$ and parent luminance [S1].
    *   *Note: The source defines $L'_{i-1} = L'_{i-2} / \gamma_{i-1}$ effectively, but since $\gamma < 1$ and children should be brighter, the logic implies $L_{child} = L_{parent} / \gamma$ or similar scaling to ensure $L_{child} > L_{parent}$ if $\gamma$ represents parent/child ratio. However, the text states $\gamma = L_i / L_j$ (parent/child) and $\gamma \in [0,1]$. Thus $L_{child} = L_{parent} / \gamma$. If $\gamma=0.5$, child is 2x brighter.*
2.  For each child $j$ of parent $i$:
    *   Get initial luminance band $L'_{ij}$ from Initialization Phase.
    *   Calculate luminance difference $\Delta L_{ij} = |L'_{ij} - L_{parent}|$ [S1].
    *   **Check Constraint:** If $\Delta L_{ij} \ge \Delta L_{JND, ij}$:
        *   Define final $L'_{ij}$ as the initial value.
    *   **Else:**
        *   Define final $L'_{ij}$ by adjusting to meet the JND difference (increase luminance if necessary) [S1].
    *   Compute illumination coefficients ($k_d$) for branch $j$ based on the new luminance $L'_{ij}$ using either Saliency or Distance illumination formulas [S1].
3.  **Update Contrast Ratio:**
    *   Calculate absolute head loss $\Delta h_i$ using the residue flow model [S1]:
        $$ \Delta h_i = \frac{1}{Q} \cdot \frac{p_i}{K \cdot n_{ci}} $$
        *   $Q$: Fluid flow speed (user parameter).
        *   $K$: Hydraulic conductivity (300 for sand) [S1].
        *   $p_i$: Persistence of branch $i$.
        *   $n_{ci}$: Number of children of branch $i$.
    *   Update contrast ratio for next level:
        $$ \gamma_i = (1.0 - \Delta h_i) \cdot \gamma_{i-1} $$
    *   This results in smaller contrast ratios (higher relative luminance for children) for deeper levels [S1].

**Step 2.2: Sibling Distribution**
Distribute the base illumination $L'_i$ among siblings $j=1 \dots n_{si}$ on level $i$ [S1]:
$$ L'_{ij} = L'_i \cdot f_b(p_{ij}, v_{ij}, hv_{ij}) \cdot f_d(s_{ij}) $$
*   $f_b$: Function controlling distribution based on importance (persistence, volume, hypervolume). Defined as the area of the importance triangle formed by these three attributes [S1].
*   $f_d$: Function controlling distribution based on saddle value $s_{ij}$ [S1]:
    $$ f_d(s_{ij}) = \frac{s_{ij} - s_{min, i}}{\Delta s_i} $$
    Where $s_{min, i}$ is the minimum saddle value among siblings on level $i$, and $\Delta s_i = s_{max, i} - s_{min, i}$ [S1].

### Edge Cases and Pitfalls
*   **Precision:** The JND formula involves logarithms and piecewise functions; care must be taken with $L_i \le 0$ (luminance must be positive) [S1].
*   **Boundary Conditions:** $\gamma=0$ results in black child branches; $\gamma=1$ results in no contrast [S1].
*   **Noise:** Raw contour trees may contain noise affecting topology; simplification is required before attribute extraction [S1].
*   **Parameter Sensitivity:** $V$ (lighting speed) and $Q$ (flow speed) are interactive and significantly affect attenuation and contrast propagation [S1].

## Operating characteristics
*   **Time Complexity:** Depends on the size of the contour tree (number of branches) and the depth of the tree. The optimization iterates through depth levels and branches. Simplification is a preprocessing step.
*   **Memory Complexity:** Proportional to the number of branches in the contour tree, storing topological attributes and illumination coefficients.
*   **Strengths:** Automatically emphasizes inner structures and important topological features without manual tuning of per-object lighting. Maximizes perceptual contrast based on human vision models (JND).
*   **Limitations:** Requires pre-computation of a contour tree. Performance depends on the complexity of the topological simplification. The model assumes a specific relationship between topological depth and visual importance (inner = more important), which may not hold for all datasets.
*   **When to use:** When the goal is to reveal internal structures in complex, nested volumetric data (e.g., biological structures) and topological hierarchy is meaningful.
*   **When not to use:** For flat or non-nested data where topological depth does not correlate with visual importance, or when real-time performance is critical and contour tree computation is too expensive.

## Background
The method was introduced by Zhou et al. in 2016 [S1]. It builds upon previous work on topology-aware visualization, such as the use of contour trees for data segmentation and opacity transfer functions [S1]. The framework integrates concepts from psychophysics (JND) and fluid dynamics (residue flow model) to model light attenuation and contrast distribution in a topological context. It is part of the broader field of topology-aware illumination design, which includes sub-methods like "Saliency Illumination" and "Distance Illumination" [S1].

## Sources
[S1] Topology-aware illumination design for volume rendering, https://doi.org/10.1186/s12859-016-1177-4, fetched 2026-10-26
