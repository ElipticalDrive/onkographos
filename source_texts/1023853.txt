# Automatic-differentiation-with-dual-numbers
A brief (and inaccurate) history of derivatives, with a brief (and incomplete) Python implementation

## Overview

While derivates are at the heart of ML training, we rarely think about computing them: arguably, the capabilities of modern ML packages when it comes to "automatic differentiation" is one of the reasons behind the explosive growth of the field.

The [blog post](https://www.cantorsparadise.com/marginally-important-automatic-differentiation-with-dual-numbers-2448dc58e5e2) "Marginally important: automatic differentiation with dual numbers" is a brief introduction to supernatural, infinitesimal and dual numbers, with unexpected application to one of the core concepts of ML: Newton-and-Leibniz calculus, re-booted.

This repo contains a simple code snippet to implement differentiation through dual numbers.

## Setup

Activate a Python virtual environment:

```
python -m venv venv
source venv/bin/activate
pip install -r requirements.txt
```

Then run the code with:

```
cd src
python dual.py
```

You should get in the terminal the examples from the [blog post](https://www.cantorsparadise.com/marginally-important-automatic-differentiation-with-dual-numbers-2448dc58e5e2), and the results of comparing the dual implementation with [MyGrad](https://github.com/rsokl/MyGrad).

## License

All the code is released without warranty, "as is" under a MIT License. This was a fun week-end project and should be treated with the appropriate sense of humour.
