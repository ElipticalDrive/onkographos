# TouchDesigner Volumetric

This repository provides components for working with volumetric displays in
TouchDesigner.

## Components

#### `volumetric.tox`

A TouchDesigner component for rendering a SOP and input texture to a volumetric
display. Currently, only cubic output resolutions are supported.

The component uses an instanced rendering method leveraging clipping planes to
decompose the input geometry into slices. Slices are rendered in all three
dimensions to ensure that geometry is visible in all orientations.

The component outputs a 3D texture that can be used with volumetric displays,
or can be combined with T3D or other volumetric rendering methods.

#### `volumetric_artnet_out.tox`

A TouchDesigner component for outputting a 3D texture to a volumetric display
made up of LED curtains controlled via Art-Net.

## Demo Video

[![Volumetric Demo](https://img.youtube.com/vi/AdCHABN6ZCg/0.jpg)](https://www.youtube.com/watch?v=AdCHABN6ZCg)
