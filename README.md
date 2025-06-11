# Neural-Network-on-FPGA

## Overview
This repository contains a fully hardware **multi-layer perceptron (MLP)** implemented in Verilog with **no vendor IP cores**, making it portable across FPGA platforms. The design was built and tested on a **Xilinx Arty A7** (Artix-7) development board. By running inference directly on the FPGA fabric, the network achieves low-latency, on-device processing—ideal for edge AI/IoT scenarios where bandwidth, power, and data privacy are critical.

## Architecture
The network is a feed-forward  MLP (multilayer perceptron) with **4 fully-connected layers**:
- **Layer 1**: 30 neurons  
- **Layer 2**: 30 neurons  
- **Layer 3**: 10 neurons  
- **Layer 4**: 10 neurons  

Data flows through each layer in sequence. The top-level module (`zynet.v`) orchestrates this flow using shift-register pipelines: it accepts external inputs, feeds them through each layer, and finally passes the 10 output scores to a `maxFinder.v` block, which selects the index of the highest activation as the predicted class.

## Verilog Modules
- **`neuron.v`**  
  Implements a single neuron: multiplies inputs by weights, adds bias, then applies a selectable activation (ReLU or Sigmoid).

- **`relu.v`**  
  Simple ReLU activation:  
  ```verilog
  assign dout = (din[MSB] == 1'b0) ? din : {WIDTH{1'b0}};

* **`Sig_ROM.v`**  
  Sigmoid activation via lookup table (ROM). Maps a truncated fixed-point input to a precomputed sigmoid output.

* **`Layer_1.v`, `Layer_2.v`, `Layer_3.v`, `Layer_4.v`**  
  Each layer module instantiates N copies of `neuron.v` (30 or 10) and connects them to `Weight_Memory.v`. Layer outputs are registered and forwarded to the next layer.

* **`Weight_Memory.v`**  
  Read-only memory storing all pretrained weights and biases. Layers fetch their parameter sets from this centralized block.

* **.mif Files**  
  Memory Initialization Files for weights and biases—loadable into FPGA block RAM to preconfigure all layer parameters before synthesis or at runtime.

* **`maxFinder.v`**  
  Scans the 10 outputs from the last layer and outputs the index with the maximum value (the predicted class label).

* **`axi_lite_wrapper.v`**  
  Wraps the network as an AXI4-Lite slave IP. Provides registers for writing input data and reading output results, enabling easy integration into a larger SoC.

* **`zynet.v`**  
  Top-level module. Instantiates all layers, the weight memory, the max finder, and the AXI wrapper. Manages control signals and registers to pipeline data through the network.

* **`include.v`**  
  Shared parameters and macros (e.g. data widths, neuron counts, fixed-point scaling).


## Hardware Platform

* **Target board**: Digilent Arty A7 (Artix-7 FPGA)  
* **FPGA family**: Xilinx Artix-7 (e.g. XC7A35T or XC7A100T)  
* **Toolchain**: Xilinx Vivado WebPACK (no paid licenses required)  

Since all modules use generic RTL (no vendor-specific DSPs or block RAM primitives), the design can be synthesized on any FPGA family with minimal changes. Simply adjust clock and I/O constraints, reassign pins, and you’re ready to run inference on your chosen board!

