)*
    # Relightable 3D Gaussian

    Also called: Relightable 3D Gaussian Splatting

    ## Overview
    Relightable 3D Gaussian extends 3D Gaussian Splatting by assigning per-Gaussian BRDF parameters and precomputed visibility to enable dynamic relighting and shadow casting [S1, unofficial]. It distinguishes itself from standard 3D Gaussian Splatting by evaluating material responses to arbitrary light sources rather than relying solely on captured environment maps [S1, unofficial]. It builds on [BRDF and Relighting Extensions](6227-brdf-and-relighting-extensions.md).

    ## How it works
    **Inputs:** A set of 3D Gaussians $\{G_i\}$ storing per-Gaussian BRDF parameters (albedo, roughness, normal) and precomputed visibility for shadows [S1, unofficial]. Scene lighting configuration (direction, intensity, type) [G].
    **Outputs:** A rendered image with physically plausible shading, including shadows and specular reflections based on the defined BRDF [G].
    **Data Structures:** Each Gaussian $i$ stores: albedo $\rho_i \in \mathbb{R}^3$ (diffuse color), roughness $r_i \in \mathbb{R}$ (microfacet roughness), normal $n_i \in \mathbb{R}^3$ (surface orientation), and visibility $v_i$ (shadow occlusion factor) [S1, unofficial].
    **Steps:** Not given in the sources.
    **Formulas:** Not given in the sources.
    **Parameters:** Typical values and numerical ranges are not given in the sources.
    **Edge Cases:** Not given in the sources.

    ## Subtopics
    ### R3DG
    R3DG is a variant that combines BRDF parameters with ambient-occlusion baked per Gaussian [S1, unofficial]. Not given in the sources for when to choose it or its link.

    ## Operating characteristics
    Time and memory complexity, measured performance, scaling, strengths, limitations, and usage guidelines are not given in the sources.

    ## Background
    Relightable 3D Gaussian was introduced by Gao et al. in ECCV'24 [S1, unofficial]. It addresses the limitation of standard 3D Gaussian Splatting in handling dynamic lighting and physically based materials [S1, unofficial].

    ## Sources
    [S1, unofficial] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
