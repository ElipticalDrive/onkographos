# Wildframe

A procedural open-world playground built with React Three Fiber.

Explore a seeded landscape of rolling terrain, mountain ridges, and coastline. Wildframe is an experimental browser-based world, with an emphasis on terrain streaming, third-person exploration, and real-time rendering.

[Browser demo](https://crapthings.github.io/wildframe/) — available after the first successful Pages deployment. Use a desktop browser with WebGL2, a keyboard, and a mouse. The high-detail tree downloads after entering the scene and may take time on slower connections.

## Current features

- A 1,024 × 1,024 meter world with deterministic terrain generation.
- Worker-generated terrain chunks, distance-based LOD, and bounded streaming caches.
- Blended ground and cliff materials with hex tiling to reduce repetition.
- Procedural ground stones, an animated character, and a sample tree.
- Third-person movement and camera controls.
- Day/night transitions, stars, moonlight, and a rain toggle.
- A terrain debug map with contours and click-to-teleport navigation.
- Development-only frame-time and streaming diagnostics.

This is an early prototype, not a finished game or a general-purpose engine. The playable world is bounded; the distant landscape provides visual continuity. Performance work is ongoing, including occasional chunk-transition hitches and the high-detail sample tree.

## Getting started

Requires Node.js `^20.19.0 || >=22.12.0` and pnpm `12.4.1`.

1. Install dependencies with `pnpm install --frozen-lockfile`.
2. Download and verify the complete asset pack from the [v0.1.0 release](https://github.com/crapthings/wildframe/releases/tag/v0.1.0), then extract it into `public/assets/`, following [the asset setup guide](ASSETS.md).
3. Start the development server with `pnpm dev` and open the URL printed by Vite.

The scene requires local assets; missing terrain textures or the player model will prevent loading from completing. It does not currently provide placeholder replacements.

For a production bundle, run `pnpm build`, then `pnpm preview` to inspect it locally. The package remains `private: true` to prevent accidental npm publication; this does not control GitHub repository visibility.

## GitHub Pages deployment

The [Pages workflow](.github/workflows/pages.yml) runs when a commit is pushed to `main`, or manually from the Actions tab. A local commit alone does not deploy anything.

It installs the pnpm version in `package.json`, restores the pinned release asset archive, verifies its SHA-256 checksum, and builds with the Pages base path. Source and binary assets are then served together from Pages; browsers do not fetch models directly from GitHub Releases. No personal access token or extra repository secret is required. GitHub Pages must use **GitHub Actions** as its deployment source; this has been configured for `crapthings/wildframe` and must be set separately for forks.

Local development defaults to `/`. To check the project-site build locally, run `VITE_BASE_PATH=/wildframe/ pnpm build`, then `pnpm preview` and open `/wildframe/`. Model, HDR, and texture URLs all use Vite's `BASE_URL`; Worker entry points are resolved by Vite.

When publishing updated assets, update `ASSET_RELEASE`, `ASSET_ARCHIVE`, and `ASSET_SHA256` in the workflow together. The site includes the full asset set and is relatively large; monitor bandwidth and optimize the sample tree before promoting the demo widely.

## Controls

| Input | Action |
| --- | --- |
| W / A / S / D | Move relative to the camera |
| Shift | Sprint |
| Left or right mouse drag | Look around |
| Mouse wheel | Zoom |
| Tab | Toggle the terrain map |
| Click land on the map | Teleport |
| Day / Night / Rain buttons | Change lighting and weather |

## Structure

`src/scenes/outdoor/` contains the first scene, including its terrain generation, streaming, materials, player, camera, environment, and UI. See [the scene guide](src/scenes/outdoor/README.md) for implementation details and performance diagnostics.

The main stack is React, Three.js, React Three Fiber, Drei, FastNoise Lite, Koota, and Zustand. Koota owns live entity state; Zustand owns UI state and commands.

## Licensing and assets

No open-source license has been selected for the project code yet. The third-party models, textures, and environments distributed in the asset pack are CC0 assets from Poly Haven and Quaternius; their license does not apply to the project code. See [ASSETS.md](ASSETS.md) for setup and release-archive distribution, and [THIRD_PARTY_ASSETS.md](THIRD_PARTY_ASSETS.md) for source and license records.
