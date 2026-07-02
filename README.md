# C<sub>model</sub> Fitting for Biomass Growth in a Reciprocating-Shaking Photobioreactor

This repository provides a Python workflow for fitting an effective mass-transfer parameter, `C_model`, from biomass growth data in a reciprocating-shaking photobioreactor. The script couples a Monod-type biomass growth model with an effective substrate transfer formulation and estimates the reactor-specific `C_model` by minimizing the weighted sum of squared biomass residuals.

The workflow reads experimental biomass data, solves the dynamic growth model, fits `C_model`, and exports fitted curves, residuals, parameter tables, and publication-ready figures.

## Features

- Reads biomass time-series data from CSV files.
- Supports optional biomass standard deviation for weighted fitting.
- Fits the effective transfer parameter `C_model` using bounded scalar optimization.
- Solves the biomass-substrate dynamic model with `scipy.integrate.solve_ivp`.
- Exports fitted curves, residuals, objective-function profiles, and fitted parameters.
- Generates high-resolution figures for model fitting, residual analysis, and `C_model` sensitivity/profile analysis.
- Can be controlled either by command-line arguments or by a JSON configuration file.

## Model Summary

The model tracks biomass concentration `X` and bulk substrate concentration `S_bulk`. The cellular substrate concentration `S_cell` is obtained from a local mass-transfer balance. The specific growth rate follows Monod kinetics:

```text
mu = mu_max * S_cell / (K_s + S_cell)
```

Biomass and substrate are then advanced by:

```text
dX/dt = mu * X
dS_bulk/dt = -(1000 / Y_XS) * dX/dt
```

The fitted parameter `C_model` is converted to the effective liquid-side transfer coefficient through the manuscript scaling:

```text
k_L = C_model * sqrt(D_AB) / nu^0.25 * sqrt(amplitude) * frequency^0.75
```

`C_model` should be interpreted as an effective reactor-specific fitted parameter, not as a universal physical constant.

## Repository Structure

A typical project structure is:

```text
.
├── fit_cmodel.py          # Main fitting script
├── config.json            # Optional JSON configuration file
├── data.csv               # Experimental biomass data
├── output/                # Generated results
└── README.md
```

The script name can be changed, but the examples below assume the main file is named `fit_cmodel.py`.

## Requirements

Python 3.9 or later is recommended.

Required Python packages:

```text
numpy
pandas
scipy
matplotlib
```

Install dependencies with:

```bash
pip install numpy pandas scipy matplotlib
```

## Input Data Format

The input biomass data must be provided as a CSV file.

Required columns:

| Column | Description | Unit |
|---|---|---|
| `time_day` | Cultivation time | day |
| `biomass_mg_dw_L` | Observed biomass concentration | mg DW L^-1 |

Optional column:

| Column | Description | Unit |
|---|---|---|
| `biomass_sd_mg_dw_L` | Standard deviation of biomass observations | mg DW L^-1 |

Example:

```csv
time_day,biomass_mg_dw_L,biomass_sd_mg_dw_L
0,40,2.5
1,55,3.1
2,74,4.0
3,96,4.8
4,121,5.2
```

If `biomass_sd_mg_dw_L` is provided and all values are valid and positive, weighted fitting is used. Otherwise, all observations are assigned equal weights.

## Usage

### 1. Run with command-line arguments

```bash
python fit_cmodel.py \
  --data data.csv \
  --case-name example_case \
  --output-dir output \
  --X0-g-dw-L 0.04 \
  --S0-mg-L 7.29 \
  --mu-max-day 0.2465 \
  --K-s-mg-L 5.16 \
  --Y-XS-g-dw-per-g-substrate 84.99 \
  --frequency-hz 2.0 \
  --amplitude-cm 2.5
```

### 2. Run with a JSON configuration file

Create a `config.json` file:

```json
{
  "case_name": "example_case",
  "data_file": "data.csv",
  "output_dir": "output",
  "biomass_unit": "mg_dw_L",
  "initial_from_first_observation": false,
  "X0_g_dw_L": 0.04,
  "S0_mg_L": 7.29,
  "mu_max_day": 0.2465,
  "K_s_mg_L": 5.16,
  "Y_XS_g_dw_per_g_substrate": 84.99,
  "frequency_hz": 2.0,
  "amplitude_cm": 2.5,
  "D_AB_m2_day": 0.0000864,
  "nu_m2_s": 0.000001,
  "alpha_m2_g_dw": 1.0,
  "fit_log10_Cmodel_bounds": [-8.0, 2.0],
  "profile_log10_half_width": 2.0,
  "profile_points": 241,
  "smooth_points": 500
}
```

Then run:

```bash
python fit_cmodel.py --config config.json
```

You can also override the data file or output directory while using a configuration file:

```bash
python fit_cmodel.py --config config.json --data new_data.csv --output-dir new_output
```

## Main Parameters

| Parameter | Description |
|---|---|
| `case_name` | Name of the fitting case |
| `data_file` | Path to the input biomass CSV file |
| `output_dir` | Directory for generated results |
| `initial_from_first_observation` | If `true`, the first observed biomass value is used as `X0` |
| `X0_g_dw_L` | Initial biomass concentration in g DW L^-1 |
| `S0_mg_L` | Initial bulk substrate concentration in mg L^-1 |
| `mu_max_day` | Maximum specific growth rate in day^-1 |
| `K_s_mg_L` | Monod half-saturation constant in mg L^-1 |
| `Y_XS_g_dw_per_g_substrate` | Biomass/substrate yield coefficient |
| `frequency_hz` | Reciprocating frequency in Hz |
| `amplitude_cm` | Reciprocating amplitude in cm |
| `D_AB_m2_day` | Diffusion coefficient in m^2 day^-1 |
| `nu_m2_s` | Kinematic viscosity in m^2 s^-1 |
| `alpha_m2_g_dw` | Biomass-specific transfer area coefficient |
| `fit_log10_Cmodel_bounds` | Lower and upper bounds for fitting `log10(C_model)` |
| `profile_log10_half_width` | Half-width of the objective-function profile around the fitted value |
| `profile_points` | Number of points used for the `C_model` profile |
| `smooth_points` | Number of time points used for the fitted smooth curve |

## Outputs

After a successful run, the output directory contains:

| File | Description |
|---|---|
| `fitted_curve.csv` | Smooth model-predicted time series |
| `residuals.csv` | Observed biomass, predicted biomass, residuals, and weights |
| `Cmodel_profile.csv` | Objective-function profile over `C_model` |
| `fitted_parameters.csv` | Fitted parameters, model settings, and goodness-of-fit metrics |
| `fitted_curve.png` | Biomass fitting figure |
| `residuals.png` | Residual plot |
| `Cmodel_profile.png` | Relative weighted SSE profile for `C_model` |

The terminal also reports the fitted `C_model`, effective `K_mass`, `k_L`, coefficient of determination `R2`, RMSE, and output directory.

## Example Terminal Output

```text
C_model fitting complete.
Case: example_case
C_model = 1.234567e-04
K_mass = 1.234567e-02 day^-1
k_L = 1.234567e-03 m day^-1
R2 = 0.9876
RMSE = 5.4321 mg DW L^-1
Outputs written to: output
```

## Notes and Limitations

- Currently, the input biomass unit is restricted to `mg_dw_L`.
- The time points in the input CSV must be nonnegative, unique, and sorted automatically by the script.
- Biomass values must be nonnegative.
- The fitted `C_model` depends on the reactor geometry, operating conditions, and parameter settings.
- The default optimization bounds are broad. For a specific experimental system, physically informed bounds are recommended.
- The generated figures use Arial as the default font. If Arial is unavailable on your system, Matplotlib may substitute another font.

## Troubleshooting

### Missing required columns

Make sure the input CSV contains:

```text
time_day, biomass_mg_dw_L
```

### Optimization fails

Try adjusting:

```text
fit_log10_Cmodel_bounds
X0_g_dw_L
S0_mg_L
mu_max_day
K_s_mg_L
Y_XS_g_dw_per_g_substrate
```

### Very poor fitting performance

Check whether:

- The biomass unit is correct.
- The initial biomass and substrate concentration are realistic.
- The yield coefficient is consistent with the experimental substrate data.
- The fitted parameter bounds are too narrow.
- The first observation should be used as the initial biomass by enabling `initial_from_first_observation`.

## License

Specify the license according to your project requirements, for example MIT, Apache-2.0, or a journal supplementary-material license.

## Citation

If this code is used in a publication, cite the associated manuscript or repository according to the final published information.
