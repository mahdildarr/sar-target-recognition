# SAR Target Recognition with CNNs — and a test for shortcut learning

Automatic target recognition (ATR) on **MSTAR** synthetic aperture radar imagery with convolutional neural networks. Beyond accuracy, the project asks *what the network is actually looking at*: a masking-based **shortcut test** measures how much of the accuracy comes from the vehicle and how much from the background clutter that MSTAR is known to share within each class.

<p align="center"><img src="docs/assets/pipeline.svg" width="100%" alt="Pipeline"></p>

> **Status:** 🚧 Work in progress — pipeline implemented and tested; MSTAR experiments running.
> This repository presents the project and its design. **The source code is kept in a private repository** and can be shared on request.

## Why this project

SAR sees through clouds and at night, which makes it central to surveillance and defence applications. Its images, however, look nothing like photographs — a target is a handful of bright scattering centres, a radar shadow and speckled clutter — and its appearance changes strongly with the viewing angle. CNNs report near-perfect accuracy on MSTAR, the standard benchmark, but each vehicle was recorded at a particular site, so the ground around the target is correlated with the class. **A high score is only meaningful if the model looks at the target.**

## What is inside

| | |
|---|---|
| **Networks** | A-ConvNet (all-convolutional, ~0.3 M parameters) · ResNet-18 from scratch · ResNet-18 ImageNet-pretrained |
| **Protocols** | SOC: 10 classes, 17° → 15° · EOC-1: 4 classes, 17° → 30° / 45° depression |
| **Shortcut test** | full vs target-only vs background-only chips, and a model trained without ever seeing the target |
| **Robustness** | noise (SNR 20 → −5 dB), corrupted pixels, partial target occlusion, per-azimuth accuracy |
| **Explainability** | Grad-CAM with *measured* localisation: heat-map energy on the target, pointing-game hit rate |
| **Rigour** | validation from the training angle only, three seeds (mean ± std), unit tests, synthetic end-to-end test |

## Design

The full experiment design — data protocol, preprocessing, network architectures, shortcut-test logic, robustness and explainability metrics, limitations — is in **[docs/DESIGN.md](docs/DESIGN.md)**.

## Results

Result tables will be published here when the experiments are completed.

| Model | Params | SOC acc. (15°) | EOC-1 acc. (30°) | Background-only acc. |
|---|---|---|---|---|
| A-ConvNet | 0.30 M | _TBD_ | _TBD_ | _TBD_ |
| ResNet-18 (scratch) | 11.2 M | _TBD_ | _TBD_ | _TBD_ |
| ResNet-18 (ImageNet) | 11.2 M | _TBD_ | _TBD_ | – |

## Tech stack

Python · PyTorch · torchvision · NumPy / SciPy · pandas · Matplotlib · Google Colab (T4 GPU)

## Author

**Şule Özdamar** — Computer Engineering, İstanbul Arel University · [github.com/mahdildarr](https://github.com/mahdildarr)

MSTAR data: Sensor Data Management System (SDMS), U.S. Air Force Research Laboratory. The dataset is not redistributed here.
