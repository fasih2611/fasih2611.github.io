---
title: "Projects"
permalink: /projects/
---

<!-- Add or remove entries freely — this is just a markdown list. -->

### [DDPM_torch](https://github.com/fasih2611/DDPM_torch)

Denoising diffusion probabilistic models implemented in PyTorch, written from
the paper rather than adapted from a reference repository — the noise schedule,
the UNet, the training loop, and the sampler. It includes a forward-process
check that corrupts real images step by step to confirm the schedule reaches
pure noise, plus training and evaluation entry points.

Three simplifications are deliberate and documented: bits-per-dimension is
computed without a binned Gaussian, FID is measured against a simple classifier
rather than the standard Inception network, and the UNet omits attention
blocks. The goal was a complete, readable diffusion pipeline, not a
leaderboard number.

### [SGD-and-Batch](https://github.com/fasih2611/SGD-and-Batch)

A neural network built from nothing but NumPy — layers, ReLU and its
derivative, and a hand-written backward pass that walks the layers in reverse
and propagates the error term itself. No autograd, no framework.

It exists to make one comparison concrete: the same 13→2→1 network trained on
the Boston housing data twice, once with stochastic gradient descent and once
with full-batch updates, so the difference in convergence behaviour comes from
the optimiser alone. Writing the chain rule out by hand is the point of the
exercise.

### [DLT3d](https://github.com/fasih2611/DLT3d)

A Direct Linear Transform implementation that recovers the mapping between 3D
world coordinates and 2D image coordinates, and inverts it to go back the other
way. That calibration is the foundation for the rest of the project.

It runs as a live control loop: OpenCV tracks coloured markers by HSV
thresholding with background subtraction, the calibrated homography converts
their pixel positions into real-world coordinates, and those coordinates are
streamed over TCP to an ESP32 running a PID controller. There is a separate
click-to-calibrate tool for collecting correspondence points and a tuner for
the PID gains — a full path from camera geometry to a physical actuator.
