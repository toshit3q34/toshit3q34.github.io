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

## Introduction & Motivation

Hi. So the main drive behind this project was to learn parallel processing through CUDA. At the same time,I was also invested in learning Finance from the book <em>Paul Wilmott Introduces Quantitative Finance</em>. I completed the basics like knowledge about financial quantities and options and I cam across a method to price
options which was <em>Monte Carlo Simulation</em>. I learned sometime before that we can parallelize any Monte Carlo Algorithm and so began my project. (Later, I decided to collaborate with one of my friends at IITG who was interested in the project as well).

The implementation of the whole project can be found here : [Github Repository](https://github.com/toshit3q34/CUDA-Based-Options-Pricing)

## CUDA Basics

For the CUDA Basics, I referred to a couple of videos and resources available online (some videos from IITM Profs, some from youtube, etc.) and learned things like writing CUDA Kernels and what are grids, blocks and things like cudaMemcpy.

## Project Overview

Our implementation included the following :
- Writing CPU Functions for European, Asian, Basket and American Options (for American used LSM so only CPU).
- Writing GPU Kernels for European, Asian and Basket Options.
- Make code as modular as possible. (Used Struct Functors for Template).
- Benchmarking Class (Used RAII-type Timer class).
- Integrate CLI (Used third-party cxxopts).

## Payoffs Functor Structs

Following is the code for Payoff Functor Structs. I originally thought of using inheritance but turns out that CUDA Kernel does not support inheritances ( due to virtual tables begin local to the memory of host :( ).

```cpp
#pragma once
#include <cmath>

struct CallPayoff {
  __device__ __host__ double operator()(double S, double K) const {
    return std::fmax(S - K, 0.0);
  }
};

struct PutPayoff {
  __device__ __host__ double operator()(double S, double K) const {
    return std::fmax(K - S, 0.0);
  }
};
```

## European Options

### European Options Class and CPU Implementation

We used templated class with functor structs to make the European Options Class. We have some variables (S0, K, sigma...) and two methods : CPU and CPU wrapper (for GPU Kernel)

```cpp
template <typename Payoff> class EuropeanOption {
private:
  double S0, K, r, sigma, T;

public:
  EuropeanOption(double _S0, double _K, double _r, double _sigma, double _T)
      : S0(_S0), K(_K), r(_r), sigma(_sigma), T(_T) {}

  double europeanOptionCPU(int paths);
  double europeanOptionGPU(int paths);
};
```

The CPU Implementation uses Black Scholes Equation. The number of paths are taken as input and then calculated. We used fixed seed random variable for reproducable results.

```cpp
template <typename Payoff>
double EuropeanOption<Payoff>::europeanOptionCPU(int paths) {

  std::mt19937_64 rng(42);
  std::normal_distribution<double> norm(0.0, 1.0);

  double payoff_sum = 0.0;
  Payoff payoff;

  for (int i = 0; i < paths; ++i) {
    double Z = norm(rng);
    double ST =
        S0 * std::exp((r - 0.5 * sigma * sigma) * T + sigma * std::sqrt(T) * Z);
    double curr_payoff = payoff(ST, K);
    payoff_sum += curr_payoff;
  }

  return std::exp(-r * T) * (payoff_sum / paths);
}
```
### European Options GPU Kernel

We have used one GPU Kernel initialized per path (we could have used batches as well but nvm). Standard Black Scholes are used per path.
Following is the implementation for the same :

```cpp
template <typename Payoff>
__global__ void europeanOptionGPUKernel(double S0, double K, double r,
                                        double sigma, double T, int paths,
                                        double *results, Payoff payoff) {
  int idx = blockIdx.x * blockDim.x + threadIdx.x;
  if (idx >= paths)
    return;

  curandState state;
  curand_init(42ULL, idx, 0, &state);

  double Z = curand_normal_double(&state);
  double ST = S0 * exp((r - 0.5 * sigma * sigma) * T + sigma * sqrt(T) * Z);
  results[idx] = payoff(ST, K);
}
```

```cpp
// Kernel Wrapper
template <typename Payoff>
double EuropeanOption<Payoff>::europeanOptionGPU(int paths) {
  double *d_results = nullptr;
  cudaMalloc(&d_results, paths * sizeof(double));

  int blockSize = 256;
  int gridSize = (paths + blockSize - 1) / blockSize;
  Payoff payoff;

  europeanOptionGPUKernel<<<gridSize, blockSize>>>(S0, K, r, sigma, T, paths,
                                                   d_results, payoff);
  cudaDeviceSynchronize();

  std::vector<double> h_results(paths);
  cudaMemcpy(h_results.data(), d_results, paths * sizeof(double),
             cudaMemcpyDeviceToHost);

  cudaFree(d_results);

  double sum = 0.0;
  for (double payoff_results : h_results) {
    sum += payoff_results;
  }

  return exp(-r * T) * (sum / static_cast<double>(paths));
}
```

## Asian Options

For Asian Options, we have used Arithmetic Mean over last <em>X fixings</em> to find the Payoff. The implementation was similar to European just had to make one extra loop inside for all <em>dt = T/tradingDays</em> steps.

The implementation of CPU function, wrapper and kernel is given in the Github Repo.

## Basket Options

This was cool! So I came to know about these options from a PS of a Hackathon. So these type of options have a portfolio where multiple assets are considered as underlying with weights assigned to each asset. The problem is that these can be correlated with other and so we use Choleskey Method. I do not really understand how this works and used it as a black box for this project. It uses Lower Triangular Matrix(L) of the Correlation Matrix(R) to change the Normal Random Variable vector (Y) to Correlated Normal Random Variable vector (Z).

The implementation of CPU function, wrapper and kernel is given in the Github Repo.

## American Options

The initial method I thought of using was traversing back the Binomial Tree which we build in the Binomial Method. However, I later found out about Least-Mean Square algorithm and read it from the original research paper. Also found a couple of implementations on youtube which made it easier. The main manipulation was evaluating regression equation using the following function :

```cpp
__host__ __device__ void quadraticRegression(double *X, double *Y, int n,
                                             double &a0, double &a1,
                                             double &a2) {
  double Sx = 0, Sx2 = 0, Sx3 = 0, Sx4 = 0;
  double Sy = 0, Sxy = 0, Sx2y = 0;

  for (int i = 0; i < n; ++i) {
    double x = X[i];
    double x2 = x * x;
    double y = Y[i];

    Sx += x;
    Sx2 += x2;
    Sx3 += x2 * x;
    Sx4 += x2 * x2;
    Sy += y;
    Sxy += x * y;
    Sx2y += x2 * y;
  }

  double D = n * (Sx2 * Sx4 - Sx3 * Sx3) - Sx * (Sx * Sx4 - Sx2 * Sx3) +
             Sx2 * (Sx * Sx3 - Sx2 * Sx2);

  if (D < 0) {
    D *= -1;
  }

  if (D < 1e-12) {
    a0 = 0;
    a1 = 0;
    a2 = 0;
    return;
  }

  double D0 = Sy * (Sx2 * Sx4 - Sx3 * Sx3) - Sx * (Sxy * Sx4 - Sx3 * Sx2y) +
              Sx2 * (Sxy * Sx3 - Sx2 * Sx2y);

  double D1 = n * (Sxy * Sx4 - Sx3 * Sx2y) - Sy * (Sx * Sx4 - Sx2 * Sx3) +
              Sx2 * (Sx * Sx2y - Sxy * Sx2);

  double D2 = n * (Sx2 * Sx2y - Sxy * Sx3) - Sx * (Sx * Sx2y - Sxy * Sx2) +
              Sy * (Sx * Sx3 - Sx2 * Sx2);

  a0 = D0 / D;
  a1 = D1 / D;
  a2 = D2 / D;
}
```

The implementation of CPU function is given in the Github Repo. The GPU method cannot work here as regression is inherently sequential.

## Benchmarking RAII Class

RAII - Resource Allocation is Initialization (done using Constructors and Destructors of Timer Class)

```cpp
class Timer {
private:
    std::string label;
    std::chrono::high_resolution_clock::time_point start;
    
public:
    Timer(const std::string& label = "") : label(label), start(std::chrono::high_resolution_clock::now()) {}

    double getDuration() const {
      auto end = std::chrono::high_resolution_clock::now();
      return std::chrono::duration<double>(end - start).count();
    }

    ~Timer() {
        auto end = std::chrono::high_resolution_clock::now();
        double duration = std::chrono::duration<double>(end - start).count();
        std::cout << label << " took " << duration << " seconds." << std::endl;
    }
};

// Sample Implementation
{
    Timer timer("CPU Function") // Constructor Called
    // Call to CPU Function
} // Destructor Called
```

## CLI Integration

Used cxxopts for CLI integration. This header was cloned from its [Github Repo](https://github.com/jarro2783/cxxopts).

## Benchmark Results

The benchmark results are also included in the Github Repo in Results.ipynb.
Following were the results :
- European Options a speed up of *2.815X* on *10^7* paths.
- Asian Options a speed up of *190.35X* on *10^6* paths.
- Basket Options a speed up of *88.28X* on *10^7* paths.

## Conclusion and Learnings

This project was a great one for learning Multithreading using CUDA and applying it to Finance. I learned not only about CUDA but also about some cool option types, things like RAII and Functor Structs. Overall a very positive learning outcome :)