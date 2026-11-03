# Ὀγκογράφος (Onkographos)

Onkographos (Ancient Greek "volume-writer") is a Wikipedia-style encyclopedia of volume rendering (voxel grids first), written by local LLMs from cited sources.
Read it at https://elipticaldrive.github.io/onkographos/.

- `index.html` + `data/`: the static viewer (search, tree diagram, version history and diffs, sources). No server needed.
- `encyclopedia/`: the articles as a directory tree, with every past version, checks and metadata.
- `source_texts/`: extracted text of the cited sources.

Run locally: `python3 -m http.server` in this folder, then open http://localhost:8000/.

Licenses: code MIT (`LICENSE`); article text CC BY-SA 4.0 (`LICENSE-CONTENT`); source texts keep their owners' licenses and
takedown rules (`NOTICE`). Articles are AI-generated and may contain errors.
