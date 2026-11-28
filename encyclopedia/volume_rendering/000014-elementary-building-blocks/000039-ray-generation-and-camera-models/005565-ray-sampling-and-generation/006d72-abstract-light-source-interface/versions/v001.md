# Abstract Light Source Interface

Also called: Base Light Source

## Overview
The Abstract Light Source Interface defines the contract for light source objects in volume rendering pipelines, specifically enabling backward ray tracing by providing a geometric bounding shape and methods for ray sampling. It distinguishes itself from sibling samplers like [Camera Ray Generation](7456-camera-ray-generation.md) by focusing on emission geometry rather than camera projection, requiring implementations to define how rays originate from a light volume.

## Shared principles
The interface mandates that all concrete light source implementations must provide a bounding shape for intersection tests and implement specific methods for generating and visualizing rays.

**Bounding Shape**
A bounding shape is required if the light source is used with backward ray tracing. This shape defines the spatial extent from which rays are emitted. If not provided, the default is `None` [S1, unofficial].

**Ray Sampling**
Implementations must provide a method to sample rays from the light source. This method takes the number of rays to generate and the target device as arguments and returns a collection of `Rays` objects [S1, unofficial].

**Visualization**
Implementations must provide a method to plot the light source position on a given matplotlib axis [S1, unofficial].

## Subtopics

### [Uniform Sphere Sampling](1669-uniform-sphere-sampling.md)
Samples rays uniformly from a spherical light source. Choose when the light source is isotropic and spherical.

### [Grid Ray Sampler](3930-grid-ray-sampler.md)
Samples rays from a grid-based light distribution. Choose when the light source has a structured, grid-aligned geometry.

### [Ray Casting](6080-ray-casting.md)
General ray casting from a light source. Choose for basic forward rendering scenarios.

### [Camera Ray Generation](7456-camera-ray-generation.md)
Generates rays from the camera eye. Not a light source, but a sibling in ray generation.

### [Ray Sample Generation](12271-ray-sample-generation.md)
General ray sample generation utilities. Choose for low-level sample manipulation.

### [RaySamples](20122-raysamples.md)
Data structure for ray samples. Choose when handling the output of sampling operations.

## Background
The interface originates from the `gradoptics` library by SLAC [S1, unofficial]. It was designed to support backward ray tracing in optical simulations, where light sources must be defined by their bounding geometry to facilitate intersection calculations with incident rays.

## Sources
[S1] magis-slac/gradoptics, https://github.com/magis-slac/gradoptics, fetched 2026-10-27 (unofficial)
