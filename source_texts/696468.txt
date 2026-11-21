ZeroBit - A high-performance file compressor written in modern C++, leveraging a multi-stage pipeline-including Burrows-Wheeler Transform (BWT), Move-To-Front (MTF), Zero Run-Length Encoding (RLE), and an adaptive context-mixing range coder-to achieve efficient, lossless compression. Specifically created for small files.

---

## 📋 Table of Contents

- [⚙️ Features](#️-features)
- [📂 Usage](#-usage)
- [🔍 File Format](#-file-format)
- [📚 Algorithms](#-algorithms)
- [👤 Author](#-author)

---

## ⚙️ Features

- **Bidirectional**: Supports both compression and decompression.
- **Adaptive Modeling**: Combines multiple context models (byte, bit, match, LZP) with online mixing.
- **Single-header Implementation**: Minimal dependencies; requires only C++17 and Qt 6.9.0.
- **Portable**: Uses `std::filesystem` for cross-platform file handling.

---

## 📂 Usage

- Best for: Text files with a lot of numerical values.
- Bad for: Text files with a lot of non-structured text.

## 🔍 File Format

Each compressed file's header is structured as follows:
- Original file size (uint64_t)
- Block length (uint32_t)
- BWT primary index (uint32_t)
- RLE symbol count (uint32_t)
- Compressed data size (uint32_t)
- Range-coded payload (bytes)

This design enables streaming decompression without loading the entire file into memory.

## 📚 Algorithms

| Stage | Technique                              | Purpose                             |
|-------|----------------------------------------|-------------------------------------|
| 1     | **Burrows–Wheeler Transform (BWT)**    | Increases symbol locality           |
| 2     | **Move-To-Front (MTF)**                | Exposes runs of low symbols         |
| 3     | **Zero Run-Length Encoding (RLE)**     | Efficiently encodes zero runs       |
| 4     | **Adaptive Context Models + Mixer**    | Learns bitwise patterns dynamically |
| 5     | **Range Coding**                       | Optimal entropy encoding            |

## 👤 Author
Developed by Stefan Rusev (stefanrusev02@gmail.com)
