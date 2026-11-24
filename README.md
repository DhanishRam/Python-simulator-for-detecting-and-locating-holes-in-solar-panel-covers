[README(2) (1).md](https://github.com/user-attachments/files/35320722/README.2.1.md)
# Self-Verifying Solar Panel Cover Simulator

Can a row of solar modules tell whether its protective cover has a hole? This Python simulator turns four modules into optical instruments. At night, one simulated module emits light and its neighbours measure the reflection from the liner. By day, twelve photovoltaic substrings track sunlight entering the cover. The direction and motion of that light help distinguish a hole in the roof from a slot in a side skirt.

![Two heatmaps of simulated substring current. The cover hole moves across three substrings as the illumination angle changes. The skirt slot is brightest early in the sequence.](docs/opening-comparison.png)

## Run the first experiment

From the extracted repository directory, use Python 3.10 or newer. NumPy is the only runtime dependency.

```bash
python -m venv .venv
source .venv/bin/activate
python -m pip install .
pvcover run --preset cover-hole --seed 42 --out runs/first
```

On Windows PowerShell, activate with `.\.venv\Scripts\Activate.ps1`. The module entry point `python -m pvcover` accepts the same commands as `pvcover`. Each command writes to a new output directory.

Open `runs/first/summary.json`. The example injects a **95 cm² hole at x = 1.737 m**. With seed 42 the fit returns **93.6 cm² at x = 1.72 m**. Fourteen of sixteen daylight scans pass the acquisition gates. Wind excludes one scan and reference instability excludes the other. The cover-plane fit leaves about 0.6% of the squared excess unexplained. [Expected output](examples/expected-summary.json)

## Experiments

| Question | Command |
| --- | --- |
| Which conditions can I inject? | `pvcover list` |
| What changes for a slot at the same position? | `pvcover run --preset lower-slot --out runs/slot` |
| What does a wet liner look like at night? | `pvcover run --preset wet-liner --out runs/wet` |
| Can the estimator find a slot when another substring is soiled? | `pvcover run --scenario examples/custom-slot.json --out runs/mixed` |
| How often does each estimator call a hole? | `pvcover benchmark --trials 10 --seed 42 --out runs/benchmark` |
| How does detection vary with hole size, sky and measured gap? | `python examples/sensitivity.py --trials 6 --seed 730 --out runs/sweep` |

To replay an exported run, pass its scenario, sky and seed back into the simulator.

```bash
pvcover run --scenario runs/first/scenario.json --sky runs/first/sky.csv --seed 42 --out runs/replay
```

The Python API exposes the raw measurements and intermediate results.

```python
from pvcover import Opening, Scenario, load_sky, run

case = Scenario(name="small cover hole", gap_m=0.28,
                opening=Opening(family="cover", x_m=2.13, area_m2=0.006))
result = run(case, load_sky(), seed=7)
print(result.verdict.label)
print(result.summary()["day"]["best_fit"])
print(result.day.current_a.shape)  # accepted scans × 12 substrings
```

## Reading a run

| File | What to inspect |
| --- | --- |
| `summary.json` | Verdict and reason, quality gates, accepted scans, fitted family, position, area, runner-up and residual |
| `day.csv` | One row per accepted scan and substring with reference and receiver currents, gain, baseline, section factor and signed excess |
| `night.csv` | Twelve directed module pairs with complex lock-in data, retracted pickup, optical phase and temperature-corrected signed coupling |
| `timeline.csv` | Logical switching states, acquisition windows, restoration to the string state and the night deployment deadline |
| `scan_log.csv` | Every attempted scan and the reason for rejection |
| `scenario.json`, `sky.csv` | Complete synthetic inputs for replay |

A verdict of `B` reports a fitted cover hole. `F` covers several edge conditions and common leakage shifts, so read its **label** as well as its code. `HOLD` means the measurements could not support a complete diagnosis. A missing fit is stored as JSON `null`. [Verdict rules and equations](docs/model.md)

## What the examples establish

The standard benchmark varies measurement noise over 15 fixed scenarios. With seeds 42–51 it returned the expected verdict code in 150 of 150 runs. On the 100 trials eligible for the daylight comparison, a raw threshold called 50 of the 90 non-hole trials holes. A threshold on local excess still called all 30 wall apertures holes. Both the four-receiver fit and twelve-substring fit classified the cover-hole trials correctly and avoided false hole declarations on this small benchmark. These fixtures alone cannot establish a resolution advantage. [Study design and reproducible results](docs/experiments.md)

The size sweep exposes a limit the default demo hides. With a 12 mm radius and six positions at each of three gaps, the model found 15 of 18 holes in the clear-sky case and 0 of 18 in haze. Under fully diffuse illumination the angular-evidence gate held the verdict. These are outcomes of the synthetic model. Field sensitivity requires independently measured calibration and validation data.

## Project layout

| Path | Purpose |
| --- | --- |
| [src/pvcover/simulation.py](src/pvcover/simulation.py) | Simulation entry point and verdict precedence |
| [src/pvcover/optics.py](src/pvcover/optics.py), [inverse.py](src/pvcover/inverse.py) | Finite-aperture ray transport and the independent coarse kernel bank |
| [src/pvcover/day.py](src/pvcover/day.py), [night.py](src/pvcover/night.py) | Electrical observations, calibration and signal processing |
| [src/pvcover/acquisition.py](src/pvcover/acquisition.py) | Per-module switching timeline |
| [src/pvcover/config.py](src/pvcover/config.py), [data/](src/pvcover/data/) | Validated input objects, synthetic sky and 15 fault presets |
| [src/pvcover/study.py](src/pvcover/study.py), [examples/](examples/) | Repeatable studies, custom inputs, reference output and figure generator |
| [tests/](tests/), [.github/workflows/ci.yml](.github/workflows/ci.yml) | Physics invariants, decision logic, command-line tests and distribution checks |

## Development and distribution

```bash
python -m pip install -e ".[dev,plot]"
python -m unittest discover -s tests -v
python -m coverage run -m unittest discover -s tests
python -m coverage report
python examples/compare_openings.py --out runs/comparison.png
python -m build
```

The build creates a wheel and source distribution in `dist/`. The wheel includes the synthetic sky and presets. GitHub Actions is configured to test on Linux, macOS and Windows, check distribution metadata and smoke-test the installed wheel outside the source checkout. [Contribution guidelines](CONTRIBUTING.md)

## Design translation and release notes

**0.1.0** implements the two optical verification paths, reference and gain correction, geometry-dependent aperture matching, signed demodulation, fault classes and measurement gating. The repository uses original synthetic dimensions, calibration curves, sky data, fault examples and artwork. The accelerated sun-angle sequence is a controlled experiment.

The daylight model traces first-pass rays through one aperture. The night model uses calibrated coupling curves and a sampled fundamental. The extent estimator fits several adjacent pairs jointly so that uniform liner loss and partial deployment can be compared. The public model adds an ambiguous-family verdict, independent noise streams, off-grid trials and replayable CSV records. Mechanical loads, fabric durability, string power electronics and field commissioning are outside its scope. [Model assumptions](docs/model.md)

The code, examples and original figure are distributed under the [MIT License](LICENSE).
