---
title: "CUDA Based Options Pricing using Monte Carlo Simulation"
date: 2025-06-20
author: "Toshit Jain"
words_per_minute : 100
read_time: true
tags :
    - C++
    - CUDA
    - Project
---

# Monte Carlo Option Pricing with C++ & CUDA

This project implements a high-performance Monte Carlo simulation framework in C++ and CUDA for pricing financial derivatives. It supports European, Asian, American, and Basket options, and enables benchmarking between CPU and GPU implementations.

---

## Features

- Pricing for:
  - European Options (Call/Put)
  - Asian Options (Arithmetic Average)
  - American Options (Early Exercise - CPU only)
  - Basket Options (Multiple Assets)
- CPU and GPU (CUDA) implementations
- Template-based payoff abstraction (e.g., `CallPayoff`, `PutPayoff`)
- CLI support using `cxxopts`
- RAII-based Timer utility for benchmarking
- Reproducible results via fixed-seed RNG
- Modular directory structure with clean host/device separation

---

## Prerequisites

- C++17 compiler
- NVIDIA GPU with CUDA Toolkit installed
- CMake or Make (based on your build system)
- `cxxopts` library (included or downloadable)

---

## Project Structure

```bash
.
├── include/
│   ├── benchmark.hpp
│   ├── european.cuh
│   ├── asian.cuh
│   ├── basket.cuh
│   └── american.cuh
├── src/
│   ├── main.cu
│   ├── european.cu
│   ├── asian.cu
│   ├── basket.cu
│   └── american.cu
├── benchmarking/
│   ├── european.cu
│   ├── asian.cu
│   ├── basket.cu
│   └── american.cu
├── third_party/cxxopts/
│   └── cxxopts.hpp
├── Results.ipynb
└── README.md
```

---

## Usage

```bash
!nvcc -std=c++17 -arch=<gpu_architecture> \
  -Iinclude -Ithird_party/cxxopts\
  src/main.cu src/european.cu src/asian.cu src/basket.cu src/american.cu \
  -o option_pricer

./option_pricer --type <option_type> [--payoff <call|put>] [--method <cpu|gpu>] [other flags...]
./option_pricer --help # For help and flags description
```