# Design

This document describes how the experiments are designed: the data protocol, the networks, and the checks used to make sure a high accuracy means what it appears to mean.

## 1. Problem

Synthetic Aperture Radar (SAR) images the ground day and night and through clouds, but its images look nothing like photographs: targets appear as a few bright scattering centres, a dark radar shadow and speckled ground clutter. The appearance of a vehicle changes strongly with the viewing geometry (azimuth and depression angle).

MSTAR is the standard public benchmark for SAR automatic target recognition (ATR). CNNs reach very high accuracy on it, but MSTAR also has a known weakness: each vehicle was recorded at a particular site, so the **ground clutter around the target is correlated with the class**. A model can therefore score well without recognising the vehicle. This project measures how much a model depends on the target versus the background.

## 2. Data and protocols

| Protocol | Classes | Train | Test | Question |
|---|---|---|---|---|
| **SOC** (standard operating condition) | 10: 2S1, BMP2, BRDM2, BTR60, BTR70, D7, T62, T72, ZIL131, ZSU234 | 17° depression | 15° depression | Baseline recognition accuracy |
| **EOC-1** (depression change) | 4: 2S1, BRDM2, T72, ZSU234 | 17° | 30° (45°) | Generalisation to a large change in viewing geometry |

* Vehicle variants follow the standard definitions: SOC uses one serial each for BMP2 (9563), BTR70 (c71) and T72 (132); EOC-1 uses T72 A64. All-variant training is one configuration switch away.
* A validation split (10 % per class) is taken **from the training angle only**; the test angle is never used for model selection or early stopping.
* Metadata (class, serial number, azimuth, depression) is read from the raw MSTAR headers, so azimuth-dependent errors can be analysed.

## 3. Preprocessing

1. Magnitude → log domain (compresses the very large dynamic range of radar returns).
2. Centre region of 100 × 100 px; **random 88 × 88 crops** for training, centre crop for testing: translation augmentation that keeps the target in view.
3. Per-chip standardisation (zero mean, unit variance), so global brightness differences between recordings cannot be used.
4. No horizontal flipping by default: a mirrored SAR chip is not a physically valid view of the same vehicle.

## 4. Networks

**A-ConvNet** (all-convolutional, after Chen et al., 2016): designed for small SAR datasets:

| Layer | Kernel | Channels | Output |
|---|---|---|---|
| Input | – | 1 | 88 × 88 |
| Conv + ReLU + MaxPool | 5 × 5 | 16 | 42 × 42 |
| Conv + ReLU + MaxPool | 5 × 5 | 32 | 19 × 19 |
| Conv + ReLU + MaxPool | 6 × 6 | 64 | 7 × 7 |
| Conv + ReLU + Dropout 0.5 | 5 × 5 | 128 | 3 × 3 |
| Conv (classifier) | 3 × 3 | 10 | 1 × 1 |

No fully connected layers: ~0.3 M parameters, against ~11 M for ResNet-18. A variant with batch normalisation is also trained.

**ResNet-18, two ways**: (a) from scratch with a 3 × 3 stride-1 stem and no max-pooling, so an 88 px chip still yields an 11 × 11 feature map; (b) ImageNet-pretrained with the RGB stem filters averaged into one channel (transfer-learning baseline).

**Training**: AdamW, one-cycle learning-rate schedule, label smoothing 0.1, mixed precision on GPU. The checkpoint is selected by validation accuracy with ties broken by validation log-loss: neighbouring azimuths make validation accuracy saturate early, and without the tie-break the first epoch that reaches 100 % would be kept. Every configuration is trained with **three seeds** and reported as mean ± standard deviation.

## 5. Shortcut-learning test

Each chip is segmented into **target**, **shadow** and **clutter** with transparent image processing (smoothing, percentile thresholds, morphology, connected components). Removed regions are filled with noise that matches the chip's own clutter statistics: not with zeros, which the network could detect as an artefact.

For the background-only chips the removed region is a **fixed central box, identical for every class** (plus any target pixels reaching beyond it). Removing exactly the target-shaped mask would leave the vehicle's outline visible, and the outline alone carries class information (a D7 bulldozer is much larger than a BRDM2). The share of chips whose target reaches beyond the box is reported with every analysis.

| Test | Train on | Test on | Interpretation |
|---|---|---|---|
| A | full chips | full chips | standard accuracy |
| B | full chips | target + shadow only | does the model still work when the clutter is replaced? |
| C | full chips | clutter only | how much does it rely on the background? |
| **D** | **clutter only** | **clutter only** | **can the class be predicted without ever seeing the vehicle?** Accuracy far above chance (10 %) quantifies the dataset bias |

The test is validated on synthetic chips in the MSTAR file format with a class-dependent clutter texture injected on purpose (`tools/make_synthetic_mstar.py --background-bias`): there, test D must score well above chance, while on unbiased synthetic chips it must stay at chance.

## 6. Robustness and explainability

* **Robustness curves** on the test set: additive noise at SNR 20 → −5 dB (computed in linear magnitude, so raw MSTAR files only), randomly corrupted pixels (1–15 %), and partial occlusion of the target (10–50 %) from a random direction.
* **Per-azimuth accuracy** (polar plot) to show at which aspect angles recognition fails.
* **Grad-CAM, measured rather than only shown**: for every test chip, the share of Grad-CAM energy on the target/shadow mask relative to the mask's area, and a pointing-game hit rate (is the heat-map maximum on the target?).

## 7. Limitations

* MSTAR is a controlled collection (one sensor, few sites, few vehicle serials); results do not transfer directly to operational imagery.
* The target/shadow segmentation is approximate; masks are dilated to remove the target completely, at the cost of also removing some nearby clutter.
* EOC-1 at 45° contains strong appearance changes and few samples; it is reported separately.

## References

1. S. Chen, H. Wang, F. Xu, Y.-Q. Jin, "Target Classification Using the Deep Convolutional Networks for SAR Images," *IEEE TGRS*, 2016.
2. R. Schumacher, K. Rosenbach, "ATR of battlefield targets by SAR: classification results using the public MSTAR dataset compared with a dataset by QinetiQ UK," *NATO RTO-MP-SET-080*, 2004.
3. C. Belloni, N. Aouf, A. Balleri et al., "Explainability of deep SAR ATR through feature analysis," *IEEE Transactions on Aerospace and Electronic Systems*, 57(1), 2021.
4. R. R. Selvaraju et al., "Grad-CAM: Visual Explanations from Deep Networks via Gradient-based Localization," *ICCV*, 2017.
5. O. Kechagias-Stamatis, N. Aouf, "Automatic Target Recognition on Synthetic Aperture Radar Imagery: A Survey," *IEEE AES Magazine*, 2021.
