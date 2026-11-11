# SDF NSVF Models

## Overview
SDF NSVF models are neural volume rendering models that use the `sdf_radiance_field` mechanism to predict both the signed distance to the surface and the view-dependent radiance [S1, unofficial]. They build on the [MLPRender_Fea](52-mlprender-fea.md) architecture but adapt it for SDF-based implicit surface representation [11064-sdf-nsvf-models.md].

## Shared principles
The family relies on the `sdf_radiance_field` mechanism, where the neural network predicts both the signed distance to the surface and the view-dependent radiance [11064-sdf-nsvf-models.md]. This builds on the [MLPRender_Fea](52-mlprender-fea.md) architecture but adapts it for SDF-based implicit surface representation [11064-sdf-nsvf-models.md].

## Subtopics

### sdf_nsvf
A standard SDF NSVF model configuration. It uses the `nerf2nope_architecture` with `feature_layers=6`, `feature_field_skip_connect=3`, and `no_layernorm_mlp=True` [S1, unofficial].

### sdf_nsvf_sfx
A variant of the SDF NSVF model that uses the `surface_volume_rendering` raymarcher [S1, unofficial].

## Background
The SDF NSVF models are part of the NSVF (Neural Surface-embedded Volume Fields) framework developed by Facebook Research [S1, unofficial]. The codebase is available on GitHub [S1, unofficial].

## Sources
[S1] facebookresearch/NSVF, https://github.com/facebookresearch/NSVF, fetched 2026-10-27 (unofficial)
