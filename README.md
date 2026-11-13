# Synthetic Market Generator

This repository generates a synthetic option market for experiments with stochastic-volatility calibration. The C++ program simulates an underlying asset and latent variance with a Heston process, evolves short rates with a Hull--White process, varies the physical-measure parameters through an Ornstein--Uhlenbeck-style jump process, and prices European call options with QuantLib's analytic Heston engine.

The output contains both the observable synthetic market and the latent parameters that generated it, making the dataset suitable for parameter-recovery and measure-coupled calibration studies.

## Model and generated data

For every simulated business day, the generator:

1. evolves the physical-measure parameter vector
   $$(\mu,\kappa_{\mathbb P},\theta_{\mathbb P},\xi,\rho,\lambda);$$
2. maps it to the risk-neutral mean-reversion parameters
   $$\kappa_{\mathbb Q}=\kappa_{\mathbb P}+\lambda,\qquad
   \theta_{\mathbb Q}=\frac{\kappa_{\mathbb P}\theta_{\mathbb P}}
   {\kappa_{\mathbb P}+\lambda};$$
3. evolves the spot price and latent variance under the Heston process;
4. evolves the short rate with a Hull--White process;
5. maintains rolling weekly and monthly option expiries;
6. prices calls across nine moneyness levels from `0.80` through `1.20`; and
7. recovers Black implied volatility from each generated call price.

The default experiment begins on `2010-01-01` (adjusted to the next TARGET business day) and runs for 12,000 business-day steps. Randomness is seeded from `std::random_device`, so separate runs are not deterministic.

## Requirements

- A C++17 compiler, such as `g++`
- [QuantLib](https://www.quantlib.org/) headers and library
- Eigen headers if you retain or extend the include configuration in `generator.sh`
- A Unix-like shell

`generator.sh` searches common `/usr/local` and Homebrew installation paths. If QuantLib or Eigen is installed elsewhere, edit the `-I` and `-L` flags in the script before running it.

## Build and run

From the repository root:

```bash
mkdir -p Data_Output
bash generator.sh
```

The script compiles [`app/main.cpp`](app/main.cpp) as C++17, links against QuantLib, runs the resulting `a.out`, and removes that temporary executable when the run finishes. The `Data_Output` directory must exist before execution because the program does not create it.

The default 12,000-step configuration prices many option surfaces and may take substantial time. For a smaller trial, reduce `time_steps` in the `generateData` constructor in [`src/generateData.cpp`](src/generateData.cpp).

## Default configuration

| Setting | Default |
|---|---:|
| Business-day steps | 12,000 |
| Initial spot, $S_0$ | 100.0 |
| Initial variance, $v_0$ | 0.04 |
| Physical drift, $\mu$ | 0.05 |
| Physical mean reversion, $\kappa_{\mathbb P}$ | 1.5 |
| Physical long-run variance, $\theta_{\mathbb P}$ | 0.04 |
| Volatility of variance, $\xi$ | 0.3 |
| Spot/variance correlation, $\rho$ | -0.7 |
| Initial short rate, $r$ | 0.06 |
| Dividend yield, $q$ | 0.00 |
| Variance-risk parameter, $\lambda$ | 0.50 |
| Parameter jump intensity | 52 per year |
| Weekly expiries maintained | 5 |
| Monthly expiries maintained | 12 |

The hard-coded experiment settings are grouped near the start of the `generateData` constructor in [`src/generateData.cpp`](src/generateData.cpp).

## Outputs

The run writes two CSV files beneath `Data_Output/`:

| File | Contents |
|---|---|
| `Data_Output/C_data.csv` | Observation date, expiry, strike, maturity, rate, underlying price, call price, and implied volatility for every generated contract. |
| `Data_Output/heston_params.csv` | Observation date, $\mu$, $\kappa_{\mathbb P}$, $\theta_{\mathbb P}$, $\xi$, $\rho$, $\lambda$, and the latent variance. |

Rates in `C_data.csv` are written in percentage points (`rates[t] * 100`); the other model parameters retain the units used internally by the generator.

## Repository layout

```text
.
├── app/
│   └── main.cpp              # Program entry point
├── src/
│   ├── generateData.cpp      # Simulation, option grid, and CSV output
│   └── callPrice.cpp         # QuantLib analytic Heston call pricer
├── Helper_Files/
│   └── getData.ipynb         # Notebook for inspecting generated data
└── generator.sh              # Compile-and-run helper
```

[`app/main.cpp`](app/main.cpp) constructs `generateData`; the constructor performs the complete simulation and writes the results. [`src/callPrice.cpp`](src/callPrice.cpp) validates inputs and prices European calls with `QuantLib::AnalyticHestonEngine`.

