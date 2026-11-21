# Layoutit Voxels

Layoutit Voxels is a browser-based 3D CSS voxel editor that renders editable
voxel models as real HTML/CSS 3D geometry, without a WebGL or canvas scene
renderer. It lets you draw voxel-style grids, import and export MagicaVoxel
`.vox` files, share models through compressed URL hashes, copy iframe embeds,
open CodePen, and download generated HTML, CSS, TXT, JSON, PNG, and VOX output.

Play the live version: [voxels.layoutit.com](https://voxels.layoutit.com)

## How to Use

Install dependencies and run the local dev server:

```sh
npm ci
npm run dev
```

For production checks:

```sh
npm test
npm run build
npm run generate
```

`npm test` runs focused regression tests for URL encoding, VOX parsing, visible
face selection, import bounds, and undo/redo history. `npm run generate` writes
the static site to the ignored `dist/` folder.

The optional gallery save action needs public Supabase client settings at build
time:

```sh
NUXT_ENV_SUPABASE_URL="<project URL>"
NUXT_ENV_SUPABASE_ANON_KEY="<anon key>"
```

## How It Works

Layoutit Voxels is a Nuxt 2 app with a shared Vue context in
`src/plugins/globalObject.js`. The main editor lives in `src/pages/index.vue`,
with scene rendering split across `src/components/layers.vue` and the shape
components for cubes, slabs, wedges, spikes, and flat faces.

Each voxel is projected into real DOM elements positioned in 3D with CSS
transforms, grid placement, and textured backgrounds from `static/textures/`.
The app uses browser DOM/CSS as the editor renderer; canvas-style rendering is
only used indirectly when exporting a PNG snapshot through `dom-to-image`.

Model state is serialized into the URL hash by `src/utils/urlCodec.mjs`, then
compressed with the browser `CompressionStream` API. The `/embed` route reads
the same hash and renders a lightweight auto-rotating view for iframe embeds:

```text
https://voxels.layoutit.com/#<compressed-model>
https://voxels.layoutit.com/embed#<compressed-model>&format=CSS
```

## Build and Runtime

The browser does not fetch source data or generate texture assets at runtime.
Example thumbnails and texture SVGs are committed under `static/`, then Nuxt
copies them to the generated site root.

`src/utils/voxParser.mjs` parses imported MagicaVoxel files into the internal
voxel layers. Export and share actions are handled from
`src/components/ribbonTools.vue`; gallery submission is enabled only when the
Supabase public client environment variables are configured.

Generated output and local dependencies are intentionally ignored:

- `.nuxt/`
- `dist/`
- `node_modules/`
- `output/playwright/`

## Source Layout

- `src/pages/` contains the editor and embed routes.
- `src/components/` contains the toolbars, sidebars, scene layers, shapes, and
  inline icon/texture components.
- `src/utils/` contains the pure model utilities for URL codec, VOX parsing,
  bounds, visible faces, and history.
- `static/` contains public source assets served from the site root.
- `test/` contains the Node regression tests.

## License

Layoutit Voxels source code is [GPL-2.0-only](LICENSE).
