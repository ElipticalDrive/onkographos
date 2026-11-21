# Dual Numbers — Interactive

二重数 (dual numbers) を用いた前進モード自動微分の対話型デモ。
React + TypeScript + Vite + Tailwind v4 + better-react-mathjax で実装。

## 開発

```bash
npm install
npm run dev          # http://localhost:5173
npm test             # Vitest (unit + RTL components)
npm run test:e2e     # Playwright (chromium)
npm run typecheck
npm run lint
npm run build
```

## アーキテクチャ

- ルーティング: HashRouter (`#/intro` etc., 6 routes)
- 状態: 各 route の useState、共有状態なし
- 数式: better-react-mathjax (`<MathJax dynamic>`)
- 描画: `useCanvas` hook + 純関数 `drawAxes`/`drawCurve`/`drawTangent`

## デプロイ

Vercel: フレームワークプリセット **Vite** で自動検出、設定不要。

## 履歴

- v1.0.0 (2026-05-10): 単一ファイル vanilla HTML/JS 版 (`docs/legacy/index.html` and tag `v1.0.0`)
- v2.0.0 (作業中): React + TypeScript + Vite 版

詳細: [`CHANGELOG.md`](./CHANGELOG.md), 設計: [`docs/superpowers/specs/`](./docs/superpowers/specs/)

## License

MIT (see [LICENSE](./LICENSE))
